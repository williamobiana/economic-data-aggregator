# R4 - Retry guard · Tasks

Pure module. All retry skip logic lives here so the EventBridge schedule stays dumb.

- [ ] 1. Create `src/guard.py` with `already_satisfied(rows, slot, now) -> bool`
  - `rows` are `fetch_log` rows the caller already fetched; `now` is a parameter, not a clock read
  - Open no connection and issue no query
  - _Requirements: 4.1, 4.5_

- [ ] 2. Implement the outcome logic
  - Paired attempt with outcome `ok` -> satisfied
  - `rate_limited`, `parse_error`, `network`, `empty`, or no attempt at all -> unsatisfied
  - _Requirements: 4.2, 4.3_

- [ ] 3. Implement the 5-minute rate-limit floor
  - Any attempt within 5 minutes of `now` -> satisfied, regardless of outcome
  - _Requirements: 4.4_

- [ ] 4. Write `tests/test_guard.py`
  - One case per outcome value, the empty-rows case, the within-5-minutes case at a fixed reference instant, and a row belonging to a *different* slot (must not satisfy)
  - _Requirements: 4.2, 4.3, 4.4_

## Gate

`pytest tests/test_guard.py` green. Nothing else - this requirement is entirely offline.

The centralization claim this requirement exists to serve ("nothing else re-derives the skip decision") is checked at **R8**, against a `handler.py` that exists by then.
