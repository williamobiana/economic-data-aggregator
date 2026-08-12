# FF Harvester MVP — Spec v1.1

**Scope:** archive what the ForexFactory feed provides — **schedule, forecast, previous** — for the 8 majors. Actuals are out of scope; their columns exist now and stay NULL, so that iteration needs no migration.

**Stack:** Python 3.11, Lambda + EventBridge, Neon/Supabase Postgres, raw payloads in S3.

---

## 1. Product statement

A scheduled Lambda downloads `https://nfs.faireconomy.media/ff_calendar_thisweek.json` four times a day, validates it, and upserts every majors event into Postgres. The feed only ever shows the current week, so this archive — every forecast and previous value as published, week after week — cannot be rebuilt retroactively. The job is: **never miss a week, never corrupt a value.** Output: an `events` table plus a CSV/JSON export.

---

## 2. Source contract

```json
{"title":"German Factory Orders m\/m","country":"EUR","date":"2026-07-06T02:00:00-04:00","impact":"Low","forecast":"1.1%","previous":"-3.8%"}
```

- `title` — display string; unescape slashes on parse. FF rewords titles over time.
- `country` — keep `USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF`, **all impact levels**; drop the rest. You can't retro-collect what you filter out.
- `date` — ISO 8601 with offset. Convert to UTC at ingestion. **Parse the offset from the string, never hard-code `-04:00`** — it becomes `-05:00` in November and every winter event silently shifts an hour.
- `impact` — `High | Medium | Low | Holiday`; log unknown variants.
- `forecast`, `previous` — display strings. **Blank is common and valid** (no consensus published). There is **no `actual` field**.
- All-Day / Tentative rows likely carry a placeholder midnight; treat `scheduled_at` as approximate for those (Phase 0 confirms).

Constraints: ~2 downloads per 5 min per IP; rate-limit responses are HTML, not JSON; current week only; unofficial feed, can change without notice.

---

## 3. Fetch schedule

| Job | Cron (UTC) | Purpose |
|---|---|---|
| Regular harvest | `0 0,6,12,18 * * *` | New week, forecast revisions, reschedules |
| **Week-final sweep** | `0 23 * * 6` | The closing week's final state — the most important fetch of the week |
| Retry slots | each slot + 10 min | Fires unconditionally; no-ops if `fetch_log` shows the paired attempt succeeded |
| Next-week probe (optional) | `30 12 * * *` | If Phase 0 confirms `ff_calendar_nextweek.json`; same pipeline, `week_offset=+1` |

Sweep time is provisional — **Phase 0 measures the actual rollover moment** and the schedule moves if 23:00 UTC Saturday lands on the wrong side of it.

Failure policy: on non-JSON, HTTP error, or empty array → write `fetch_log` and exit. **No in-process sleep-retry**; the paired retry slot handles it. One retry per slot, 5-minute hard floor between any two requests.

---

## 4. Data model

**Week key.** `ff_week_start(d)` = the Sunday beginning the FF week containing `d`, computed in `America/New_York`. Everything week-scoped uses this one function.

- **Sunday-start, not ISO.** ISO weeks begin Monday, which puts Sunday events in the previous week and splits each feed week across two keys.
- **New York, not UTC.** A Saturday 20:00 ET event is already Sunday in UTC.
- For a payload, derive from contents (`ff_week_start(min(date))`), never from fetch time — they disagree during rollover and on backfills.

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

Versioning on, no lifecycle expiry — **never delete raw payloads.**

**`events`** — the product. One row per event occurrence.

```sql
id              bigserial PRIMARY KEY,
event_key       text UNIQUE NOT NULL,   -- sha1(country|normalized_title|week_key)
week_key        date NOT NULL,
title           text NOT NULL,
country         text NOT NULL,
scheduled_at    timestamptz NOT NULL,   -- UTC
impact          text,
forecast_raw    text,                   -- as published: '1.1%', '-14.5B', ''
previous_raw    text,
forecast_num    numeric,                -- fully expanded; NULL if blank/unparseable
previous_num    numeric,
forecast_flag   text,                   -- 'lt' | 'gt' | NULL, from '<0.1%'
previous_flag   text,
unit            text,                   -- 'percent' | 'count' | 'currency' | 'none'
source_suffix   text,                   -- 'K' | 'M' | 'B' — provenance only
-- reserved for actuals, always NULL in MVP:
actual_raw text, actual_num numeric, actual_source text, actual_first_seen_at timestamptz,
first_seen_at   timestamptz NOT NULL,
last_updated_at timestamptz NOT NULL
```

```sql
CREATE INDEX ON events (scheduled_at);
CREATE INDEX ON events (country, scheduled_at);
CREATE INDEX ON events (week_key);
```

Identity: `normalize` = trim, collapse whitespace, lowercase, unescape slashes. Week-scoped rather than timestamp-scoped, so mid-week reschedules update the row instead of duplicating it. Log loudly if one payload yields two events with the same key.

Upsert — **only write when something actually changed**, or `last_updated_at` degrades to "last fetched" and `events_updated` reports the whole week every run:

```sql
ON CONFLICT (event_key) DO UPDATE SET ..., last_updated_at = now()
WHERE (events.scheduled_at, events.impact, events.forecast_raw, events.previous_raw)
      IS DISTINCT FROM
      (EXCLUDED.scheduled_at, EXCLUDED.impact, EXCLUDED.forecast_raw, EXCLUDED.previous_raw)
```

