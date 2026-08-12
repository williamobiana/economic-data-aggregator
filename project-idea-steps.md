# FF Harvester — build & deploy steps

Companion to `project-idea.md` (spec v1.1). Spec says *what* and *why*; this says *in what order*. Spec wins on conflicts.

**Target:** Python 3.11, AWS Lambda, Supabase Postgres, S3. Steps 0–4 are AWS-free. Manual deploy, no IaC.

Build in this order — each step depends on the ones above.

---

## 0. Observe the feed first

Two `curl`s and a weekend. Honour the 5-minute floor. These are the only open decisions left.

- Does `ff_calendar_nextweek.json` exist? → decides the next-week probe.
- **When does the week roll over?** Fetch Sat evening + Sun morning, diff. Sets the sweep time.
- What does `date` carry for All-Day / Tentative / Holiday rows?
- **Do titles drift within a week?** Diff the same event Wed vs Sat. Whatever drifts, `normalize_title` must absorb.
- **Does `(country, title)` ever repeat in one week?** Assume no unless observed; if yes, add event date to the identity tuple (step 3).
- Save every payload — test fixtures, and the only record of this week.

## 1. Pure modules + tests

No AWS, no database, no mocking.

- `parse.py` — spec §5. `num` expanded, `unit` a dimension, scale suffix separate, `flag` returned *and* persisted. Frozen `@dataclass`, not a dict.
- `week.py` — `ff_week_start`. Sunday-start, `America/New_York`. Not ISO weeks — `isocalendar()` starts Monday and splits each feed week.
- `identity.py` — `normalize_title` + `event_key`. The one definition of "same event". Assert Phase 0's drift cases collapse and distinct events don't.
- `guard.py` — `already_satisfied(rows, slot)`. Pure logic over query results, no connection. Step 7 depends on it.

Stdlib only:

- `datetime.fromisoformat` handles the feed's offset. Never construct the offset — DST moves it to `-05:00` in November.
- `zoneinfo.ZoneInfo("America/New_York")`. No `pytz`, no `dateutil`. Add `tzdata` on Lambda.

Test `week.py` (Sunday event, Sat 20:00 ET event, either side of November DST) and `identity.py` (every Phase 0 variant) harder than the parser. A bad parse logs a warning; a bad week or identity key silently duplicates rows.

## 2. Database — Supabase

- **Pooler connection string, port `6543`, transaction mode.** Not direct `5432` — Lambda opens a fresh connection per invocation and will exhaust the limit.
- **`psycopg` v3, binary extra.** Ignore the Supabase client libraries; this is plain Postgres.
- Free projects pause after ~7 days idle. Four fetches a day means it never fires.

Keep the string local for now; it goes into SSM in step 5.

## 3. Run the migration

Once, laptop → Supabase, raw `.sql`. No Alembic. Creates `raw_snapshots`, `events`, `fetch_log`, indexes, and the four empty `actual_*` columns (spec §4).

**`events` gets a named `UNIQUE` constraint on `(country, normalized_title, week_key)`.** `source_suffix` does *not* participate — it's drift for `normalize_title` to absorb. Add event date only if Phase 0 found real same-week repeats.

Must be a constraint, not a convention: step 4's `ON CONFLICT` needs a target, and step 10's check only means something if the DB enforced it all along. Changing it later means deduplicating first.

## 4. Write the handler

Local; runs against AWS in step 6.

Spec §6 order: **guard on `fetch_log`** → fetch → reject HTML → `json.loads` → non-empty list → required fields → filter to 8 currencies → **put raw to S3** → parse → upsert → write `fetch_log`.

- Guard calls `guard.already_satisfied`. The handler fetches rows and acts on the answer; it doesn't re-derive it.
- S3 before parse, so a parser crash still leaves the payload recoverable.
- `week_offset` and `week_final` from the event payload — one function, every schedule.
- Conditional upsert (`IS DISTINCT FROM`, spec §4) targeting the step 3 constraint.
- `httpx`, **10s connect / 20s read timeout**. A hung socket costs the whole slot.

## 5. AWS prerequisites

First AWS step. Four resources, in order.

