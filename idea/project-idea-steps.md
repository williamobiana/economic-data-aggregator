# FF Harvester — build & deploy steps

Companion to `project-idea.md` (spec v1.2). Spec says *what* and *why*; this says *in what order*.

**Superseded on conflict by `.claude/specs/ff-harvester/requirements.md` and `design.md`.** Both are newer than this document and than the spec; where any of them disagree, the spec documents win. The step order below matches the requirement numbering (step N = Requirement N).

**Target:** Python 3.11, AWS Lambda, Supabase Postgres, S3. Steps 1–3 are AWS-free. Manual deploy, no IaC.

Build in this order — each step depends on the ones above.

---

## 1. Pure modules + tests

No AWS, no database, no mocking.

- `parse.py` — spec §5. `num` expanded, `unit` a dimension, scale suffix separate, `flag` returned *and* persisted. Frozen `@dataclass`, not a dict.
- `week.py` — `ff_week_start`. Sunday-start, `America/New_York`. Not ISO weeks — `isocalendar()` starts Monday and splits each feed week.
- `identity.py` — `normalize_title` + `event_key`. The one definition of "same event". Titles were measured and do not drift, so assert the negative — same `(country, date)` across fetches yields the same title — and assert distinct events stay distinct.
- `guard.py` — `already_satisfied(rows, slot, now)`. Pure logic over query results, no connection. `now` is a parameter, not a clock read — the 5-minute rate-limit floor lives here and needs to be testable. Step 8 depends on it.

Stdlib only:

- `datetime.fromisoformat` handles the feed's offset. Never construct the offset — DST moves it to `-05:00` in November.
- `zoneinfo.ZoneInfo("America/New_York")`. No `pytz`, no `dateutil`. Add `tzdata` on Lambda.

Test `week.py` (Sunday event, Sat 20:00 ET event, either side of November DST) and `identity.py` (the cross-country title collisions, and the two same-week repeats that must not collapse) harder than the parser. A bad parse logs a warning; a bad week or identity key silently duplicates rows.

Fixtures come from the three payloads in `json-files/`. The November DST case has no real payload behind it — hand-build it.

## 2. Database — Supabase

- **Pooler connection string, port `6543`, transaction mode.** Not direct `5432` — Lambda opens a fresh connection per invocation and will exhaust the limit.
- **`psycopg` v3, binary extra.** Ignore the Supabase client libraries; this is plain Postgres.
- Free projects pause after ~7 days idle. Four fetches a day means it never fires.

Keep the string local for now; it goes into SSM in step 6.

## 3. Run the migration

Once, laptop → Supabase, raw `.sql`. No Alembic. Creates `raw_snapshots`, `events`, `fetch_log`, indexes, `superseded_at`, and the four empty `actual_*` columns (spec §4).

**`events` gets a named `UNIQUE` constraint on `(country, normalized_title, week_key, event_timestamp)`.** `source_suffix` does *not* participate — it's display provenance, not identity, and this feed doesn't emit one anyway.

Must be a constraint, not a convention: step 5's `ON CONFLICT` needs a target, and step 11's check only means something if the DB enforced it all along. Changing it later means deduplicating first.

## 4. S3 bucket

First AWS step, and it comes **before** the handler — step 5's checks assert that a `raw/` object landed and that two invocations within a minute produce two distinct keys. Neither can run against a bucket that doesn't exist.

- Prefixes `raw/<week_key>/<fetched_at>.json` and `exports/<date>/`.
- Block public access, **versioning off**, **no lifecycle expiry on `raw/`**. Lifecycle on `exports/` is fine.
- With versioning off, the `<fetched_at>` timestamp is the only thing preventing overwrites — don't simplify that key.

Local runs write to this bucket under the developer's own credentials from `~/.aws/credentials`; Lambda uses the execution role from step 6. `boto3` resolves both through its default chain, so nothing in the code changes between them. Install `boto3` locally as a dev dependency — it must not enter `requirements.txt` or the layer, since the Lambda runtime already ships it.

## 5. Write the handler

Local, against the real bucket and the real database. Deployed in step 7.

Spec §6 order: **guard on `fetch_log`** → fetch → reject HTML → `json.loads` → non-empty list → required fields → filter to the 9 tracked currencies → **put raw to S3** → parse → upsert → **supersede** → write `fetch_log`.

- Guard calls `guard.already_satisfied`. The handler fetches rows and acts on the answer; it doesn't re-derive it.
- S3 before parse, so a parser crash still leaves the payload recoverable.
- `week_final` from the event payload — one function, every schedule.
- Conditional upsert (`IS DISTINCT FROM`, spec §4) targeting the step 3 constraint. `superseded_at` is in the comparison and reset to `NULL` on update, so a returning event revives even when its values haven't changed.
- **Supersede in the same transaction as the upsert**, scoped to the `week_key` the payload itself yields — never to a `week_key` derived from the clock. Retiring rows in a week this payload doesn't cover is the one way this step can destroy data.
- `httpx`, **10s connect / 20s read timeout**. A hung socket costs the whole slot.

