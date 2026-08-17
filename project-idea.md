# FF Harvester MVP — Spec v1.2

**Scope:** archive what the ForexFactory feed provides — **schedule, forecast, previous** — for the 8 majors. Actuals are out of scope; their columns exist now and stay NULL, so that iteration needs no migration.

**Stack:** Python 3.11, Lambda + EventBridge, Supabase Postgres, raw payloads in S3.

Spec says *what* and *why*. `project-idea-steps.md` says *in what order* — it is the build document and the newer of the two. Phase 0 findings live in the Obsidian vault (`Notes/2026-08-17 - Economic Data Aggregator`), not here.

---

## 1. Product statement

A scheduled Lambda downloads `https://nfs.faireconomy.media/ff_calendar_thisweek.json` four times a day, validates it, and upserts every majors event into Postgres. The feed only ever shows the current week, so this archive — every forecast and previous value as published, week after week — cannot be rebuilt retroactively. The job is: **never miss a week, never corrupt a value.** Output: an `events` table plus a CSV/JSON export.

---

## 2. Source contract

```json
{"title":"German Factory Orders m\/m","country":"EUR","date":"2026-07-06T02:00:00-04:00","impact":"Low","forecast":"1.1%","previous":"-3.8%"}
```

- `title` — display string; unescape slashes on parse.
- `country` — keep `USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF`, **all impact levels**; drop the rest (the feed also carries `CNY`). You can't retro-collect what you filter out.
- `date` — ISO 8601 with offset, on **every** row without exception. Convert to UTC at ingestion. **Parse the offset from the string, never hard-code `-04:00`** — it becomes `-05:00` in November and every winter event silently shifts an hour.
- `impact` — `High | Medium | Low | Holiday`; log unknown variants. **There is no Tentative marker** — tentative releases carry a fake minute-precise time and simply move.
- `forecast`, `previous` — display strings. **Blank is common and valid** (80 of 244 rows measured). There is **no `actual` field**.

Constraints: ~2 downloads per 5 min per IP; rate-limit responses are HTML, not JSON; current week only; unofficial feed, can change without notice. `ff_calendar_nextweek.json` **does not exist — it 404s**, so there is no alternative source for a week you failed to capture.

**Measured over three payloads (Aug 2026):** titles **do not drift** within a week (zero changes across 137 titles), **timestamps do** (reschedules and cancellations, future events only), and past rows are **byte-identical across re-fetches** — history is never revised.

---

## 3. Fetch schedule

| Job | Cron (UTC) | Purpose |
|---|---|---|
| Regular harvest | `0 0,6,12,18 * * *` | New week, forecast revisions, reschedules |
| **Week-final sweep** | `0 11 * * 6` (Sat 07:00 ET) | The closing week's final state — the most important fetch of the week |
| Retry slots | each slot + 10 min | Fires unconditionally; no-ops if `fetch_log` shows the paired attempt succeeded |
| Export | Sunday, after the sweep | Full dump to `exports/<date>/` |

**The sweep is deliberately early.** Rollover was bracketed to the 24h after Sat 07:48 ET, never pinned, and a sweep on the wrong side of it loses the closing week permanently. Early costs nothing: no `actual` field, past rows never change, so the week is settled by Saturday morning.

Failure policy: on non-JSON, HTTP error, or empty array → write `fetch_log` and exit. **No in-process sleep-retry**; the paired retry slot handles it. One retry per slot, 5-minute hard floor between any two requests.

---

## 4. Data model

**Week key.** `ff_week_start(d)` = the Sunday beginning the FF week containing `d`, computed in `America/New_York`. Everything week-scoped uses this one function.

- **Sunday-start, not ISO.** ISO weeks begin Monday, which puts Sunday events in the previous week and splits each feed week across two keys. Confirmed against real payloads: `isocalendar()` splits both captured weeks.
- **New York, not UTC.** A Saturday 20:00 ET event is already Sunday in UTC.
- For a payload, derive from contents (`ff_week_start(min(date))`), never from fetch time.

**`raw_snapshots`** — every successful fetch. The reparse safety net; never skip.

```sql
id             bigserial PRIMARY KEY,
fetched_at     timestamptz NOT NULL,
week_key       date NOT NULL,
week_final     boolean DEFAULT false,
s3_key         text NOT NULL,       -- raw/<week_key>/<fetched_at>.json
byte_size      int,
payload_sha256 text NOT NULL
```

**S3 versioning is off**, so `<fetched_at>` in the key is the only thing preventing overwrites. No lifecycle expiry on `raw/` — **never delete raw payloads.**

**`events`** — the product. One row per event occurrence.

