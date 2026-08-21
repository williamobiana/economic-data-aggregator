# R2 - Week key · Tasks

Pure module. Test this harder than the parser - a bad week key duplicates rows silently.

- [ ] 1. Create `src/week.py` with `ff_week_start(d: datetime) -> date`
  - Convert the input to `America/New_York` via `zoneinfo.ZoneInfo`, then walk back to Sunday
  - Take the offset from the input; never construct or assume one
  - No `isocalendar()`, no Monday-start arithmetic anywhere in the file
  - _Requirements: 2.1, 2.2, 2.4_

- [ ] 2. Add the payload-level helper used by the handler
  - Week key for a payload is `ff_week_start(min(row.date))` over its own rows
  - _Requirements: 2.5_

- [ ] 3. Write `tests/test_week.py` - boundary cases
  - A Sunday event keys to itself
  - `2026-08-09T19:50:00-04:00` and `2026-08-16T18:30:00-04:00` key to the Sunday *before*, and match the Monday events following them
  - _Requirements: 2.1, 2.3_

- [ ] 4. Add the DST cases - **hand-built, no real payload exercises `-05:00`**
  - One event before the November change (`-04:00`) and one after (`-05:00`), each keying correctly
  - _Requirements: 2.4_

- [ ] 5. Add the regression case from `json-files/`
  - A real payload collapses to exactly one key
  - Assert the measured `isocalendar()` split it replaces: week of 08-09 gives `(2026,32)`x3 + `(2026,33)`x71; week of 08-16 gives `(2026,33)`x11 + `(2026,34)`x85
  - _Requirements: 2.2, 2.5_

## Gate

`pytest tests/test_week.py` green with all five task groups present.
