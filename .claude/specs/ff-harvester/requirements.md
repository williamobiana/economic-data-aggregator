# Requirements Document

## Introduction

FF Harvester archives the ForexFactory economic calendar feed — schedule, forecast, previous — into Postgres on a schedule. The feed publishes the current week only and `ff_calendar_nextweek.json` 404s, so a week that is missed is lost permanently. The governing objective is **never miss a week, never corrupt a value**.

Derived from `idea/project-idea.md` (spec v1.2) and `idea/project-idea-steps.md` (build order — the newer document, which wins on conflicts). Scope and settled technical constraints are fixed by `.claude/steering/product.md`, `tech.md` and `structure.md`.

Each requirement below is a **gate**: an objective, a user story, acceptance criteria in EARS form, and the exact check that proves it. Requirement N+1 SHALL NOT begin until requirement N's gate passes. Requirements 1–4 need neither network, AWS nor database; 5–6 need a database; 7 onward need AWS as well.

**Notation.** EARS keywords (`WHEN`, `IF`, `WHERE`, `WHILE`) are followed by `SHALL`; a criterion with no keyword is ubiquitous and always applies. `[DERIVED]` marks a criterion stated in neither source document — a decision made here to make the gate testable. `[OPEN]` marks a conflict or gap the source documents do not resolve.

**Non-functional requirements** are carried by the global invariants G1–G7 below, which apply across every numbered requirement rather than belonging to any one of them.

---

## Global Invariants

Any change to one of these is a change to the whole system, not to a single module.

- **G1** — THE SYSTEM SHALL treat a missed week as permanent data loss. There is no backfill: `ff_calendar_nextweek.json` 404s and the feed publishes the current week only.
- **G2** — THE SYSTEM SHALL NOT drop an event for any reason short of its absence from the payload. IF a value fails to parse THEN THE SYSTEM SHALL log a warning and store the raw string regardless.
- **G3** — WHERE a week is derived THE SYSTEM SHALL use the single `week.ff_week_start` implementation, and WHERE identity is determined THE SYSTEM SHALL use the single `identity.normalize_title` / `identity.event_key` pair. No other module re-derives either.
- **G4** — WHERE an operation is week-scoped THE SYSTEM SHALL take its `week_key` from the payload's own contents (`ff_week_start(min(date))`), never from the clock.
- **G5** — WHEN a row leaves its week's payload THE SYSTEM SHALL mark it dead via `superseded_at` rather than delete it, and THE SYSTEM SHALL NOT delete or expire any raw payload.
- **G6** — THE SYSTEM SHALL incur $0/month: no Secrets Manager, no NAT gateway, no VPC, no servers, no IaC.
- **G7** — WHERE a module under `src/` is not `handler.py` or `export.py` THE SYSTEM SHALL keep it pure — stdlib only, no AWS imports, no database connection — so that `pytest` runs with no mocking.

---

## Requirements

### Requirement 1 — Value parser

**Objective.** Turn a published display string into a frozen `Value` without ever losing the original.

**User Story:** As the archive, I want `"-14.5B"` stored as both the published string and `-14500000000`, so that a later consumer needs neither a parser nor access to the feed.

#### Acceptance Criteria

1. WHERE a value is parsed THE SYSTEM SHALL return a frozen `@dataclass` `Value(num, unit, suffix, flag)`, not a dict.
2. THE SYSTEM SHALL represent `num` as `Decimal` or `None`, and SHALL NOT use `float`.
3. WHEN a scale suffix is present THE SYSTEM SHALL return `num` fully expanded and SHALL report `suffix` as provenance only; no caller SHALL re-apply it.
4. THE SYSTEM SHALL produce exactly this mapping:

   | Input | num | unit | suffix | flag |
   |---|---|---|---|---|
   | `"1.1%"` | `1.1` | `percent` | — | — |
   | `"-13.4"` | `-13.4` | `none` | — | — |
   | `"-14.5B"` | `-14500000000` | `count` | `B` | — |
   | `"215K"` | `215000` | `count` | `K` | — |
   | `"7.6M"` | `7600000` | `count` | `M` | — |
   | `""` | `None` | `None` | — | — |
   | `"<0.1%"` | `0.1` | `percent` | — | `lt` |
   | anything else | `None` | `None` | — | — + warning |

