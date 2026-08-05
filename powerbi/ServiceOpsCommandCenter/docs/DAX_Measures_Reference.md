# DAX Measures Reference

> **Not implemented in the shipped model.** Per a later scope change, the semantic model in this package
> is deliberately DAX-free (bare structural tables + built-in Count/Average aggregations only — see the
> README's "What's intentionally NOT here" section). This document is kept as a **build guide**: the
> measures and calculated columns below are what you'd add, and how, when you're ready to layer in real
> SLA/aging/time-intelligence logic. Nothing on this page currently exists in the `.tmdl` files.

Two calculated columns would underpin most of this: `Fact_ServiceRequests[SLA Outcome]` (Met / Breached /
At Risk / On Track / Cancelled, evaluated live against `NOW()`) and `Fact_ServiceRequests[AgingBucket]`
(0-1 Day / 1-3 Days / 3-7 Days / 7-14 Days / 14+ Days, for open items only).

## Volume

| Measure | DAX intent |
|---|---|
| `Total Requests` | `COUNTROWS('Fact_ServiceRequests')` |
| `Open Requests` | Total Requests where `IsOpen = TRUE` |
| `Completed Requests` | Total Requests where `Status = "Completed"` |
| `Cancelled Requests` | Total Requests where `Status = "Cancelled"` |
| `Backlog Volume` | Alias of `Open Requests` |
| `Unassigned Requests` | Open Requests still in `New`/`Assigned` (not yet started) |
| `Open Over 3 Days` | Open Requests with `AgingHours > 72` |

## SLA

| Measure | DAX intent |
|---|---|
| `SLA Met` | Count where `SLA Outcome = "Met"` |
| `SLA Breached` | Count where `SLA Outcome = "Breached"` (covers completed-late **and** still-open-past-due) |
| `SLA At Risk` | Count where `SLA Outcome = "At Risk"` (open, <25% of SLA window remaining) |
| `SLA Compliance %` | `DIVIDE([SLA Met], [Completed Requests])` |
| `SLA Target %` | Fixed reference line, `0.90` |
| `Cancellation Rate` | `DIVIDE([Cancelled Requests], [Total Requests])` |

## Performance (hours)

| Measure | DAX intent |
|---|---|
| `Average Resolution Time (Hrs)` | `AVERAGE([ResolutionHours])` — Created → Completed |
| `Average Time to Start (Hrs)` | `AVERAGE([TimeToStartHours])` — Assigned → In Progress |
| `Average Handling Time (Hrs)` | `AVERAGE([HandlingHours])` — In Progress → Completed, minus hold time |
| `Average Hold Time (Hrs)` | `AVERAGE([HoldMinutesTotal]) / 60` |

## Time Intelligence

| Measure | DAX intent |
|---|---|
| `Previous Period Requests` | Total Requests for the equal-length window immediately before whatever date range is currently filtered — works for any slicer selection (a month, a quarter, a custom range), not just calendar months. This is what drives every "vs Previous Period" KPI card. |
| `Period Variance` / `Period Variance %` | `[Total Requests] - [Previous Period Requests]`, and the % version |
| `Requests PM` / `MoM Change %` | Prior calendar month via `DATEADD(..., -1, MONTH)` |
| `Requests PQ` / `QoQ Change %` | Prior calendar quarter |
| `Requests PY` / `YoY Change %` | `SAMEPERIODLASTYEAR` |
| `Rolling 3/6/12 Month Volume` | `DATESINPERIOD` trailing windows anchored to the max selected date |
| `YTD Requests` / `YTD SLA Compliance` / `YTD Resolution Time` | `TOTALYTD` / `DATESYTD` wrappers |
| `Reporting Period Label` | Text measure for the report header, e.g. "August 2026" or "January 2026 – June 2026", derived from `MIN`/`MAX` of the filtered `Dim_Date[Date]` |
| `Resolved Requests (Completed Date)` | Same as `Completed Requests`, but evaluated against the **completion** date axis (`USERELATIONSHIP` on the inactive `CompletedDate → Dim_Date` relationship) instead of the creation date axis — this is what the Overview trend line uses for the "Resolved" series. |

## Fact_StatusHistory

| Measure | DAX intent |
|---|---|
| `Status Change Events` | `COUNTROWS('Fact_StatusHistory')` |
| `Requests That Had a Hold` | Distinct count of requests with at least one `On Hold` event |

## Notes for extension

- `SLA Outcome` and `AgingHours`/`AgingBucket` use `NOW()`, so they're live relative to whenever the
  report is opened/refreshed — appropriate for an operational dashboard, but it means the bundled demo
  data (frozen as of Aug 5, 2026) will look increasingly "aged" the further past that date you open it.
  Refresh against real data on a schedule and this resolves itself.
- To add Region/Team cutting, extend `Dim_Assignee` with a real source column (currently a placeholder
  constant `"Service Ops"` for every assignee) and everything else — relationships, measures, visuals —
  continues to work unchanged.
