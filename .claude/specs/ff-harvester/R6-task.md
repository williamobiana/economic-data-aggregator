# R6 - Schema migration · Tasks

Get the constraint right here. Changing it after rows exist means deduplicating first.

- [ ] 1. Write `migrations/001.sql` - `raw_snapshots`
  - `id, fetched_at, week_key, week_final, s3_key, byte_size, payload_sha256`
  - _Requirements: 6.2_

- [ ] 2. Add `events`
  - Identity columns, `title` and `normalized_title`, `impact`, `unit`, `source_suffix`
  - `forecast_raw` / `forecast_num` / `forecast_flag`, same for `previous`
  - `first_seen_at`, `last_updated_at`, `superseded_at`
  - The four reserved `actual_*` columns, NULL in the MVP so the actuals iteration needs no migration
  - _Requirements: 6.5, 6.6_

- [ ] 3. Declare the named constraint
  - `CONSTRAINT events_identity UNIQUE (country, normalized_title, week_key, event_timestamp)`
  - Named, not a convention - `ON CONFLICT` needs a real target. `source_suffix` stays out
  - _Requirements: 6.3, 6.4_

- [ ] 4. Add the three indexes
  - `events (event_timestamp)`, `events (country, event_timestamp)`, `events (week_key) WHERE superseded_at IS NULL`
  - _Requirements: 6.2_

- [ ] 5. Add `fetch_log`
  - `fetched_at, week_final, http_status, outcome, events_seen, events_new, events_updated, events_superseded`
  - `outcome` constrained to `('ok','rate_limited','parse_error','network','empty')`
  - _Requirements: 6.7_

- [ ] 6. Run it once, laptop to Supabase, as raw SQL
  - No Alembic, no migration framework
  - _Requirements: 6.1_

## Gate

Against the live database:

- `SELECT conname FROM pg_constraint WHERE conname = 'events_identity'` returns 1 row
- `SELECT indexdef FROM pg_indexes WHERE tablename = 'events'` shows 3 + pkey + unique
- Inserting the same `(country, normalized_title, week_key, event_timestamp)` twice raises `23505`