`IS DISTINCT FROM`, not `<>`, so NULLs compare correctly.

**`fetch_log`** — one row per attempt: `fetched_at, week_offset, http_status, outcome ('ok'|'rate_limited'|'parse_error'|'network'|'empty'), events_seen, events_new, events_updated`. Gap detector and health record.

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
- Strip whitespace and commas. Raw string is stored regardless of outcome; a parse failure is a warning, never a dropped event.
- **Tested before any other code exists.** Live sample is the first fixture; add one per distinct shape Phase 0 finds.

---

## 6. Stack & layout

EventBridge Scheduler → one Lambda → Neon/Supabase free-tier Postgres, with raw payloads in S3. No servers, no VPC, no NAT gateway. See `project-idea-steps.md` for the deploy sequence.

Managed Postgres over Aurora Serverless v2: Aurora bills storage even at 0 ACU and adds resume latency, and the Data API mainly solves a VPC problem Neon doesn't have. **SSM Parameter Store, not Secrets Manager** — the latter is $0.40/secret/month, the only thing that would break the $0 target.

```
ff-harvester/
├─ src/handler.py        # fetch → validate → S3 → parse → upsert (~200 lines)
├─ src/parse.py          # value parser (pure, no AWS imports)
├─ src/week.py           # ff_week_start — the one implementation
├─ src/export.py         # events → CSV/JSON into s3://…/exports/
├─ tests/test_parse.py
├─ tests/test_week.py    # rollover + Sunday-event + DST cases
├─ migrations/001.sql
├─ requirements.txt      # psycopg[binary], httpx, tzdata
└─ template.yaml         # SAM: function, role, schedules, alarms
```

`parse.py` and `week.py` stay pure so they run under plain `pytest` with no mocking. Two 3.11 stdlib notes: `datetime.fromisoformat` handles the feed's offset natively, and `zoneinfo.ZoneInfo("America/New_York")` does the week arithmetic — but **ship `tzdata` to Lambda**, whose image has no system tz database.

Validation order: HTTP 200 → `Content-Type`/first-char check (HTML = rate-limited, abort) → `json.loads` → non-empty list → required fields → filter to 8 currencies → **snapshot raw to S3** → parse → upsert → write `fetch_log`. Snapshot before parsing, so a parser crash still leaves the payload recoverable.

**Export:** `export.py` dumps `events` to CSV and JSON into `exports/` on demand and weekly (Sunday, after the sweep). S3 versioning gives dataset history free. HTTP API is next-iteration.

---

## 7. Build order (2–3 evenings)

0. **Phase 0 — observe before building.** A few manual `curl`s over a weekend, honouring the 5-min floor: does `ff_calendar_nextweek.json` exist; **when exactly does the week roll over** (fetch Sat evening + Sun morning, diff); what does `date` carry for All-Day rows; collect every distinct value shape as fixtures.
1. **Fixtures (30 min)** — live payload, Phase-0 captures, one hand-mutated edge case.
2. **Parser + week key + tests (1–2 h).** `week.py`'s tests matter more: a Sunday event, a Saturday 20:00 ET event, both sides of the November DST change. A wrong parse logs; a wrong week key silently duplicates.
3. **Migration + upsert + handler, locally (½ day).** Run against the real DB before deploying. A rerun must yield `events_new=0` **and `events_updated=0`**.
4. **Deploy + schedules + sweep alarm (1–2 h).**
5. **Export (1 h).**
6. **Week-two check (15 min)** — §9.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| Missed Saturday sweep → week's final forecast state lost | 4×/day caps the loss at ~6h of revisions; alarm + manual rerun same evening |
| Sweep scheduled on the wrong side of rollover | Phase 0 measures it before the schedule is committed |
| Week boundary mis-resolved → Sunday events duplicated | One `ff_week_start`, Sunday-start and NY-based, tested at rollover and DST edges; §9 checks it directly |
| DST shift misreads every timestamp by an hour | Offset parsed from the feed, never assumed |
| Feed URL/schema change | Strict validation fails loudly; raw snapshots make recovery a reparse, not a loss |
| Rate-limit ban | 5-min floor; separate retry slot, no in-process loop; `reserved_concurrency=1`, Lambda retries 0 |
| Scheduler drift | Harmless at this cadence; `fetch_log` reveals gaps |
| Title rewording mid-week duplicates a row | Rare; collision log; alias map deferred |

---

## 9. Definition of done

Two consecutive weeks unattended, every row a query you can run:

| Check | Passes when |
|---|---|
| No missed fetches | An `ok` row per scheduled slot; no gap > 8h |
| Both sweeps captured | 2 rows with `week_final=true`, each with a readable S3 object |
| Idempotent | Re-running a completed slot yields `events_new=0, events_updated=0` |
| Week identity intact | One row per `(country, normalized_title)` per `week_key`; Sunday events share a `week_key` with the Monday events after them |
| Parse coverage | Zero warnings on non-blank values, or each traced to a format now covered by a fixture |
| Export | One CSV + one JSON in `exports/`, row count matching `SELECT count(*) FROM events` |
| No rate limiting | Zero `rate_limited` outcomes |

---

## 10. Next-iteration hooks

Actuals resolver (FMP + FRED override) fills the reserved `actual_*` columns; `event_map` added then. Surprise becomes computable the day actuals arrive — retroactively for every archived event, since FMP supports historical date ranges. Nothing here gets rebuilt.
