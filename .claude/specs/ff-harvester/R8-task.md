# R8 - Harvester handler, run locally · Tasks

The big one. Runs end-to-end from a laptop against the real bucket and the real database, before any Lambda exists.

- [ ] 1. Create `src/handler.py` as orchestration only
  - Week arithmetic, identity, parsing and guard logic stay in their own modules
  - `week_final` comes from the invocation event payload, so one function serves every schedule
  - _Requirements: 8.2, 8.12_

- [ ] 2. Implement the validation chain, in this exact order
  - `guard.already_satisfied` -> HTTP 200 -> HTML check -> `json.loads` -> non-empty list -> required fields -> currency filter
  - `httpx` at 10s connect / 20s read
  - HTML body means rate-limited: abort **before** `json.loads`
  - Filter to `USD EUR GBP JPY AUD NZD CAD CHF CNY`, all impact levels including Holiday
  - _Requirements: 8.1, 8.3, 8.4, 8.6_

- [ ] 3. Snapshot raw to S3 before parsing
  - `raw/<week_key>/<fetched_at>.json`, then record `s3_key`, `byte_size`, `payload_sha256` in `raw_snapshots`
  - A parser crash must still leave the payload recoverable
  - _Requirements: 8.7_

- [ ] 4. Parse and derive
  - `datetime.fromisoformat`, convert to UTC at ingestion, never construct an offset
  - `week_key = ff_week_start(min(date))` from the payload's own contents
  - Log unknown `impact` variants without rejecting the row
  - _Requirements: 8.8, 8.13_

- [ ] 5. Implement the conditional upsert
  - `ON CONFLICT ON CONSTRAINT events_identity`
  - Compare with `IS DISTINCT FROM`, not `<>`
  - Hold `superseded_at` inside the comparison tuple and reset it to `NULL` on update
  - _Requirements: 8.9_

- [ ] 6. Implement supersession in the same transaction
  - Scoped to the `week_key` the payload yields, retiring live rows whose `event_key` is absent
  - Soft delete only; reads default to `WHERE superseded_at IS NULL`
  - _Requirements: 8.10, 8.11_

- [ ] 7. Write `fetch_log` on every path
  - Including failures. No in-process sleep-retry - the paired retry slot handles it
  - _Requirements: 8.5_

- [ ] 8. Confirm the handler does not re-derive guard logic
  - `grep -rn "already_satisfied\|outcome" src/handler.py` shows it calling the function and acting on the answer
  - The only static check in the spec, and the only thing that verifies task 1's orchestration-only rule
  - _Requirements: 8.2_

## Gate

Run locally twice against the real feed, respecting the 5-minute floor:

- run 1: one `ok` in `fetch_log`; a `raw/` object exists and its sha256 matches `raw_snapshots.payload_sha256`
- run 2: `events_new=0, events_updated=0, events_superseded=0` - idempotence, checked before deploying anything
- `SELECT count(DISTINCT week_key) FROM events` = 1

Then the fixture replays against a clean database:

- **supersession** - `this_WED.json` then `this_SAT.json` gives **74 live, 5 superseded** (79 without). The five: `AUD RBA Gov Bullock Speaks`, `CNY Foreign Direct Investment ytd/y`, `CNY M2 Money Supply y/y`, `CNY New Loans`, `USD Mortgage Delinquencies`
- **revive** - re-harvest `this_WED.json`, all five return to live
- **scope** - harvest `this_SUN.json` and confirm the week-of-08-09 rows are untouched
- a saved HTML rate-limit body produces `rate_limited` with zero writes to `events`
- two invocations within one minute produce two distinct `raw/` keys
