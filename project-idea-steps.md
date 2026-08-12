# FF Harvester — build & deploy steps

Companion to `project-idea.md` (spec v1.1). The spec says *what* is stored and *why*; this says *in what order to build it*. Where they ever disagree, the spec wins.

**Target:** Python 3.11, AWS for compute, managed Postgres for the database, S3 for raw payloads. Steps 0–2 are deliberately AWS-free.

---

## 0. Phase 0 — observe the feed before writing anything

Do this first; it takes two `curl`s and a weekend, and it answers questions that are expensive to get wrong later. Honour the 5-minute floor between requests.

- Does `ff_calendar_nextweek.json` exist? Decides whether the next-week probe is real.
- **When exactly does the week roll over?** Fetch Saturday evening and again Sunday morning, then diff. This sets the sweep time — the single most important schedule in the system.
- What does `date` carry for All-Day / Tentative / Holiday rows?
- **Do titles stay stable across fetches within one week?** Diff the same event between the Wednesday and Saturday payloads. Revision markers, a source suffix, casing, `Prelim` becoming `Final` — whatever drifts is what `normalize_title` has to absorb, and you can't design it from one payload.
- **Does the same `(country, title)` ever legitimately repeat in one week?** Two speeches by the same official on different days is the usual case. If it happens, the identity tuple needs a discriminator and step 4 changes shape.
- Save every payload. They are your test fixtures, and they are the only record of what the feed looked like this week.

## 1. Pure modules, locally, with tests

Plain Python — no AWS, no database, no mocking. Four pure modules:

- `parse.py` — value parser per spec §5. `num` fully expanded, `unit` a dimension, scale suffix kept separately, `flag` returned *and* persisted. Return a frozen `@dataclass`, not a dict, so a typo in a field name fails at import rather than writing `None` to a column.
- `week.py` — `ff_week_start`, the **one** week-key implementation in the codebase. Sunday-start, computed in `America/New_York`. Not ISO weeks: `date.isocalendar()` starts on Monday and will split each feed week in two.
- `identity.py` — `normalize_title` and `event_key`, the **one** definition of what makes two rows the same event. Feed it the drift cases from Phase 0 and assert they collapse to a single key; feed it the genuinely distinct pair and assert they don't. Normalization is a data question, but the answer gets baked into a unique constraint in step 4, so settle it here.
- `guard.py` — `already_satisfied(rows, slot)`, taking `fetch_log` rows already in hand and returning whether this invocation should exit immediately. Pure logic over a query result, no connection. The whole retry design in step 7 rests on it, and wiring EventBridge is the wrong time to find out it's awkward.

Python 3.11 gives you both halves of the date handling in the stdlib, so this needs no dependencies:

- `datetime.fromisoformat` parses the feed's `2026-07-06T02:00:00-04:00` including the offset. (Pre-3.11 it was far pickier — one of the concrete reasons to stay on 3.11+ here.) Never construct the offset yourself; DST will move it to `-05:00` in November.
- `zoneinfo.ZoneInfo("America/New_York")` for the week calculation — no `pytz`, no `dateutil`. On Lambda add the `tzdata` package, since the runtime image has no system tz database.

Test `week.py` and `identity.py` at least as hard as the parser: for `week.py`, a Sunday event, a Saturday 20:00 ET event, and one date each side of the November DST change; for `identity.py`, every title variant Phase 0 turned up. A wrong parse logs a warning; a wrong week key or identity key silently duplicates rows.

## 2. Pick the database

**Recommended: Neon or Supabase free tier.** No VPC, no NAT gateway, connection string in hand, $0.

Aurora Serverless v2 with the Data API is the alternative if you want everything inside AWS — min capacity 0 ACU so it sleeps between fetches. But it still bills storage at 0 ACU, adds resume latency, and its main selling point (HTTPS access, no VPC) is a problem Neon simply doesn't have. Only worth it if AWS-native is a requirement in itself.

Either way: connection string in **SSM Parameter Store**, not Secrets Manager. Secrets Manager is $0.40/secret/month and is the only line item that would break the $0 target.

Driver: `psycopg` (v3) with the binary extra. It supports the `ON CONFLICT … WHERE` upsert directly and `execute_batch`-style batching for the ~150 rows a week payload holds.

## 3. Create the S3 bucket

One bucket, two prefixes: `raw/` for verbatim payloads (`raw/<week_key>/<fetched_at>.json`), `exports/` for the CSV and JSON dumps. Block public access, versioning on, **no lifecycle expiry on `raw/`** — that archive is the entire point of the project and cannot be rebuilt. A lifecycle rule on `exports/` is fine.

## 4. Run the migration

Once, from your laptop, against the chosen database: `raw_snapshots`, `events`, `fetch_log`, the indexes, and the four empty `actual_*` columns. Per spec §4, `raw_snapshots` holds `s3_key` + `payload_sha256` rather than a `jsonb` payload, and `events` carries `week_key`, the `*_flag` columns and `source_suffix`.

