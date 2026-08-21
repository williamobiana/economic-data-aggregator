# R12 - Export function · Tasks

A dated, dumb, complete dump. Built before alerting, because R13's health check runs inside this function.

- [ ] 1. Create `src/export.py`
  - `csv.DictWriter` and `json.dump` to a temp file, then upload
  - Full dump, not incremental
  - Dump **live rows only** (`WHERE superseded_at IS NULL`)
  - _Requirements: 12.1, 12.5_

- [ ] 2. Write to dated keys
  - `exports/<date>/events.csv` and `exports/<date>/events.json`
  - A fixed key destroys the previous dump; the date replaces version history
  - _Requirements: 12.2_

- [ ] 3. Deploy it
  - Shares the harvester's execution role and dependency layer
  - Create, invoke by hand, set 30-day log retention - all before any schedule is attached
  - _Requirements: 12.3_

- [ ] 4. Schedule it once proven
  - Add its ARN to the existing scheduler role from R11
  - Sunday schedule `0 12 * * 0`, comfortably after the Saturday sweep
  - _Requirements: 12.4_

## Gate

Hand-invoke, then: both objects exist under today's `exports/<date>/`, both are readable, and the CSV row count equals `SELECT count(*) FROM events WHERE superseded_at IS NULL`.

Invoke again the next day and confirm the previous day's prefix is untouched.