5. IF the input is blank THEN THE SYSTEM SHALL treat it as valid and SHALL NOT warn — 80 of 244 measured rows are blank.
6. WHEN whitespace or thousands separators are present THE SYSTEM SHALL strip them before parsing.
7. IF the input matches no known shape, including the bond-auction form `"4.58|2.6"`, THEN THE SYSTEM SHALL log one warning and return an all-`None` `Value`, and SHALL NOT raise.
8. THE SYSTEM SHALL NOT branch on `impact`.

#### Validation Gate

`pytest tests/test_parse.py` green, with one case per table row plus `"4.58|2.6"`, a comma-bearing input and a whitespace-padded input. Assert `isinstance(v.num, Decimal)` and that `Value` raises on attribute assignment.

---

### Requirement 2 — Week key

**Objective.** One function that files every event of a feed week under one key.

**User Story:** As the archive, I want a Saturday 20:00 ET event and the Sunday event following it filed under the same key, so that a week is one row-set rather than two.

#### Acceptance Criteria

1. WHEN given an instant THE SYSTEM SHALL return the `date` of the Sunday beginning the FF week containing it, evaluated in `America/New_York`.
2. THE SYSTEM SHALL NOT use `isocalendar()` or any Monday-start arithmetic — ISO weeks split every captured feed week.
3. WHEN the input falls on a Saturday at 20:00 ET, and is therefore already Sunday in UTC, THE SYSTEM SHALL key it to the *earlier* Sunday.
4. WHEN the input falls on either side of the November DST transition THE SYSTEM SHALL key it correctly, taking the offset from the input rather than assuming one.
5. WHERE a payload's key is derived THE SYSTEM SHALL compute `ff_week_start(min(date))` over the payload's rows.

#### Validation Gate

`pytest tests/test_week.py` green, with at minimum:

- a Sunday event → keys to itself;
- a Saturday 20:00 ET event → keys to the Sunday *before* it, and to the same key as the Monday event following it;
- an event before the November change (`-04:00`) and one after (`-05:00`) → each keys correctly. **No real payload exercises `-05:00`; hand-build this fixture.**
- a real payload from `json-files/` → all rows collapse to exactly one key. Regression assertion against the measured split: `isocalendar()` yields `(2026,32)`×3 + `(2026,33)`×71 for the week of 08-09 and `(2026,33)`×11 + `(2026,34)`×85 for the week of 08-16, while `ff_week_start` yields one key each.

---

### Requirement 3 — Event identity

**Objective.** The one definition of "same event", strong enough that the database can enforce it.

**User Story:** As the archive, I want a re-fetched event to update its existing row while a genuinely different event gets its own, so that neither duplicates nor merges appear.

#### Acceptance Criteria

1. WHEN normalizing a title THE SYSTEM SHALL trim, collapse internal whitespace, lowercase, and unescape slashes (`m\/m` → `m/m`).
2. WHERE `event_key` is used THE SYSTEM SHALL compute it as a sha1 over the four identity parts `(country, normalized_title, week_key, event_timestamp)`, and SHALL treat it as convenience rather than authority — the database constraint is the authority.
3. THE SYSTEM SHALL NOT admit `source_suffix` into identity; it is display provenance, and this feed emits none.
4. THE SYSTEM SHALL include `event_timestamp` at full precision. Truncating to the date merges two rows carrying different `previous`; dropping it merges a speaker appearing twice in one week.
5. WHEN the same `(country, date)` event appears across successive fetches THE SYSTEM SHALL yield an identical `normalized_title` — titles were measured across 137 rows and do not drift.

