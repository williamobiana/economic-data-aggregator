# FF Harvester MVP — Spec v1.0

**Scope:** Ingest and permanently archive what the ForexFactory JSON feed actually provides — **event schedule, forecast, and previous** — for the 8 major currencies. Actuals are explicitly **out of scope** (next iteration, per spec v0.2 §3 Job B). The schema built here is forward-compatible with that iteration: no migration will be needed, only new columns get filled.

---

## 1. Product statement

A cron-driven script downloads `https://nfs.faireconomy.media/ff_calendar_thisweek.json` four times a day, validates and parses it, and upserts every majors event into Postgres. Because the feed only ever shows the current week, the archive this builds — every event's consensus forecast and previous value, as published, week after week — is data that cannot be obtained retroactively from the feed later. The MVP's entire job is: **never miss a week, never corrupt a value.** Output: a growing `events` table plus a CSV/JSON export.

---

## 2. Source contract (verified against live sample)

Array of objects with exactly these fields:

```json
{"title":"German Factory Orders m\/m","country":"EUR","date":"2026-07-06T02:00:00-04:00","impact":"Low","forecast":"1.1%","previous":"-3.8%"}
```

- `title` — display string; JSON-escaped slashes (`m\/m`) normalize on parse; FF occasionally rewords titles over time.
- `country` — currency code. Keep: `USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF`. Archive **all events** for these currencies (storage is trivial; you can't retro-collect what you filter out). Drop others.
- `date` — ISO 8601 with offset (feed appears to use US Eastern). Convert to UTC at ingestion; store nothing in local time.
- `impact` — `High | Medium | Low | Holiday` (verify the exact holiday/non-economic variants in week one and log unknowns).
- `forecast`, `previous` — display strings; **blank string is common and valid** (means "no consensus published"), especially on Low impact. There is **no `actual` field.**

Operational constraints: ~2 downloads per 5 minutes per IP; rate-limit violations return an HTML page, not JSON; current week only, rolling over around the weekend; unofficial feed — URL and schema can change without notice.

---

## 3. Fetch schedule

| Job | Cron (UTC) | Purpose |
|---|---|---|
| Regular harvest | `0 0,6,12,18 * * *` (4×/day) | Catch new week, forecast revisions, reschedules |
| **Week-final sweep** | `0 23 * * 6` (Sat 23:00) | Archive the closing week's final state before rollover — the single most important fetch of the week |
| Next-week probe (optional) | `30 12 * * *` (1×/day) | If Phase-0 confirms `ff_calendar_nextweek.json` exists: capture forecasts days earlier. Same pipeline, `week_offset=+1` |

Failure policy: on non-JSON response, HTTP error, or empty array → retry once after 10 minutes (never sooner — hard-coded floor of 5 min between any two requests), then give up until the next scheduled run. Alerting is free: a failed GitHub Actions run emails you. Escalate mentally only if the Saturday sweep fails — rerun that one manually.

---

## 4. Data model (3 tables)

**`raw_snapshots`** — every successful fetch, verbatim. The reparse safety net; never skip.

```sql
id          bigserial PRIMARY KEY,
fetched_at  timestamptz NOT NULL,
week_key    date NOT NULL,          -- Sunday (UTC) of the week the payload covers
week_final  boolean DEFAULT false,
payload     jsonb NOT NULL,
byte_size   int
```

**`events`** — the product. One row per real-world event occurrence.

```sql
id             bigserial PRIMARY KEY,
event_key      text UNIQUE NOT NULL,   -- sha1(country|normalized_title|iso_week)
title          text NOT NULL,
country        text NOT NULL,
scheduled_at   timestamptz NOT NULL,   -- UTC
impact         text,
forecast_raw   text,                   -- as published: '1.1%', '-14.5B', ''
previous_raw   text,
forecast_num   numeric,                -- parsed; NULL if blank/unparseable
previous_num   numeric,
unit           text,                   -- 'percent' | 'K' | 'M' | 'B' | 'none'
-- reserved for next iteration (actuals) — created now, always NULL in MVP:
actual_raw     text, actual_num numeric, actual_source text, actual_first_seen_at timestamptz,
first_seen_at  timestamptz NOT NULL,
last_updated_at timestamptz NOT NULL
```

Identity: `event_key = sha1(country + '|' + normalize(title) + '|' + iso_year_week(scheduled_at))`, where `normalize` = trim, collapse whitespace, lowercase, unescape slashes. Week-scoped (not timestamp-scoped) so mid-week reschedules **update** the row instead of duplicating it. Log loudly if one payload produces two events with the same key (possible for repeated intra-week items — acceptable to keep latest in MVP, revisit if the log ever fires).

Upsert semantics: insert if new; on conflict update `scheduled_at, impact, forecast_*, previous_*, unit, last_updated_at` — a changed forecast is a legitimate revision, last write wins (raw snapshots preserve the earlier value if you ever care).

**`fetch_log`** — one row per attempt: `fetched_at, week_offset, http_status, outcome ('ok'|'rate_limited'|'parse_error'|'network'|'empty'), events_seen, events_new, events_updated`. Your gap detector and health record.

---

## 5. Value parser

`parse(raw: string) -> { num: number|null, unit: string|null, flag?: string }`

| Input | num | unit |
|---|---|---|
| `"1.1%"` | 1.1 | percent |
| `"-13.4"` | -13.4 | none |
| `"-14.5B"` | -14 500 000 000 | B |
| `"215K"` / `"7.6M"` | 215 000 / 7 600 000 | K / M |
| `""` | null | null |
| `"<0.1%"` | 0.1 | percent, flag `lt` |
| anything else | null | null, log warning |

Rules: strip whitespace and commas; raw string is stored regardless of outcome; a parse failure is a warning, never a dropped event. **This function gets unit tests before any other code exists** — use your live sample payload as the first fixture.

---

## 6. Stack & layout

GitHub Actions cron → one script → Supabase/Neon free-tier Postgres. No servers, no framework, $0/month.

```
ff-harvester/
├─ src/harvest.ts        # fetch → validate → parse → upsert (~200 lines)
├─ src/parse.ts          # value parser (pure function)
├─ src/export.ts         # events → CSV/JSON dump
├─ test/parse.test.ts    # fixtures incl. your live sample
├─ migrations/001.sql
└─ .github/workflows/harvest.yml   # the three cron schedules
```

Validation order inside `harvest.ts`: HTTP 200 → `Content-Type`/first-char check (HTML = rate-limited, abort) → `JSON.parse` → is non-empty array → each item has `title, country, date` → filter to 8 currencies → snapshot raw → parse values → upsert → write `fetch_log`.

**Export (MVP definition of "export"):** `export.ts` dumps the full `events` table to `events.csv` and `events.json` on demand and via a weekly cron (Sunday, after the sweep), optionally committing the files to the repo — a zero-infrastructure way to have a downloadable, versioned dataset. An HTTP API is next-iteration territory, alongside actuals.

---

## 7. Build order (realistically 2–3 evenings)

1. **Fixture first (30 min):** save your live payload + one hand-mutated edge-case payload (blank forecasts, `<0.1%`, weird title) into `test/`.
2. **Parser + tests (1–2 h).**
3. **Migration + upsert + harvest script (½ day):** run manually a few times, eyeball rows, confirm a rerun produces `events_new=0` (idempotency proof).
4. **Cron + Saturday sweep + failure email (1–2 h):** let it run unattended.
5. **Export script (1 h).**
6. **Week-two check (15 min):** query `fetch_log` for gaps, confirm both weeks fully present, confirm rollover created new rows rather than clobbering old ones. If clean → MVP done.

---

## 8. Risks (MVP-scoped)

| Risk | Mitigation |
|---|---|
| Missed Saturday sweep → final forecast state of a week lost | 4×/day cadence means at most ~6h of late revisions lost; manual rerun on failure email |
| Feed URL/schema change | Strict validation fails loudly; raw snapshots make recovery a reparse, not a data loss |
| Rate-limit ban via retry bug | 5-min floor enforced in code; max 1 retry per run |
| GH Actions cron drift/skipped runs | Harmless at this cadence; `fetch_log` reveals gaps |
| Duplicate rows from title rewording mid-week | Same-week rewording is rare; identity collision log; alias map deferred to next iteration |

## 9. Definition of done

Two consecutive weeks unattended with: every majors event archived with forecast/previous (raw + parsed); both Saturday sweeps captured (`week_final=true` snapshots exist); reruns idempotent; zero parse warnings on standard formats; one clean CSV/JSON export; zero rate-limit incidents in `fetch_log`.

## 10. Explicit next-iteration hooks (not built now, already accommodated)

Actuals resolver (FMP + FRED override per spec v0.2) fills the four reserved `actual_*` columns; `event_map` table added then; surprise becomes computable the day actuals arrive — retroactively for every archived event, since FMP supports historical date-range queries. Nothing in this MVP has to be rebuilt.
