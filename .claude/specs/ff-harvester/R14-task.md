# R14 - Two-week unattended acceptance · Tasks

Nine checks, every one a query someone actually ran. Two consecutive weeks, untouched.

- [ ] 1. Let two consecutive weeks run unattended
  - No manual invocations, no edits
  - _Requirements: 14.1_

- [ ] 2. Run the nine checks as SQL and S3 reads, in one sitting
  - `fetch_log`: an `ok` per slot, **no gap > 8h**, **zero** `rate_limited` - _14.1_
  - Exactly two `week_final=true` snapshots, each S3 object **fetched**, not just its row read - _14.2_
  - Re-run a completed slot: `events_new=0, events_updated=0, events_superseded=0` - _14.3_
  - Each identity tuple appears once per `week_key`; rows returned here mean the constraint is wrong, not the data - _14.4_
  - Live rows per `(country, week_key)` **equal** the final payload count. **Read the direction:** below means `normalize_title` over-merged, above means supersession did not run - _14.5_
  - Sunday events share a `week_key` with the Monday events after them - _14.6_
  - Zero parse warnings on non-blank values, or each traced to a format now covered by a fixture - _14.7_
  - `exports/<date>/` holds one CSV and one JSON per run, row counts matching live rows at that time - _14.8_
  - **Two `HEALTH` lines, both `ok`** - if the health check never ran, the other checks told you nothing - _14.9_

- [ ] 3. Paste the outputs into the vault
  - `Notes/2026-08-17 - Economic Data Aggregator` in the Obsidian vault
  - Step 0 was deleted from the build document, so the vault is the sole record of decision history
  - _Requirements: 14.1-14.9_

## Gate

All nine clean -> MVP done.

Actuals (spec section 10) are the next iteration and need **no migration** - the four `actual_*` columns already exist. Check the feed before reaching for FMP or FRED: actuals surface as the *following* week's `previous`.
