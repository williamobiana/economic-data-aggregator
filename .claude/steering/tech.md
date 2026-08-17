# Tech

Python 3.11 · AWS Lambda + EventBridge Scheduler · Supabase Postgres · S3.

`requirements.txt`: `psycopg[binary]`, `httpx`, `tzdata`. Nothing else. Tests run under plain `pytest` with no mocking.

## Decisions that are already settled

- **AWS, not GitHub Actions.** Actions disables scheduled workflows after ~60 days of repo inactivity — exactly the failure mode a finished archiver hits. EventBridge fires until deleted.
- **Supabase pooler, port `6543`, transaction mode.** Not direct `5432` — Lambda opens a connection per invocation and would exhaust the limit. Plain Postgres via `psycopg` v3; the Supabase client libraries are unused.
- **SSM Parameter Store `SecureString`, not Secrets Manager** ($0.40/secret/month breaks the $0 target).
- **Manual console deployment, no SAM/CDK.** Accepted cost: runtime bumps are a manual edit per function.
- **S3 versioning off.** The `<fetched_at>` in `raw/<week_key>/<fetched_at>.json` and the dated `exports/<date>/` prefix are the only things preventing overwrites. Never simplify those keys. No lifecycle expiry on `raw/`.
- **Raw payloads to S3, not a `jsonb` column.**

## Dates

- `datetime.fromisoformat` for the feed's timestamps — every row carries a full ISO-8601 offset, Holiday rows included. No `pytz`, no `dateutil`, no branching on `impact`.
- **Never hard-code `-04:00`.** It becomes `-05:00` in November. No real payload exercises that yet — hand-build the fixture.
- `zoneinfo.ZoneInfo("America/New_York")` for week arithmetic. Ship `tzdata` — the Lambda image has no system tz database.
- `Decimal`, not `float` — parsed values land in `numeric`.

## Correctness rules that fail silently

- **Week key is Sunday-start, computed in New York.** `isocalendar()` starts Monday and splits every feed week; UTC puts a Sat 20:00 ET event into Sunday. Derive from payload contents (`ff_week_start(min(date))`), never from fetch time.
- **`events_identity` is a named UNIQUE constraint** on `(country, normalized_title, week_key, event_timestamp)`. `source_suffix` is display provenance and stays out. The upsert's `ON CONFLICT` needs a real target, and the week-two check proves nothing unless the DB enforced identity from the first insert. Changing it after rows exist means deduplicating first.
- **Conditional upsert** using `IS DISTINCT FROM` (not `<>`, so NULLs compare correctly), or `last_updated_at` degrades to "last fetched". `superseded_at` is in the comparison tuple and reset to `NULL`, so a returning event revives even when its values are unchanged.
- **Supersession runs in the same transaction as the upsert**, scoped to the `week_key` the payload itself yields. Scoping it by a clock-derived `week_key` is the one way this system destroys data. Soft delete only; reads default to `WHERE superseded_at IS NULL`. Self-healing — a bad supersede costs one slot.
- **Snapshot raw to S3 before parsing**, so a parser crash still leaves the payload recoverable.

## Fetch policy

`httpx`, 10s connect / 20s read. Rate-limit responses are **HTML, not JSON** — check content type / first char before `json.loads` and abort. On any failure: write `fetch_log` and exit. **No in-process sleep-retry** — the paired retry slot handles it. 5-minute hard floor between requests, `reserved_concurrency=1`, Lambda async `MaximumRetryAttempts: 0`.

## AWS resources

S3 bucket (public access blocked) · SSM `SecureString` · one Lambda execution role shared by both functions (`AWSLambdaBasicExecutionRole`, `ssm:GetParameter` on the one parameter, `kms:Decrypt`, `s3:PutObject` on `raw/*` and `exports/*`, nothing else) · a scheduler role trusting `scheduler.amazonaws.com` · a layer with `psycopg[binary]`, `httpx`, `tzdata` built for **manylinux**, not from a laptop's `site-packages`.

Set log retention to 30 days after the first invoke — the log group does not exist before it and defaults to never expire.
