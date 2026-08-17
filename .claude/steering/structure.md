# Structure

```
ff-harvester/
├─ src/handler.py        # harvester: guard -> fetch -> validate -> S3 -> parse -> upsert -> supersede -> log
├─ src/parse.py          # value parser -> frozen Value dataclass
├─ src/week.py           # ff_week_start - the one implementation
├─ src/identity.py       # normalize_title + event_key - the one definition of "same event"
├─ src/guard.py          # already_satisfied(rows, slot) - pure retry logic
├─ src/export.py         # live events -> exports/<date>/, plus the HEALTH line
├─ tests/                # test_parse, test_week, test_identity, test_guard
├─ migrations/001.sql    # raw, hand-run once. No Alembic.
├─ json-files/           # captured payloads, the fixture source
└─ requirements.txt
```

**Everything under `src/` except `handler.py` and `export.py` stays pure** — no AWS imports, no DB connection, stdlib only. That is what makes the tests mock-free.

## Module boundaries

- `week.py`, `identity.py` — single implementations. Nothing else derives a week key or an event key.
- `guard.py` — takes `fetch_log` rows already in hand and returns an answer. The handler fetches the rows and acts on it; it never re-derives the logic, and neither does the EventBridge schedule (retry slots are configuration only).
- `parse.py` — returns `Value(num, unit, suffix, flag)`, frozen dataclass, not a dict. `num` is always fully expanded (`-14.5B` -> -14500000000); `suffix` is provenance, never re-applied. Blank input is valid and common. A parse failure is a warning and the raw string is stored regardless — **never a dropped event**.
- `handler.py` — orchestration only. Validation order is fixed: guard -> HTTP 200 -> HTML check -> `json.loads` -> non-empty list -> required fields -> currency filter -> S3 -> parse -> upsert -> supersede -> `fetch_log`.

## Tables

`raw_snapshots` (one row per successful fetch, points at the S3 object) · `events` (the product, one row per event occurrence) · `fetch_log` (one row per attempt — the gap detector).

Store both the raw string and the parsed value: `forecast_raw` / `forecast_num` / `forecast_flag`, same for `previous`.

## Testing emphasis

Test `week.py` and `identity.py` harder than the parser — a bad parse logs a warning, a bad week or identity key silently duplicates rows.

- `week.py`: a Sunday event, a Sat 20:00 ET event, both sides of the November DST change.
- `identity.py`: assert the negative — the same `(country, date)` across fetches yields the same title (titles were measured and do not drift) — and that distinct events stay distinct: `FOMC Member Hammack Speaks` twice in one week, and the ADP 08:14/08:15 pair.

Fixtures come from `json-files/`. The DST case has no real payload behind it — hand-build it.

## Documents

- `project-idea.md` — the spec (v1.2): what and why.
- `project-idea-steps.md` — the build order: in what order. **Newer of the two; wins on conflicts.**
- Obsidian vault (`/home/user/workspace/lab/second-brain`) — Phase 0 findings and decision history. Step 0 was deleted from the steps document, so the vault is their **sole** record: `Notes/2026-08-17 - Economic Data Aggregator`.

## Build order

Pure modules + tests -> Supabase -> migration -> handler locally -> AWS prerequisites -> deploy and hand-invoke -> EventBridge schedules -> export function -> alarms -> week-two check. Steps 1-4 are AWS-free.