**`events` gets a named `UNIQUE` constraint on the identity tuple from step 1** — `(country, normalized_title, week_key)` unless Phase 0 found legitimate same-week repeats. It has to be a constraint rather than a convention: step 5's `ON CONFLICT` needs a real target to name, and step 10's "once per `week_key`" check only means anything if the database was enforcing it all along. Decide whether `source_suffix` participates before you run this — changing a unique constraint once rows exist means deduplicating first.

Keep it as raw `.sql` applied by hand. Alembic is more machinery than three tables justify, and the next iteration adds columns that are already reserved.

## 5. Write the handler

Wrap the pipeline from step 1 in a Lambda handler, in spec §6's validation order: **guard on `fetch_log`** → fetch → reject HTML → `json.loads` → non-empty list → required fields → filter to the 8 currencies → **put raw to S3** → parse → upsert → write `fetch_log`.

The guard runs first and calls `guard.already_satisfied` from step 1 — the handler fetches the rows and acts on the answer, it doesn't re-derive it. Snapshot to S3 *before* parsing, so a parser crash still leaves the payload recoverable. Take `week_offset` and `week_final` from the event payload, so one function serves every schedule. Use the conditional upsert from spec §4 (`IS DISTINCT FROM`), targeting the constraint from step 4 — without it `events_updated` and `last_updated_at` are meaningless and step 10's checks can't distinguish a revision from a re-fetch.

Fetch with `httpx` or `requests` and an explicit timeout — a hung socket burning the whole Lambda timeout looks identical to a failed fetch in the logs but costs you the slot.

## 6. Deploy with SAM or CDK

One template holding the function, its IAM role, the schedules and the alarms, so a redeploy is one command. Runtime `python3.11`; check the Lambda runtime deprecation calendar before you settle in, since Python runtimes age out on a published schedule.

- Dependencies (`psycopg[binary]`, `httpx`, `tzdata`) go in a Lambda layer or a SAM-built package — `psycopg`'s binary wheel must be built for `manylinux`, so build it in the SAM container (`sam build --use-container`) rather than shipping your laptop's wheels.
- IAM: read one SSM parameter, write to the one bucket prefix. Nothing else.
- Timeout ~30s, **reserved concurrency 1** so two runs can never overlap.
- **Lambda retry attempts = 0** (`MaximumRetryAttempts: 0` on the async invoke config). You want the deliberate 10-minute retry slot, not an instant retry that gets you rate-limited.

## 7. EventBridge schedules

EventBridge Scheduler takes cron in UTC directly. Four daily harvest slots, the Saturday sweep passing `week_final: true`, optionally the next-week probe with `week_offset: 1`.

Then a retry rule 10 minutes after each main slot, firing unconditionally: the handler's first action is the step 1 guard, which exits immediately if the paired attempt already succeeded. This is better than sleeping inside the run — it costs nothing when things are healthy and it survives the process being killed. Since `guard.py` is already written and tested, this step is schedule configuration only; if you find yourself writing skip logic here, it belongs back in the module.

**Set the sweep time from Phase 0's measurement, not from the placeholder.** Spec §3 pencils in Sat 23:00 UTC on the assumption that rollover is at or after midnight ET; if step 0 says otherwise, move it.

## 8. Failure alerting

Handler emits a CloudWatch metric or a structured log line on failure; alarm on it, publish to an SNS topic subscribed to your email.

Add a **second, separate alarm for the Saturday sweep**. Everything else self-heals on the next slot six hours later; a missed sweep loses the closing week's final forecast state permanently. That is the one page you act on the same evening.

## 9. Export

A second small Lambda dumps `events` to CSV and JSON into `exports/` on a Sunday schedule, after the sweep — `csv.DictWriter` and `json.dump`, streamed to a temp file and uploaded. S3 versioning gives you dataset history for free.

## 10. Week-two check

After two weeks unattended, run spec §9's table as actual queries. In particular:

- `fetch_log` has an `ok` row per scheduled slot, no gap > 8h, no `rate_limited`.
- Both `week_final=true` snapshots exist **and their S3 objects are readable** — a row pointing at a missing object is worse than no row.
- Rollover added rows rather than overwriting: each identity tuple appears once per `week_key`. The step 4 constraint should have made a violation impossible, so this confirms the constraint rather than discovering duplicates — if it returns rows, the constraint is wrong, not the data. The opposite failure, `normalize_title` merging two distinct events, no constraint can catch: compare rows per `(country, week_key)` against the raw payload's count.
- Sunday events carry the same `week_key` as the Monday events after them. This is the week-identity bug's signature and week two is the first time it can show.

Clean on all of it → MVP done, and actuals (spec §10) become the next iteration with no migration.