```sql
id               bigserial PRIMARY KEY,
event_key        text NOT NULL,          -- sha1 over the four identity parts; convenience, not the authority
week_key         date NOT NULL,
title            text NOT NULL,          -- as published
normalized_title text NOT NULL,          -- identity
country          text NOT NULL,
event_timestamp  timestamptz NOT NULL,   -- UTC
impact           text,
forecast_raw     text,                   -- as published: '1.1%', '-14.5B', ''
previous_raw     text,
forecast_num     numeric,                -- fully expanded; NULL if blank/unparseable
previous_num     numeric,
forecast_flag    text,                   -- 'lt' | 'gt' | NULL, from '<0.1%'
previous_flag    text,
unit             text,                   -- 'percent' | 'count' | 'currency' | 'none'
source_suffix    text,                   -- 'K' | 'M' | 'B' — provenance only
-- reserved for actuals, always NULL in MVP:
actual_raw text, actual_num numeric, actual_source text, actual_first_seen_at timestamptz,
first_seen_at    timestamptz NOT NULL,
last_updated_at  timestamptz NOT NULL,

CONSTRAINT events_identity UNIQUE (country, normalized_title, week_key, event_timestamp)
```

```sql
CREATE INDEX ON events (event_timestamp);
CREATE INDEX ON events (country, event_timestamp);
CREATE INDEX ON events (week_key);
```

**Identity is the named constraint, not a convention** — the upsert's `ON CONFLICT` needs a real target, and §9's duplicate check proves nothing unless the database enforced it from the first insert. Changing it after rows exist means deduplicating first. `normalize` = trim, collapse whitespace, lowercase, unescape slashes.

**`event_timestamp` is in the key, so a rescheduled event lands as a second row** and the old one is never retracted — measured, 79 rows against a closing 74. Accepted for the MVP; supersession is a later pass. Both weaker keys were tested and rejected: dropping `event_timestamp` merges a speaker appearing twice in a week, truncating to the date merges two rows with different `previous`.

Upsert — **only write when something actually changed**, or `last_updated_at` degrades to "last fetched" and `events_updated` reports the whole week every run:

```sql
ON CONFLICT ON CONSTRAINT events_identity DO UPDATE SET ..., last_updated_at = now()
WHERE (events.impact, events.forecast_raw, events.previous_raw)
      IS DISTINCT FROM
      (EXCLUDED.impact, EXCLUDED.forecast_raw, EXCLUDED.previous_raw)
```

`IS DISTINCT FROM`, not `<>`, so NULLs compare correctly.

**`fetch_log`** — one row per attempt: `fetched_at, week_final, http_status, outcome ('ok'|'rate_limited'|'parse_error'|'network'|'empty'), events_seen, events_new, events_updated`. Gap detector and health record.

---

## 5. Value parser

```python
@dataclass(frozen=True)
class Value:
    num: Decimal | None
    unit: str | None       # 'percent' | 'count' | 'currency' | 'none'
    suffix: str | None     # 'K' | 'M' | 'B' — provenance only
    flag: str | None       # 'lt' | 'gt'

def parse(raw: str) -> Value: ...
```

| Input | num | unit | suffix | flag |
|---|---|---|---|---|
| `"1.1%"` | 1.1 | percent | — | — |
| `"-13.4"` | -13.4 | none | — | — |
| `"-14.5B"` | -14 500 000 000 | count | B | — |
| `"215K"` / `"7.6M"` | 215 000 / 7 600 000 | count | K / M | — |
| `""` | null | null | — | — |
| `"<0.1%"` | 0.1 | percent | — | `lt` |
| anything else | null | null | — | — (log warning) |

- `num` is **always fully expanded**; `suffix` is display provenance. Never re-apply it.
- `Decimal`, not `float` — these land in `numeric`.
- Strip whitespace and commas. Raw string is stored regardless; a parse failure is a warning, never a dropped event.
- **No special-casing by `impact`.** Every row carries a real timestamp, Holiday included, so `datetime.fromisoformat` takes the payload as-is. Branching there adds a path with nothing to handle.
- One known shape not in the table: bond-auction `previous` values like `"4.58|2.6"`.

---

## 6. Stack & layout

EventBridge Scheduler → one Lambda → **Supabase** free-tier Postgres, raw payloads in S3. A second Lambda handles export and the health check. No servers, no VPC, no NAT gateway.

- **Supabase pooler string, port `6543`, transaction mode** — not direct `5432`. Lambda opens a fresh connection per invocation and would exhaust the limit. Plain Postgres via `psycopg` v3; the Supabase client libraries are unused.
- **Manual console deployment, no IaC.** Accepted cost: runtime bumps are a manual edit per function.
- **SSM Parameter Store, not Secrets Manager** — the latter is $0.40/secret/month, the only thing that would break the $0 target.