#### Validation Gate

`pytest tests/test_identity.py` green, with:

- **the negative assertion** — the same `(country, date)` across the payloads in `json-files/` yields an identical normalized title and an identical `event_key`;
- `FOMC Member Hammack Speaks` twice in one week → two distinct keys;
- the ADP 08:14 / 08:15 pair → two distinct keys;
- the measured cross-country collisions → distinct keys per country: `Flash Manufacturing PMI` (AUD, EUR, GBP, JPY, USD), `Flash Services PMI` (AUD, EUR, GBP, USD), `Unemployment Rate` (AUD, CNY, GBP), and the remaining 14 of the 17 measured collisions;
- normalization equivalence — `"  German  Factory\/Orders m\/m  "` and `"German Factory/Orders m/m"` → one key.

---

### Requirement 4 — Retry guard

**Objective.** All retry skip logic in one pure function, so the schedule can stay dumb.

**User Story:** As the operator, I want the +10 minute retry to fire unconditionally and exit for free when its paired attempt already succeeded, so that retry coverage costs nothing when the system is healthy.

#### Acceptance Criteria

1. WHERE retry suppression is decided THE SYSTEM SHALL evaluate `already_satisfied` over `fetch_log` rows supplied by the caller, together with a caller-supplied reference instant, and SHALL return a boolean. `[DERIVED]` — the reference instant is a parameter rather than a clock read, because criterion 4 needs one and G7 forbids a non-deterministic pure module.
2. IF the paired main attempt for the slot recorded outcome `ok` THEN THE SYSTEM SHALL report the slot satisfied.
3. IF the paired attempt recorded `rate_limited`, `parse_error`, `network` or `empty`, or no attempt is recorded, THEN THE SYSTEM SHALL report the slot unsatisfied.
4. IF any attempt occurred within 5 minutes of the reference instant THEN THE SYSTEM SHALL report the slot satisfied, enforcing the rate-limit floor. `[DERIVED]` — both documents state the 5-minute hard floor but neither sites it; the guard is the only pure place it can live.
5. THE SYSTEM SHALL NOT open a connection or issue a query from within the guard.
6. THE SYSTEM SHALL NOT place skip logic in the EventBridge configuration or in `handler.py`; schedules are configuration only.

#### Validation Gate

`pytest tests/test_guard.py` green, covering each outcome value, the empty-rows case, the within-5-minutes case at a fixed reference instant, and a row belonging to a *different* slot (must not satisfy). Then `grep -rn "already_satisfied\|outcome" src/handler.py` shows the handler calling the function and acting on the answer, never re-deriving it.

---

### Requirement 5 — Database reachable

**Objective.** A pooled connection that survives Lambda's connection-per-invocation pattern.

**User Story:** As the operator, I want the connection proven from a laptop before any schema exists, so that a connection failure and a schema failure are never diagnosed as one problem.

#### Acceptance Criteria

1. WHERE the database is reached THE SYSTEM SHALL use the Supabase pooler string on port `6543` in transaction mode, and SHALL NOT use direct `5432` — per-invocation connections exhaust its limit.
2. WHERE database access occurs THE SYSTEM SHALL use plain Postgres via `psycopg` v3 (`psycopg[binary]`), and the Supabase client libraries SHALL NOT be a dependency.
3. THE SYSTEM SHALL declare exactly `psycopg[binary]`, `httpx` and `tzdata` in `requirements.txt`.
4. WHILE this requirement is the active gate THE SYSTEM SHALL hold the connection string locally and SHALL NOT require any secret store, so that connectivity is proven independently of how the credential is later distributed.

#### Validation Gate

A one-liner from the laptop against the pooler string returns `SELECT 1`, and the port in the string is `6543`. Free projects pause after ~7 days idle — four fetches a day means this never fires, so no keepalive is required.

---

### Requirement 6 — Schema migration

