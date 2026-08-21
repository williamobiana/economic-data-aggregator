# R11 - Schedules · Tasks

Fire until deleted, and never sweep on the wrong side of rollover.

- [ ] 1. Create the scheduler role
  - Trusts `scheduler.amazonaws.com`, `lambda:InvokeFunction` on the harvester
  - Its own role - do not reuse the Lambda execution role. R12 extends this same role rather than creating a second
  - _Requirements: 11.1_

- [ ] 2. Create the four harvest schedules
  - `0 0,6,12,18 * * *` UTC
  - _Requirements: 11.2_

- [ ] 3. Create the week-final sweep
  - `0 11 * * 6` UTC (Sat 07:00 ET), input `{"week_final": true}`
  - Rollover is Sat 19:00 ET; this cron clears it by roughly twelve hours in either season
  - _Requirements: 11.3_

- [ ] 4. Create the retry schedules
  - `10 0,6,12,18 * * *`, plus `10 11 * * 6` carrying `week_final: true` for the sweep
  - They fire unconditionally; the guard makes a healthy retry a free no-op
  - Configuration only - no skip logic here, it belongs in `guard.py`
  - _Requirements: 11.4, 11.5_

## Gate

Let one full day pass unattended, then:

- `fetch_log` shows four `ok` rows at the expected hours
- each retry slot appears as an immediate no-op, guard-satisfied, with no second fetch within 5 minutes of an `ok`
- zero `rate_limited`

Then inject a failure - temporarily break the SSM parameter before one main slot - and confirm the paired retry recovers it.
