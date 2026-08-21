# R10 - Harvester deployed and hand-invoked · Tasks

Catch a wrong connection string, a missing permission or a wrong-platform layer before any schedule exists.

- [ ] 1. Create the function
  - `python3.11`, 30-second timeout, **reserved concurrency 1**
  - Execution role and layer from R9; env vars for bucket and parameter name
  - _Requirements: 10.1_

- [ ] 2. Set the async invoke config
  - **`MaximumRetryAttempts: 0`** - separate console screen, and the easiest setting here to miss
  - Recovery belongs to the scheduled retry ten minutes later; an immediate in-place retry gets the IP rate-limited
  - _Requirements: 10.2_

- [ ] 3. Invoke manually, before any schedule exists
  - _Requirements: 10.3_

- [ ] 4. Set log retention to 30 days
  - The log group does not exist until after the first invoke, and defaults to never expire
  - _Requirements: 10.4_

- [ ] 5. Note the runtime deprecation date
  - With no template, a runtime bump is a manual edit per function
  - _Requirements: 10.5_

## Gate

After the manual invoke: a new object under `raw/`, new rows in `events`, exactly one `ok` in `fetch_log` for that time, and `aws logs describe-log-groups` showing `retentionInDays: 30`.

Read reserved concurrency and `MaximumRetryAttempts` back from the deployed configuration rather than assuming them.