```
ff-harvester/
├─ src/handler.py        # guard → fetch → validate → S3 → parse → upsert
├─ src/parse.py          # value parser
├─ src/week.py           # ff_week_start — the one implementation
├─ src/identity.py       # normalize_title + event_key — the one definition of "same event"
├─ src/guard.py          # already_satisfied(rows, slot) — pure retry logic
├─ src/export.py         # events → CSV/JSON into exports/<date>/
├─ tests/                # test_parse, test_week, test_identity, test_guard
├─ migrations/001.sql
└─ requirements.txt      # psycopg[binary], httpx, tzdata
```

Everything under `src/` except `handler.py` stays pure — no AWS imports, no DB connection — so it runs under plain `pytest` with no mocking. `datetime.fromisoformat` handles the feed's offset natively and `zoneinfo` does the week arithmetic, but **ship `tzdata` to Lambda**, whose image has no system tz database.

**Validation order:** `guard.already_satisfied` → HTTP 200 → `Content-Type`/first-char check (HTML = rate-limited, abort) → `json.loads` → non-empty list → required fields → filter to 8 currencies → **snapshot raw to S3** → parse → upsert → write `fetch_log`. Snapshot before parsing, so a parser crash still leaves the payload recoverable.

**Export:** full dump to `exports/<date>/events.csv` — dated, because versioning is off and a fixed key would destroy the previous dump.

**Alerting:** no SNS yet. Two CloudWatch alarms with no actions attached, plus a `HEALTH ok slots=28 gaps=0` line the export function logs after the `fetch_log` gap query. Wire SNS when remembering to look becomes the failure mode.

---

## 7. Build order

See `project-idea-steps.md`. Roughly 2–3 evenings: pure modules with tests → Supabase → migration → handler locally → AWS prerequisites → deploy and hand-invoke → schedules → export → alarms → week-two check.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| Missed Saturday sweep → week's final state lost | 4×/day caps the loss at ~6h; alarm + manual rerun same evening |
| Sweep on the wrong side of rollover | Sweep set early (Sat 07:00 ET) rather than at the untested boundary |
| Week boundary mis-resolved → Sunday events duplicated | One `ff_week_start`, Sunday-start and NY-based; §9 checks it directly |
| DST shift misreads every timestamp by an hour | Offset parsed from the feed, never assumed. **No real payload exercises `-05:00` yet** — hand-build that fixture |
| Reschedule leaves a stale row | Known and accepted; §9 reads the *direction* of the row-count mismatch |
| Feed URL/schema change | Strict validation fails loudly; raw snapshots make recovery a reparse, not a loss |
| Rate-limit ban | 5-min floor; separate retry slot, no in-process loop; `reserved_concurrency=1`, Lambda retries 0 |
| Scheduler drift | Harmless at this cadence; `fetch_log` reveals gaps |

---

## 9. Definition of done

Two consecutive weeks unattended, every row a query you can run:

| Check | Passes when |
|---|---|
| No missed fetches | An `ok` row per scheduled slot; no gap > 8h |
| Both sweeps captured | 2 rows with `week_final=true`, each with a readable S3 object |
| Idempotent | Re-running a completed slot yields `events_new=0, events_updated=0` |
| Week identity intact | One row per identity tuple per `week_key`; Sunday events share a `week_key` with the Monday events after them |
| No over-merging | Rows per `(country, week_key)` are **not fewer** than the raw payload's count |
| Parse coverage | Zero warnings on non-blank values, or each traced to a format now covered by a fixture |
| Export | One CSV + one JSON in `exports/<date>/`, row count matching `SELECT count(*) FROM events` |
| Health | Two `HEALTH` lines, both `ok` — if it never ran, the other checks told you nothing |
| No rate limiting | Zero `rate_limited` outcomes |

On the over-merging row, **read the direction**: above the payload count is stale reschedule rows, which are expected; below it is `normalize_title` merging two distinct events, which no constraint can catch and is the real defect.

---

## 10. Next-iteration hooks

Actuals resolver fills the reserved `actual_*` columns; `event_map` added then. Surprise becomes computable retroactively for every archived event. Nothing here gets rebuilt.

**Check the feed before reaching for FMP or FRED.** Actuals surface as the *following* week's `previous` — Crude Oil `2.5M` → `17.4M`, Claims `199K` → `209K`, ADP `11.0K` → `8.3K`. Weekly series resolve in a week, monthly in a month. Reading `previous` forward across week boundaries may be the whole feature.