**Objective.** Identity enforced by the database from the first insert.

**User Story:** As the operator, I want identity enforced by the database from the very first insert, so that any later duplicate check is evidence rather than an assumption — a constraint added after rows exist proves nothing about the rows already there.

#### Acceptance Criteria

1. THE SYSTEM SHALL apply `migrations/001.sql` as raw SQL, hand-run once from laptop to Supabase, with no migration framework.
2. THE SYSTEM SHALL create `raw_snapshots`, `events`, `fetch_log`, and the three indexes `events (event_timestamp)`, `events (country, event_timestamp)`, and `events (week_key) WHERE superseded_at IS NULL`.
3. WHERE identity is enforced THE SYSTEM SHALL declare a **named** constraint `events_identity UNIQUE (country, normalized_title, week_key, event_timestamp)`; a convention SHALL NOT substitute, because `ON CONFLICT` needs a real target.
4. THE SYSTEM SHALL NOT admit `source_suffix` into that constraint.
5. THE SYSTEM SHALL include `superseded_at timestamptz` and the four reserved columns `actual_raw`, `actual_num`, `actual_source`, `actual_first_seen_at`, all NULL in the MVP, so that the actuals iteration needs no migration.
6. THE SYSTEM SHALL store `forecast` and `previous` twice each — `*_raw` (text, as published) and `*_num` (`numeric`) — plus `*_flag`.
7. THE SYSTEM SHALL define `fetch_log` with `fetched_at, week_final, http_status, outcome, events_seen, events_new, events_updated, events_superseded`, constraining `outcome` to `('ok','rate_limited','parse_error','network','empty')`.

#### Validation Gate

Against the live database:

```sql
SELECT conname FROM pg_constraint WHERE conname = 'events_identity';          -- 1 row
SELECT indexdef FROM pg_indexes WHERE tablename = 'events';                   -- 3 + pkey + unique
INSERT ... same (country, normalized_title, week_key, event_timestamp) twice; -- second raises 23505
```

The duplicate insert raising `unique_violation` is the gate. Changing this constraint after rows exist means deduplicating first — get it right here.

---

### Requirement 7 — Object store

**Objective.** One bucket, correctly configured, before anything writes to it.

**User Story:** As the operator, I want the bucket standing and its overwrite protection settled before anything writes to it, so that raw snapshots land in real storage from the first write and no key scheme has to be retrofitted.

#### Acceptance Criteria

1. WHERE objects are stored THE SYSTEM SHALL use one S3 bucket with public access blocked and **versioning off**, keyed `raw/<week_key>/<fetched_at>.json` and `exports/<date>/`.
2. WHILE versioning is off, `<fetched_at>` and `<date>` are the only protection against overwrite, and THE SYSTEM SHALL NOT simplify either key.
3. THE SYSTEM SHALL apply no lifecycle expiry to `raw/`; lifecycle on `exports/` is permitted.
4. THE SYSTEM SHALL grant write access to `raw/` to the credentials a developer runs the harvester under locally. `[DERIVED]` — neither source document says how a locally run handler authenticates; the build order assumes the handler meets AWS only at deploy time, which cannot hold once a local run has to write a snapshot.

#### Validation Gate

- `aws s3api get-bucket-versioning` returns empty or `Suspended`; `get-public-access-block` all true; `get-bucket-lifecycle-configuration` contains no rule matching `raw/`.
- A put and get round-trip from the laptop under `raw/` succeeds and the bytes match — this proves the local credentials, not just the bucket.

---

### Requirement 8 — Harvester handler, run locally

**Objective.** One function, every schedule, running end-to-end from a laptop.

**User Story:** As the operator, I want a full run to produce a raw snapshot, correct rows and one `fetch_log` line before any AWS resource exists, so that a failure is not obscured by infrastructure.

#### Acceptance Criteria

