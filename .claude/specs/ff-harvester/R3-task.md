# R3 - Event identity · Tasks

Pure module. The one definition of "same event". Over-merging here is the one corruption no constraint catches.

- [ ] 1. Create `src/identity.py` with `normalize_title(s) -> str`
  - Unescape `\/` to `/`, trim, collapse internal whitespace, lowercase
  - _Requirements: 3.1_

- [ ] 2. Implement `event_key(country, normalized_title, week_key, event_timestamp) -> str`
  - sha1 hex over the four parts; convenience only, the DB constraint is the authority
  - `event_timestamp` at full precision - no truncation to the date
  - `source_suffix` must not appear anywhere in this function
  - _Requirements: 3.2, 3.3, 3.4_

- [ ] 3. Write `tests/test_identity.py` - the negative assertion
  - The same `(country, date)` across the payloads in `json-files/` yields an identical normalized title *and* an identical `event_key`
  - _Requirements: 3.5_

- [ ] 4. Add the must-not-collapse cases
  - `FOMC Member Hammack Speaks` twice in one week gives two distinct keys
  - The ADP `08:14` / `08:15` pair gives two distinct keys
  - _Requirements: 3.4_

- [ ] 5. Add the cross-country collision cases
  - `Flash Manufacturing PMI` (AUD, EUR, GBP, JPY, USD), `Flash Services PMI` (AUD, EUR, GBP, USD), `Unemployment Rate` (AUD, CNY, GBP), and the remaining 14 of the 17 measured collisions - all distinct per country
  - _Requirements: 3.2_

- [ ] 6. Add the normalization equivalence case
  - `"  German  Factory\/Orders m\/m  "` and `"German Factory/Orders m/m"` produce one key
  - _Requirements: 3.1_

## Gate

`pytest tests/test_identity.py` green with all six task groups present.
