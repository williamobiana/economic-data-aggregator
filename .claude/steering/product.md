# Product

FF Harvester archives the ForexFactory economic calendar feed into Postgres on a schedule.

The feed (`https://nfs.faireconomy.media/ff_calendar_thisweek.json`) publishes **only the current week**. `ff_calendar_nextweek.json` 404s. A week you fail to capture is gone permanently — there is no backfill.

**Objective: never miss a week, never corrupt a value.**

## Scope

In scope: schedule, forecast, previous — for `USD EUR GBP JPY AUD NZD CAD CHF CNY` (the 9 codes the feed emits), **all impact levels** including Holiday. You cannot retro-collect what you filter out.

Out of scope: actuals. The four `actual_*` columns exist now and stay NULL, so that iteration needs no migration.

Output: an `events` table plus a dated CSV/JSON export to S3.

## Cadence

| Job | Cron (UTC) |
|---|---|
| Harvest | `0 0,6,12,18 * * *` |
| Week-final sweep | `0 11 * * 6` (Sat 07:00 ET) |
| Retry | each slot + 10 min, fires unconditionally |
| Export | Sunday, after the sweep |

The sweep is deliberately early. Rollover is **Sat 19:00 ET** — anchored to ET, since a UTC anchor drifts an hour in November. A sweep on the wrong side of it loses the closing week for good; Sat 07:00 ET clears it by roughly twelve hours in either season. Early costs nothing — there is no `actual` field and past rows are byte-identical across re-fetches.

## Definition of done

Two consecutive unattended weeks, every check a query:

- An `ok` row per slot; no gap > 8h; zero `rate_limited`.
- Two `week_final=true` snapshots, each with a readable S3 object.
- Re-running a completed slot yields `events_new=0, events_updated=0, events_superseded=0`.
- One row per identity tuple per `week_key`; Sunday events share a `week_key` with the Monday events after them.
- Live rows per `(country, week_key)` **equal** the final payload count. Below means `normalize_title` over-merged; above means supersession did not run.
- Two `HEALTH ok` lines.

## Non-goals

$0/month operating cost. No servers, no VPC, no IaC, no SNS yet, no UI, no API.