1. WHERE an invocation runs THE SYSTEM SHALL execute this order unmodified: `guard.already_satisfied` → HTTP 200 → HTML check (content type / first character) → `json.loads` → non-empty list → required fields → filter to the 9 tracked currencies → **put raw to S3** → parse → upsert → supersede → write `fetch_log`.
2. WHERE the handler runs THE SYSTEM SHALL orchestrate only; week arithmetic, identity, parsing and guard logic SHALL live in their own modules.
3. THE SYSTEM SHALL track `USD EUR GBP JPY AUD NZD CAD CHF CNY` at all impact levels including `Holiday`, and SHALL NOT filter by impact.
4. IF the response body is HTML, indicating a rate-limit reply, THEN THE SYSTEM SHALL abort before `json.loads`, write `fetch_log` with outcome `rate_limited`, and exit.
5. IF any attempt fails — non-JSON, HTTP error, empty array, or network — THEN THE SYSTEM SHALL write `fetch_log` and exit, and SHALL NOT sleep-retry in process; the paired retry slot handles it.
6. WHERE HTTP is performed THE SYSTEM SHALL apply a 10-second connect and 20-second read timeout.
7. WHEN validation succeeds THE SYSTEM SHALL write the raw payload to `raw/<week_key>/<fetched_at>.json` **before** parsing, so that a parser crash leaves the payload recoverable, and SHALL record `s3_key`, `byte_size` and `payload_sha256` in `raw_snapshots`.
8. WHEN reading timestamps THE SYSTEM SHALL use `datetime.fromisoformat` and convert to UTC at ingestion, and SHALL NOT construct or hard-code an offset.
9. WHEN upserting THE SYSTEM SHALL target `ON CONFLICT ON CONSTRAINT events_identity`, compare with `IS DISTINCT FROM` rather than `<>`, hold `superseded_at` inside the comparison tuple, and reset it to `NULL` on update — so that a returning event revives even when its values are unchanged and `last_updated_at` never degrades to "last fetched".
10. WHEN the upsert completes THE SYSTEM SHALL supersede within the **same transaction**, scoped to the `week_key` the payload itself yields, retiring live rows in that week whose `event_key` is absent from the payload. Scoping this by a clock-derived `week_key` is the one way this system destroys data.
11. WHERE rows are read THE SYSTEM SHALL default to `WHERE superseded_at IS NULL`.
12. WHERE `week_final` is needed THE SYSTEM SHALL take it from the invocation event payload, so that one function serves every schedule.
13. IF an unknown `impact` variant is encountered THEN THE SYSTEM SHALL log it and SHALL NOT reject the row.

#### Validation Gate

Run locally twice in succession against the real feed, respecting the 5-minute floor, then:

- run 1: `fetch_log` has one `ok`; `events_new` equals the filtered payload count; a `raw/` object exists and its sha256 matches `raw_snapshots.payload_sha256`;
- run 2: `events_new=0, events_updated=0, events_superseded=0` — **idempotence, checked before deploying anything**;
- `SELECT count(DISTINCT week_key) FROM events` = 1;
- **the measured supersession case** — against a clean database, harvest `json-files/this_WED.json` then `this_SAT.json` and assert **74 live rows and 5 superseded**; without supersession this yields 79. The five are `AUD RBA Gov Bullock Speaks`, `CNY Foreign Direct Investment ytd/y`, `CNY M2 Money Supply y/y`, `CNY New Loans`, `USD Mortgage Delinquencies`;
- **the revive case** — re-harvest `this_WED.json` and assert all five return to live. This proves criterion 9's `superseded_at` reset, which fails silently if `superseded_at` is left out of the comparison tuple;
- **the scope case** — harvest `this_SUN.json`, a different week, and assert the week-of-08-09 rows are untouched;
- a saved HTML rate-limit body is fed in and produces `rate_limited` with zero writes to `events`;
- invoke twice within one minute and confirm two distinct `raw/` keys — with versioning off, `<fetched_at>` is the only thing preventing an overwrite.

