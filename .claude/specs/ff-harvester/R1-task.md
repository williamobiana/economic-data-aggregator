# R1 - Value parser · Tasks

Pure module. No network, no AWS, no database.

- [ ] 1. Create `src/parse.py` with the frozen `Value`
  - `@dataclass(frozen=True)` carrying `num: Decimal | None`, `unit`, `suffix`, `flag`
  - Never a dict; attribute assignment must raise
  - _Requirements: 1.1, 1.2_

- [ ] 2. Implement `parse(raw: str) -> Value`
  - Strip whitespace and thousands separators before matching
  - Blank input returns an all-`None` `Value` with no warning
  - `%` sets `unit='percent'`; a bare number sets `unit='none'`
  - `K`/`M`/`B` expand `num` fully and report `suffix` as provenance
  - Leading `<` / `>` sets `flag` to `lt` / `gt`
  - _Requirements: 1.3, 1.4, 1.5, 1.6_

- [ ] 3. Implement the fallback path
  - Unknown shapes log one warning and return an all-`None` `Value`
  - Never raise, never branch on `impact`
  - _Requirements: 1.7, 1.8_

- [ ] 4. Write `tests/test_parse.py`
  - One case per row of the R1.4 table
  - Plus `"4.58|2.6"` (bond auction), a comma-bearing input, a whitespace-padded input
  - Assert `isinstance(v.num, Decimal)` and that assignment to a field raises
  - _Requirements: 1.1, 1.2, 1.4, 1.7_

## Gate

`pytest tests/test_parse.py` green, all cases above present.