## 6. Remaining AWS prerequisites

Three resources, in order. The bucket is already up from step 4.

1. **SSM parameter** — `SecureString`, the pooler string from step 2. Not Secrets Manager ($0.40/secret/month breaks the $0 target).
2. **IAM role: Lambda execution** — trust `lambda.amazonaws.com`. `AWSLambdaBasicExecutionRole`, `ssm:GetParameter` on that one parameter, `kms:Decrypt`, `s3:PutObject` on `<bucket>/raw/*` and `<bucket>/exports/*`. Nothing else. Both functions share it.
3. **Lambda layer** — `psycopg[binary]`, `httpx`, `tzdata`. Build for `manylinux` in Docker/CloudShell/EC2, **not** from your laptop's `site-packages`.

## 7. Deploy the harvester, invoke by hand

1. **Create the function** — `python3.11`, timeout 30s, **reserved concurrency 1**, layer attached, env vars for bucket + parameter name, role from step 6.
2. **Async invoke config: `MaximumRetryAttempts: 0`.** Separate console screen; easiest thing here to miss. You want step 8's 10-minute retry, not an instant one that gets you rate-limited.
3. **Invoke once manually**, before any schedule. Cheapest place to catch a wrong connection string, missing IAM permission, or a layer built for the wrong platform. Confirm: a `raw/` object landed, `events` has rows, `fetch_log` has one `ok`.
4. **Set log retention to 30 days.** The log group only exists after that first invoke, and defaults to never expire.

Check the Lambda runtime deprecation calendar — with no template, a runtime bump is a manual edit per function.

## 8. EventBridge schedules

1. **Scheduler IAM role** — trust `scheduler.amazonaws.com`, `lambda:InvokeFunction` on the harvester. Needs its own role; won't reuse the Lambda one. Step 9 extends this same role to the export ARN rather than adding a second.
2. **Schedules**, cron in UTC — four daily harvest slots, Saturday sweep with `week_final: true`.
3. **Retry rule 10 minutes after each main slot**, firing unconditionally, including one for the sweep. The guard exits immediately if the paired attempt succeeded. Costs nothing when healthy, survives the process being killed. Configuration only — if you're writing skip logic here, it belongs in `guard.py`.

**Sweep goes at `0 11 * * 6` (Sat 07:00 ET).** The feed rolls over **Saturday 19:00 ET**, so the once-provisional `0 23 * * 6` sat exactly on it in summer — a coin flip on which week you fetch. Sat 07:00 ET clears the roll by about twelve hours in either season. Early costs nothing: the endpoint has no `actual` field and past rows never change.

## 9. Export function

Before alerting — step 10's health check runs inside this function.

Dumps `events` to CSV + JSON on a Sunday schedule after the sweep. `csv.DictWriter`, `json.dump`, temp file, upload. Full dump of **live rows** (`WHERE superseded_at IS NULL`), not incremental. Same role and layer.

**Write to `exports/<date>/events.csv`.** A fixed key destroys last week's dump; the date replaces version history.

Deploy as in step 7 — create, invoke by hand, set retention. Then add its ARN to the scheduler role and create the Sunday schedule.

## 10. Failure alerting

**No SNS or email for now**, so alarms are eyes-only and nothing looks at them on its own. The check moves into code.

**Add to the export function:** run the `fetch_log` gap query, log one line — `HEALTH ok slots=28 gaps=0` or `HEALTH FAIL missing_sweep`.

Create **two CloudWatch alarms**, no actions attached:

- general failure — informational; everything self-heals on the next slot.
- **Saturday sweep** — a missed sweep loses the closing week's final state permanently. Act same evening.

Wire SNS → email when remembering to look becomes the failure mode.

## 11. Week-two check

Run spec §9 as real queries.

- `fetch_log`: an `ok` row per slot, no gap > 8h, no `rate_limited`.
- Both `week_final=true` snapshots exist **and their S3 objects are readable**.
- Each identity tuple appears once per `week_key`. Rows here mean the constraint is wrong, not the data. The opposite failure — `normalize_title` merging distinct events — no constraint catches: compare **live** rows per `(country, week_key)` against the raw payload count. They must be **equal**. **Read the direction.** Below the payload count is over-merging; above it means supersession didn't run.
- Sunday events share a `week_key` with the Monday events after them. Week two is the first time the week-identity bug can show.
- Two `HEALTH` lines, both `ok`. If it never ran, the other checks told you nothing.

Clean → MVP done. Actuals (spec §10) are the next iteration, no migration.