---

### Requirement 9 — Remaining AWS prerequisites

**Objective.** Three resources, least privilege, no monthly cost.

**User Story:** As the operator, I want credentials, permissions and dependencies scoped before any function exists, so that a permission failure surfaces as a permission failure rather than as a broken handler.

#### Acceptance Criteria

1. WHERE the database credential is stored THE SYSTEM SHALL use an SSM `SecureString`, and SHALL NOT use Secrets Manager — $0.40/secret/month breaks G6.
2. WHERE the functions execute THE SYSTEM SHALL grant one shared role trusting `lambda.amazonaws.com` carrying `AWSLambdaBasicExecutionRole`, `ssm:GetParameter` on that single parameter, `kms:Decrypt`, and `s3:PutObject` on `<bucket>/raw/*` and `<bucket>/exports/*`, and nothing else.
3. WHERE dependencies are shipped THE SYSTEM SHALL provide a layer carrying `psycopg[binary]`, `httpx` and `tzdata`, built for **manylinux** in Docker, CloudShell or EC2 rather than from a laptop's `site-packages`. `tzdata` is mandatory — the Lambda image has no system tz database.

#### Validation Gate

- `aws ssm get-parameter --with-decryption` returns the pooler string and its `Type` is `SecureString`.
- The role's inline policy JSON matches criterion 2 exactly — presence is not the test; **absence of anything else is**, with no wildcards beyond the two prefixes.
- Unzip the layer and confirm `psycopg_binary` ships `*.so` files with `manylinux` in their wheel provenance, and that `zoneinfo` data is present.

---

### Requirement 10 — Harvester deployed and hand-invoked

**Objective.** Catch a wrong connection string, a missing permission or a wrong-platform layer at the cheapest possible moment — before any schedule exists.

**User Story:** As the operator, I want one manual invocation to prove the deployed path end to end, so that the first scheduled run is not also the first real run.

#### Acceptance Criteria

1. WHERE the function is configured THE SYSTEM SHALL use `python3.11`, a 30-second timeout, **reserved concurrency 1**, the shared execution role, the dependency layer, and environment variables for bucket and parameter name.
2. WHERE asynchronous invocation is configured THE SYSTEM SHALL set **`MaximumRetryAttempts: 0`**, so that a failed invocation is never retried by Lambda itself. Recovery belongs to a scheduled retry ten minutes later; an immediate in-place retry gets the IP rate-limited. This is a separate console screen and the easiest setting here to miss.
3. THE SYSTEM SHALL be invoked manually at least once **before** any schedule is created.
4. WHEN the first invocation has created the log group THE SYSTEM SHALL set its retention to 30 days — the group does not exist beforehand and defaults to never expire.
5. THE SYSTEM SHALL have its runtime checked against the Lambda deprecation calendar and the date noted; with no template, a runtime bump is a manual edit per function.

#### Validation Gate

After the manual invoke: a new object under `raw/`, new rows in `events`, exactly one `ok` in `fetch_log` for that time, and `aws logs describe-log-groups` showing `retentionInDays: 30`. Read reserved concurrency and `MaximumRetryAttempts` back from the deployed configuration rather than assuming them.

---

### Requirement 11 — Schedules

**Objective.** Fire until deleted, and never sweep on the wrong side of rollover.

**User Story:** As the operator, I want the harvest to continue unattended for months without my touching the repository, so that the archive survives exactly the quiet period that would disable it elsewhere.

#### Acceptance Criteria