1. **S3 bucket** — prefixes `raw/<week_key>/<fetched_at>.json` and `exports/`. Block public access, **versioning off**, **no lifecycle expiry on `raw/`**. Lifecycle on `exports/` is fine.
   With versioning off, the `<fetched_at>` timestamp is the only thing preventing overwrites — don't simplify that key.
2. **SSM parameter** — `SecureString`, the pooler string from step 2. Not Secrets Manager ($0.40/secret/month breaks the $0 target).
3. **IAM role: Lambda execution** — trust `lambda.amazonaws.com`. `AWSLambdaBasicExecutionRole`, `ssm:GetParameter` on that one parameter, `kms:Decrypt`, `s3:PutObject` on `<bucket>/raw/*` and `<bucket>/exports/*`. Nothing else. Both functions share it.
4. **Lambda layer** — `psycopg[binary]`, `httpx`, `tzdata`. Build for `manylinux` in Docker/CloudShell/EC2, **not** from your laptop's `site-packages`.

## 6. Deploy the harvester, invoke by hand

1. **Create the function** — `python3.11`, timeout 30s, **reserved concurrency 1**, layer attached, env vars for bucket + parameter name, role from step 5.
2. **Async invoke config: `MaximumRetryAttempts: 0`.** Separate console screen; easiest thing here to miss. You want step 7's 10-minute retry, not an instant one that gets you rate-limited.
3. **Invoke once manually**, before any schedule. Cheapest place to catch a wrong connection string, missing IAM permission, or a layer built for the wrong platform. Confirm: a `raw/` object landed, `events` has rows, `fetch_log` has one `ok`.
4. **Set log retention to 30 days.** The log group only exists after that first invoke, and defaults to never expire.

Check the Lambda runtime deprecation calendar — with no template, a runtime bump is a manual edit per function.

## 7. EventBridge schedules

1. **Scheduler IAM role** — trust `scheduler.amazonaws.com`, `lambda:InvokeFunction` on the harvester. Needs its own role; won't reuse the Lambda one. Step 8 adds the export ARN.
2. **Schedules**, cron in UTC — four daily harvest slots, Saturday sweep with `week_final: true`, next-week probe with `week_offset: 1` only if Phase 0 confirmed the file.
3. **Retry rule 10 minutes after each main slot**, firing unconditionally. The guard exits immediately if the paired attempt succeeded. Costs nothing when healthy, survives the process being killed. Configuration only — if you're writing skip logic here, it belongs in `guard.py`.

**Set the sweep time from Phase 0, not the placeholder.** Spec §3's Sat 23:00 UTC assumes rollover at or after midnight ET.

## 8. Export function

Before alerting — step 9's health check runs inside this function.

Dumps `events` to CSV + JSON on a Sunday schedule after the sweep. `csv.DictWriter`, `json.dump`, temp file, upload. Full dump, not incremental. Same role and layer.

**Write to `exports/<date>/events.csv`.** A fixed key destroys last week's dump; the date replaces version history.

Deploy as in step 6 — create, invoke by hand, set retention. Then add its ARN to the scheduler role and create the Sunday schedule.

## 9. Failure alerting

**No SNS or email for now**, so alarms are eyes-only and nothing looks at them on its own. The check moves into code.

**Add to the export function:** run the `fetch_log` gap query, log one line — `HEALTH ok slots=28 gaps=0` or `HEALTH FAIL missing_sweep`.

Create **two CloudWatch alarms**, no actions attached:

- general failure — informational; everything self-heals on the next slot.
- **Saturday sweep** — a missed sweep loses the closing week's final state permanently. Act same evening.

Wire SNS → email when remembering to look becomes the failure mode.

## 10. Week-two check

Run spec §9 as real queries.

- `fetch_log`: an `ok` row per slot, no gap > 8h, no `rate_limited`.
- Both `week_final=true` snapshots exist **and their S3 objects are readable**.
- Each identity tuple appears once per `week_key`. Rows here mean the constraint is wrong, not the data. The opposite failure — `normalize_title` merging distinct events — no constraint catches: compare rows per `(country, week_key)` against the raw payload count.
- Sunday events share a `week_key` with the Monday events after them. Week two is the first time the week-identity bug can show.
- Two `HEALTH` lines, both `ok`. If it never ran, the other checks told you nothing.

Clean → MVP done. Actuals (spec §10) are the next iteration, no migration.
