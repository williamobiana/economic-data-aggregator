# FF Harvester MVP — Spec v1.1

**Scope:** Ingest and permanently archive what the ForexFactory JSON feed actually provides — **event schedule, forecast, and previous** — for the 8 major currencies. Actuals are explicitly **out of scope** (next iteration, per spec v0.2 §3 Job B). The schema built here is forward-compatible with that iteration: no migration will be needed, only new columns get filled.

> **v1.1 review changes.** (1) **Stack decided — AWS.** v1.0 §6 specified GitHub Actions + Postgres-held payloads, while `project-idea-steps.md` specified AWS Lambda + S3. Settled on **Python 3.11 + AWS compute + managed Postgres (Neon/Supabase) + S3 raw payloads**. Closed decision, not a placeholder. (2) **Week identity bug fixed** — `event_key` used ISO weeks (Monday-start) while `week_key` used Sunday-start; both now use one canonical `ff_week_start`. See §4. (3) Parser `flag` now has somewhere to land. (4) Phase 0 added — v1.0 §3 referenced it but §7 never defined it.

---

## 1. Product statement

A scheduled Lambda downloads `https://nfs.faireconomy.media/ff_calendar_thisweek.json` four times a day, validates and parses it, and upserts every majors event into Postgres. Because the feed only ever shows the current week, the archive this builds — every event's consensus forecast and previous value, as published, week after week — is data that cannot be obtained retroactively from the feed later. The MVP's entire job is: **never miss a week, never corrupt a value.** Output: a growing `events` table plus a CSV/JSON export.

---

## 2. Source contract (verified against live sample)

Array of objects with exactly these fields:

```json
{"title":"German Factory Orders m\/m","country":"EUR","date":"2026-07-06T02:00:00-04:00","impact":"Low","forecast":"1.1%","previous":"-3.8%"}
```