1. WHERE schedules invoke the function THE SYSTEM SHALL use a dedicated role trusting `scheduler.amazonaws.com` with `lambda:InvokeFunction` on the harvester, and SHALL NOT reuse the Lambda execution role. WHEN a further function is placed on a schedule THE SYSTEM SHALL extend this same role to that function's ARN rather than create a second scheduler role.
2. THE SYSTEM SHALL run harvest schedules at `0 0,6,12,18 * * *` UTC.
3. THE SYSTEM SHALL run the week-final sweep at `0 11 * * 6` UTC (Sat 07:00 ET) with input `{"week_final": true}`, and SHALL NOT use spec §3's Sat 23:00 UTC. Rollover was bracketed to the 24 hours after Sat 07:48 ET and never pinned; a sweep on the wrong side loses the closing week for good. Early costs nothing — no `actual` field, and past rows are byte-identical across re-fetches.
4. THE SYSTEM SHALL fire a retry 10 minutes after each main slot unconditionally — `10 0,6,12,18 * * *`, and `10 11 * * 6` carrying `week_final: true` for the sweep. `[DERIVED]` — the sweep's own retry cron is implied by "each slot + 10 min" but stated in neither document.
5. THE SYSTEM SHALL NOT place skip logic in retry schedules; IF skip logic is being written there THEN it belongs in `guard.py`.
6. WHERE scheduling is provided THE SYSTEM SHALL use EventBridge rather than GitHub Actions, which disables scheduled workflows after ~60 days of repository inactivity — exactly the failure mode a finished archiver hits.

#### Validation Gate

Let one full day pass unattended, then: `fetch_log` shows four `ok` rows at the expected hours; each retry slot appears as an immediate no-op, guard-satisfied, with no second fetch within 5 minutes of an `ok`; zero `rate_limited`. Then inject a failure — temporarily break the SSM parameter before one main slot — and confirm the paired retry recovers it.

---

### Requirement 12 — Export function

**Objective.** A dated, dumb, complete dump.

**User Story:** As a consumer, I want the archive as flat files on a dated path each week, so that the data is usable without Postgres and last week's dump survives this week's.

#### Acceptance Criteria

1. WHERE export runs THE SYSTEM SHALL dump `events` to CSV and JSON via `csv.DictWriter` / `json.dump` to a temp file and then upload, in full rather than incrementally.
2. THE SYSTEM SHALL write to `exports/<date>/events.csv` and `exports/<date>/events.json`; a fixed key destroys the previous dump, and the date replaces version history.
3. THE SYSTEM SHALL share the harvester's execution role and dependency layer, and SHALL be created, invoked by hand and given a 30-day log retention before any schedule is attached to it.
4. WHEN the function is proven THE SYSTEM SHALL add its ARN to the scheduler role and create a Sunday schedule running **after** the Saturday sweep. `[DERIVED]` — the export cron is unpinned by both documents; `0 12 * * 0` sits comfortably after `0 11 * * 6`.
5. `[OPEN]` — spec §6 says the export dumps **live rows**; spec §9 checks the export row count against `SELECT count(*) FROM events`. Resolved here as: dump live rows, and qualify the §9 check with `WHERE superseded_at IS NULL`. Record the choice, because the two counts diverge the first time anything is superseded.

#### Validation Gate

Hand-invoke, then: both objects exist under today's `exports/<date>/`, both are readable, and the CSV row count equals `SELECT count(*) FROM events WHERE superseded_at IS NULL`. Invoke again the next day and confirm the previous day's prefix is untouched.

---

### Requirement 13 — Health line and alarms

**Objective.** Make the absence of failure observable without paying for anything.

**User Story:** As the operator, I want the gap check to run inside the system, so that noticing a failure does not depend on my remembering to look.

#### Acceptance Criteria

1. WHEN the export function runs THE SYSTEM SHALL execute the `fetch_log` gap query and log exactly one line: `HEALTH ok slots=28 gaps=0` or `HEALTH FAIL <reason>`.
2. IF the Saturday sweep is absent THEN THE SYSTEM SHALL report it explicitly rather than as a generic gap.
3. THE SYSTEM SHALL provide two CloudWatch alarms with **no actions attached**: a general failure alarm, informational because everything self-heals on the next slot; and a **Saturday sweep** alarm, because a missed sweep loses the closing week's final state permanently and warrants acting the same evening.
4. THE SYSTEM SHALL NOT wire SNS or email in this iteration. Wire it when remembering to look becomes the failure mode.

