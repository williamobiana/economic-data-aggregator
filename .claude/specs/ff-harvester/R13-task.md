# R13 - Health line and alarms · Tasks

Make the absence of failure observable without paying for anything.

- [ ] 1. Add the gap query to the export function
  - Runs against `fetch_log` on every export invocation
  - _Requirements: 13.1_

- [ ] 2. Log exactly one HEALTH line
  - `HEALTH ok slots=28 gaps=0` or `HEALTH FAIL <reason>`
  - A missing Saturday sweep is reported explicitly, not as a generic gap
  - _Requirements: 13.1, 13.2_

- [ ] 3. Create two CloudWatch alarms, **no actions attached**
  - General failure - informational, since everything self-heals on the next slot
  - Saturday sweep - a missed sweep loses the closing week's final state permanently and warrants acting the same evening
  - _Requirements: 13.3_

- [ ] 4. Do not wire SNS or email
  - Wire it when remembering to look becomes the failure mode
  - _Requirements: 13.4_

## Gate

Invoke the export function and find one `HEALTH` line in its log group.

Both alarms exist and list zero alarm actions.