- `title` — display string; JSON-escaped slashes (`m\/m`) normalize on parse; FF occasionally rewords titles over time.
- `country` — currency code. Keep: `USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF`. Archive **all events** for these currencies (storage is trivial; you can't retro-collect what you filter out). Drop others.
- `date` — ISO 8601 with offset (feed appears to use US Eastern). Convert to UTC at ingestion; store nothing in local time. **Parse the offset from the string; never hard-code `-04:00`** — the sample is July (EDT) and the feed will emit `-05:00` from early November. A hard-coded offset silently shifts every winter event by an hour.
- `impact` — `High | Medium | Low | Holiday` (verify the exact holiday/non-economic variants in week one and log unknowns).
- **All-day / tentative events.** FF renders some rows as "All Day" or "Tentative" (bank holidays, summits, unscheduled releases). Verify in Phase 0 what `date` carries for these — likely local midnight, which is a placeholder, not a real release time. Store them, but treat `scheduled_at` as approximate for `impact='Holiday'`; do not let a midnight placeholder push an event across the week boundary (§4 resolves the week from the feed's own week, not from the timestamp).
- `forecast`, `previous` — display strings; **blank string is common and valid** (means "no consensus published"), especially on Low impact. There is **no `actual` field.**

Operational constraints: ~2 downloads per 5 minutes per IP; rate-limit violations return an HTML page, not JSON; current week only, rolling over around the weekend; unofficial feed — URL and schema can change without notice.

---

## 3. Fetch schedule

| Job | Cron (UTC) | Purpose |
|---|---|---|
| Regular harvest | `0 0,6,12,18 * * *` (4×/day) | Catch new week, forecast revisions, reschedules |
| **Week-final sweep** | `0 23 * * 6` (Sat 23:00) | Archive the closing week's final state before rollover — the single most important fetch of the week |
| Retry slots | each main slot + 10 min | Fires unconditionally; handler exits immediately if `fetch_log` shows the paired attempt already succeeded (see failure policy) |
| Next-week probe (optional) | `30 12 * * *` (1×/day) | If Phase 0 confirms `ff_calendar_nextweek.json` exists: capture forecasts days earlier. Same pipeline, `week_offset=+1` |

**Sweep timing needs a Phase-0 answer.** Sat 23:00 UTC is Sat 19:00 EDT / 18:00 EST. That is before rollover *if* the feed turns over at or after midnight ET Saturday, but the exact rollover moment is unverified and it is the one thing this schedule depends on. Confirm it in Phase 0 (§7.0) and move the sweep if it lands wrong — losing the closing week's final forecast state is the one unrecoverable failure in this design.

Failure policy: on non-JSON response, HTTP error, or empty array → **do not sleep-and-retry inside the run.** Exit, write `fetch_log`, and let the paired retry slot 10 minutes later pick it up; that retry checks `fetch_log` first and no-ops if the main attempt succeeded. This keeps the 5-minute inter-request floor without paying for a process to sit idle, and it survives the run being killed. Max one retry per slot. Escalate only if the Saturday sweep fails — rerun that one manually the same evening (§7.4 wires the alarm).

---

## 4. Data model (3 tables)

**Canonical week key (used by both tables — read this first).**

```
ff_week_start(d) = the date of the Sunday that begins the ForexFactory week
                   containing d, computed in America/New_York
```

Everything week-scoped in this system uses that one function. Two rules make it safe:

- **Sunday-start, not ISO.** ISO-8601 weeks begin Monday, so `iso_week()` puts a Sunday event in the week that started the *previous* Monday — splitting one feed week across two keys and duplicating rows at every rollover. Do not use ISO weeks anywhere.
- **New York, not UTC.** A Saturday 20:00 ET event is already Sunday in UTC; resolving the week from a UTC timestamp walks it into the following week.

For a payload, derive the week from its contents — `ff_week_start(min(date))` — never from the wall-clock time of the fetch, which disagrees with the payload during the rollover window and on any manual backfill.

**`raw_snapshots`** — every successful fetch, verbatim. The reparse safety net; never skip.

```sql
id             bigserial PRIMARY KEY,
fetched_at     timestamptz NOT NULL,
week_key       date NOT NULL,       -- ff_week_start(min(date)) of the payload
week_final     boolean DEFAULT false,
s3_key         text NOT NULL,       -- raw/<week_key>/<fetched_at>.json; payload lives in S3
byte_size      int,
payload_sha256 text NOT NULL        -- dedupe + integrity check on reparse
```

The payload itself goes to S3 rather than a `jsonb` column (`project-idea-steps.md` step 2): it keeps the free-tier Postgres row count small, and raw archival is exactly what object storage is for. Versioning on, no lifecycle expiry — **never delete raw payloads.**

**`events`** — the product. One row per real-world event occurrence.

```sql
id             bigserial PRIMARY KEY,
event_key      text UNIQUE NOT NULL,   -- sha1(country|normalized_title|week_key)
week_key       date NOT NULL,          -- ff_week_start; stored, not just hashed
title          text NOT NULL,
country        text NOT NULL,
scheduled_at   timestamptz NOT NULL,   -- UTC
impact         text,
forecast_raw   text,                   -- as published: '1.1%', '-14.5B', ''
previous_raw   text,
forecast_num   numeric,                -- fully expanded; NULL if blank/unparseable
previous_num   numeric,
forecast_flag  text,                   -- 'lt' | 'gt' | NULL — qualifier from '<0.1%'
previous_flag  text,
unit           text,                   -- dimension: 'percent' | 'count' | 'currency' | 'none'
source_suffix  text,                   -- informational only: 'K' | 'M' | 'B' | NULL
-- reserved for next iteration (actuals) — created now, always NULL in MVP:
actual_raw     text, actual_num numeric, actual_source text, actual_first_seen_at timestamptz,
first_seen_at  timestamptz NOT NULL,
last_updated_at timestamptz NOT NULL
```

```sql
CREATE INDEX ON events (scheduled_at);
CREATE INDEX ON events (country, scheduled_at);
CREATE INDEX ON events (week_key);
```

Identity: `event_key = sha1(country + '|' + normalize(title) + '|' + ff_week_start(scheduled_at_in_NY))`, where `normalize` = trim, collapse whitespace, lowercase, unescape slashes. Week-scoped (not timestamp-scoped) so mid-week reschedules **update** the row instead of duplicating it. `week_key` is also stored as a column, not just folded into the hash — otherwise "show me everything from week X" requires recomputing the hash for every candidate row. Log loudly if one payload produces two events with the same key (possible for repeated intra-week items — acceptable to keep latest in MVP, revisit if the log ever fires).

Upsert semantics: insert if new; on conflict update `scheduled_at, impact, forecast_*, previous_*, unit, last_updated_at` — a changed forecast is a legitimate revision, last write wins (raw snapshots preserve the earlier value if you ever care).

**Only write when something actually changed:**

```sql
ON CONFLICT (event_key) DO UPDATE SET ... , last_updated_at = now()
WHERE (events.scheduled_at, events.impact, events.forecast_raw, events.previous_raw)
      IS DISTINCT FROM
      (EXCLUDED.scheduled_at, EXCLUDED.impact, EXCLUDED.forecast_raw, EXCLUDED.previous_raw)
```

Without the `WHERE`, every one of the ~4 daily runs rewrites every row of the week: `last_updated_at` stops meaning "when this value last changed" and starts meaning "when we last fetched", and `fetch_log.events_updated` reports the full week's event count every run instead of the revision count. Both are load-bearing — §7.3's idempotency proof and §9's definition of done are stated in terms of them. Use `IS DISTINCT FROM`, not `<>`, so NULLs compare correctly.

**`fetch_log`** — one row per attempt: `fetched_at, week_offset, http_status, outcome ('ok'|'rate_limited'|'parse_error'|'network'|'empty'), events_seen, events_new, events_updated`. Your gap detector and health record.

---

## 5. Value parser

```python
@dataclass(frozen=True)
class Value:
    num: Decimal | None
    unit: str | None          # 'percent' | 'count' | 'currency' | 'none'
    suffix: str | None        # 'K' | 'M' | 'B' — provenance only
    flag: str | None          # 'lt' | 'gt'

def parse(raw: str) -> Value: ...
```

`Decimal`, not `float` — these land in a `numeric` column and binary floating point will turn `1.1` into `1.1000000000000001` somewhere between the parser and Postgres. A frozen dataclass rather than a dict so a mistyped field name fails loudly instead of writing `None`.

| Input | num | unit | suffix | flag |
|---|---|---|---|---|
| `"1.1%"` | 1.1 | percent | — | — |
| `"-13.4"` | -13.4 | none | — | — |
| `"-14.5B"` | -14 500 000 000 | count | B | — |
| `"215K"` / `"7.6M"` | 215 000 / 7 600 000 | count | K / M | — |
| `""` | null | null | — | — |
| `"<0.1%"` | 0.1 | percent | — | `lt` |
| anything else | null | null | — | — (log warning) |

Two v1.0 corrections here:

- **`num` is always fully expanded, and `unit` is a dimension, not a scale.** v1.0 set `unit` to `'B'` while `num` held the already-multiplied value — a consumer reading `(-1.45e10, 'B')` can quite reasonably multiply by a billion a second time. The scale suffix is preserved separately in `source_suffix` as display provenance only. **Never re-apply it.**
- **`flag` now persists.** v1.0's parser returned it and the schema had nowhere to put it, so `<0.1%` was silently indistinguishable from `0.1%`. Columns `forecast_flag` / `previous_flag` added in §4.

Rules: strip whitespace and commas; raw string is stored regardless of outcome; a parse failure is a warning, never a dropped event. **This function gets unit tests before any other code exists** — use your live sample payload as the first fixture. Feed the Phase-0 corpus (§7.0) through it and add a fixture for every distinct shape it finds; the `anything else` row is where unknown formats go to be discovered, so treat a warning in week one as a spec question, not noise.

---

## 6. Stack & layout

EventBridge Scheduler → one Lambda → Neon/Supabase free-tier Postgres, with raw payloads in S3. No servers, no VPC, no NAT gateway. See `project-idea-steps.md` for the deploy sequence.

Managed Postgres over Aurora Serverless v2 deliberately: Aurora at 0 ACU still bills storage and adds resume latency, and the Data API exists mainly to dodge the VPC problem that Neon doesn't have. **Use SSM Parameter Store, not Secrets Manager, for the connection string** — Secrets Manager is $0.40/secret/month, which is the only thing in this design that would break the $0 claim. Standard Parameter Store is free.

```
ff-harvester/
├─ src/handler.py        # fetch → validate → S3 → parse → upsert (~200 lines)
├─ src/parse.py          # value parser (pure function, no AWS imports)
├─ src/week.py           # ff_week_start — the one week-key implementation
├─ src/export.py         # events → CSV/JSON into s3://…/exports/
├─ tests/test_parse.py   # fixtures incl. your live sample
├─ tests/test_week.py    # rollover + Sunday-event + DST cases
├─ migrations/001.sql
├─ requirements.txt      # psycopg[binary], httpx, tzdata
└─ template.yaml         # SAM: function, role, schedules, alarms
```

`parse.py` and `week.py` stay pure and AWS-free so they run under plain `pytest` with no mocking — that is what makes §7's "parser first, before any cloud" order possible.

Two 3.11 stdlib details worth stating, because both are places where a plausible-looking implementation is wrong:

- **`datetime.fromisoformat` handles the feed's offset natively** on 3.11 (it was much pickier on older versions). Parse the offset out of the string; never assume `-04:00`, or every event from November onward shifts by an hour.
- **`zoneinfo.ZoneInfo("America/New_York")`** does the week arithmetic — no `pytz`, no `dateutil`. Ship the `tzdata` package to Lambda, though: the runtime image carries no system tz database and `ZoneInfo` will raise at import without it.

Validation order inside the handler: HTTP 200 → `Content-Type`/first-char check (HTML = rate-limited, abort) → `json.loads` → is non-empty list → each item has `title, country, date` → filter to 8 currencies → snapshot raw to S3 → parse values → upsert → write `fetch_log`. Snapshot **before** parsing: if parsing throws, the payload is already safe and the run is a reparse away from recovery.

**Export (MVP definition of "export"):** `export.py` dumps the full `events` table to `events.csv` and `events.json` on demand and via a weekly schedule (Sunday, after the sweep), writing to the `exports/` prefix of the same bucket — a zero-infrastructure way to have a downloadable, versioned dataset (S3 versioning gives you the history for free). An HTTP API is next-iteration territory, alongside actuals.

---

## 7. Build order (realistically 2–3 evenings)

0. **Phase 0 — observe before building (2 fetches, ~20 min of attention, spread over a weekend).** Everything downstream is guessing until these are answered. Pull the feed by hand a few times, `curl` only, honouring the 5-minute floor:
   - Does `ff_calendar_nextweek.json` exist? (decides the §3 probe)
   - **When exactly does the week roll over?** One fetch Saturday evening, one Sunday morning; diff them. This sets the sweep time and is the highest-value 5 minutes in the project.
   - What does `date` carry for All-Day / Tentative / Holiday rows?
   - Collect every distinct `forecast`/`previous` string shape you see → these become §5 fixtures.
1. **Fixture first (30 min):** save your live payload + the Phase-0 captures + one hand-mutated edge-case payload (blank forecasts, `<0.1%`, weird title) into `test/`.
2. **Parser + week-key + tests (1–2 h).** `week.py` gets tests too, and they matter more than the parser's: a Sunday event, a Saturday 20:00 ET event, and one date on each side of the November DST change. Wrong week keys corrupt identity silently; wrong parses just log.
3. **Migration + upsert + handler, run locally (½ day):** invoke as a plain script against the real DB before any deploy. Eyeball rows, then confirm a rerun produces `events_new=0` **and `events_updated=0`** — the second half is the actual idempotency proof (§4).
4. **Deploy + schedules + sweep alarm (1–2 h):** SAM template, then let it run unattended.
5. **Export script (1 h).**
6. **Week-two check (15 min):** query `fetch_log` for gaps, confirm both weeks fully present, confirm rollover created new rows rather than clobbering old ones. **Specifically check that Sunday events share a `week_key` with the Monday events that follow them** — that is the v1.0 bug's signature, and week two is when it would first show. If clean → MVP done.

---

## 8. Risks (MVP-scoped)

| Risk | Mitigation |
|---|---|
| Missed Saturday sweep → final forecast state of a week lost | 4×/day cadence means at most ~6h of late revisions lost; manual rerun on failure email |
| Feed URL/schema change | Strict validation fails loudly; raw snapshots make recovery a reparse, not a data loss |
| Rate-limit ban via retry bug | 5-min floor enforced in code; separate retry slot, never an in-process loop; `reserved_concurrency=1` and Lambda retries set to 0 so two runs can't overlap |
| Scheduler drift/skipped runs | Harmless at this cadence; `fetch_log` reveals gaps |
| Duplicate rows from title rewording mid-week | Same-week rewording is rare; identity collision log; alias map deferred to next iteration |
| **Week boundary mis-resolved** → same event split across two `week_key`s, silently duplicating every Sunday event | One `ff_week_start` implementation (§4), Sunday-start and NY-based, unit-tested at the rollover and DST edges; §9 checks the invariant directly |
| DST shift (Nov / Mar) misreads every timestamp by an hour | Offset parsed from the feed string, never assumed; §7.2 tests both sides of the change |
| Sweep scheduled on the wrong side of rollover → the one unrecoverable loss | Phase 0 measures the actual rollover time before the schedule is committed (§7.0) |

## 9. Definition of done

Two consecutive weeks unattended, every item a query you can actually run:

| Check | Passes when |
|---|---|
| No missed fetches | `fetch_log` has an `ok` row for every scheduled slot; no gap > 8h |
| Both sweeps captured | 2 rows with `week_final=true`, each with a readable S3 object |
| Reruns idempotent | A manual re-run of a completed slot yields `events_new=0, events_updated=0` |
| Week identity intact | Each `(country, normalized_title)` has exactly one row per `week_key`; Sunday events carry the same `week_key` as the Monday events after them |
| Parse coverage | Zero warnings on non-blank values across both weeks — or every warning traced to a format now covered by a fixture |
| Export | One CSV + one JSON in `exports/`, row count matching `SELECT count(*) FROM events` |
| No rate limiting | Zero `rate_limited` outcomes in `fetch_log` |

"Zero parse warnings on standard formats" from v1.0 was not checkable — *standard* was never defined, so any warning could be argued away. The bar above is either met or not.

## 10. Explicit next-iteration hooks (not built now, already accommodated)

Actuals resolver (FMP + FRED override per spec v0.2) fills the four reserved `actual_*` columns; `event_map` table added then; surprise becomes computable the day actuals arrive — retroactively for every archived event, since FMP supports historical date-range queries. Nothing in this MVP has to be rebuilt.