#### Validation Gate

Invoke the export function and find one `HEALTH` line in its log group. Then force the failure path — remove a slot's `fetch_log` row in a scratch copy, or run the query against a seeded gap — and confirm the line reads `HEALTH FAIL`. Both alarms exist and list zero alarm actions.

---

### Requirement 14 — Two-week unattended acceptance

**Objective.** The MVP is done when two consecutive weeks pass untouched and every check is a query someone actually ran.

**User Story:** As the operator, I want the definition of done to be nine executable checks rather than an impression, so that "it works" is a claim with evidence behind it.

#### Acceptance Criteria

WHILE two consecutive weeks run unattended:

1. `fetch_log` SHALL show an `ok` row per scheduled slot, **no gap > 8h**, and **zero** `rate_limited`.
2. THE SYSTEM SHALL hold exactly **two `week_final=true` snapshots**, each pointing at an S3 object that is **readable** — fetch them, do not just read the row.
3. WHEN a completed slot is re-run THE SYSTEM SHALL yield `events_new=0, events_updated=0, events_superseded=0`.
4. Each identity tuple SHALL appear **once** per `week_key`. IF rows are returned here THEN the constraint is wrong, not the data.
5. Live rows per `(country, week_key)` SHALL **equal** the final payload's count for that country. **Read the direction:** *below* means `normalize_title` over-merged — the one corruption no constraint can catch; *above* means supersession did not run. Equality is the point.
6. Sunday events SHALL share a `week_key` with the Monday events after them. Week two is the first time the week-identity bug can surface.
7. Parse warnings on non-blank values SHALL be zero, or each SHALL be traced to a format now covered by a fixture.
8. `exports/<date>/` SHALL hold one CSV and one JSON per export run, each row count equal to `SELECT count(*) FROM events WHERE superseded_at IS NULL` at the time of that run.
9. THE SYSTEM SHALL have logged **two `HEALTH` lines, both `ok`**. IF the health check never ran THEN the other checks told you nothing.

#### Validation Gate

Run all nine as SQL and S3 reads in one sitting, and paste the outputs into the Obsidian vault note `Notes/2026-08-17 - Economic Data Aggregator` — step 0's findings were deleted from the steps document, so the vault is the sole record of decision history.

Clean → MVP done. Actuals (spec §10) are the next iteration and need **no migration**: the four `actual_*` columns already exist. Check the feed before reaching for FMP or FRED — actuals surface as the *following* week's `previous`.

---

## Traceability

| Spec §9 check | Proven by |
|---|---|
| No missed fetches | R11 gate, R14.1 |
| Both sweeps captured | R11.3, R14.2 |
| Idempotent | R8 gate (local), R14.3 |
| Week identity intact | R2, R6.3, R14.4, R14.6 |
| Row count exact | R3, R8.10, R14.5 |
| Parse coverage | R1, R14.7 |
| Export | R12, R14.8 |
| Health | R13, R14.9 |
| No rate limiting | R4.4, R8.4–8.5, R10.1–10.2, R14.1 |

| Global invariant | Enforced by |
|---|---|
| G1 no backfill | R11.3, R14.1–14.2 |
| G2 never drop an event | R1.5, R1.7, R8.13 |
| G3 one week / identity implementation | R2.1, R3.1–3.2, R4 gate, R8.2 |
| G4 week key from payload contents | R2.5, R8.10 |
| G5 soft delete only | R6.5, R7.3, R8.10–8.11 |
| G6 $0/month | R9.1, R13.4 |
| G7 pure modules | R1–R4 gates, R5.2–5.3 |

## Non-Goals

$0/month operating cost is a requirement, not an aspiration. No servers, no VPC, no IaC, no SNS, no UI, no API, no actuals.
