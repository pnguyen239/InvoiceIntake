# SaskTel Operations Command Center — Power BI Design Package

A design/mockup package for a Service Request Operations report: a SaskTel-branded theme, a full
three-layer report (Executive → Operations → Detail) built as real, working Power BI visuals, and an
interactive HTML wireframe — all wired to a small illustrative dataset so it opens populated and looking
right, with **no DAX or business logic baked in**. You bring the real data and measures; this package is
the design.

> **Scope note:** earlier drafts of this package included a full DAX measure library, calculated
> columns (SLA status, aging buckets, time intelligence), and a larger synthetic dataset. That's been
> intentionally stripped back to a bare-bones shell per your request — see
> [What's intentionally NOT here](#whats-intentionally-not-here) below. The DAX reference doc is kept as
> a guide for when you're ready to add that logic yourself.

> **Data note:** the model ships with a small **self-contained illustrative dataset** (60 requests / ~240
> status-history rows), shaped to match the field names visible in the SharePoint list screenshots you
> provided (`appRequestHeader`, `appReqHeaderStatus`, `appAssignees`). **No connection to any SharePoint
> site is made anywhere in this package** — the data is generated once and embedded as static values.

---

## 1. What's in this folder

```
ServiceOpsCommandCenter.pbip                     ← open THIS in Power BI Desktop
ServiceOpsCommandCenter.Report/                   ← report pages (Power BI Project format)
ServiceOpsCommandCenter.SemanticModel/            ← bare structural tables + illustrative data (TMDL format)
theme/SaskTel_CommandCenter_Theme.json            ← standalone theme file (also embedded in the report)
mockup/SaskTel_CommandCenter_Mockup.html          ← interactive HTML wireframe (open in any browser)
docs/DAX_Measures_Reference.md                    ← measures to build yourself, as a guide (not implemented)
README.md                                         ← this file
```

## 2. Getting a working `.pbit` from this package

A `.pbit`'s data-connection layer (`DataMashup`) is a proprietary binary package format with no public
spec, and this was built without access to Power BI Desktop to verify a hand-built one — a bad
`DataMashup` typically fails the *whole file* silently. So this is delivered as a **Power BI Project
(`.pbip`)** — plain text (TMDL + JSON), which Desktop parses natively with full fidelity.

**To get a `.pbit`, it's one extra step:**

1. Open **Power BI Desktop**. **File → Open → Browse** → select `ServiceOpsCommandCenter.pbip`.
2. **File → Save As → Power BI template files (\*.pbit)**.

## 3. What's actually in the model

Bare structural tables, no calculated columns, no DAX measures anywhere in the file:

- `Fact_ServiceRequests` — one row per request, columns matching `appRequestHeader`'s fields
  (`ReadableID`, `RequestType`, `Assignee`, `Status`, `IsUrgent`, `IsOpen`, `CreatedDateTime`,
  `SLADueDateTime`, `CompletedDateTime`, plus pre-computed `TimeToStartHours`/`HandlingHours`/
  `ResolutionHours`/`HoldMinutesTotal` as plain numbers, not formulas)
- `Fact_StatusHistory` — one row per status change, matching `appReqHeaderStatus`
- `Dim_Assignee`, `Dim_RequestType`, `Dim_Status`, `Dim_Priority` — small lookup tables, matching
  `appAssignees` and the choice fields on the header list
- `Dim_Date` — a standard calendar table (2025–2027), built entirely in Power Query (`Date.Year`,
  `Date.MonthName`, etc.) rather than DAX, so there's zero DAX anywhere in this file

Relationships are the plain star-schema joins (each `Fact_ServiceRequests` column to its matching
dimension, plus `Fact_StatusHistory → Fact_ServiceRequests`) — no calculated relationships, no
`USERELATIONSHIP`, nothing requiring DAX to function.

**Every visual in the report binds to a plain column** using Power BI's built-in aggregation (Count for
IDs, Average for the hour/minute columns) — that's set once, on the column itself
(`summarizeBy: count` / `summarizeBy: average` in the TMDL), not via a custom measure. A handful of cards
and charts (Open Requests, Completed Requests, Cancelled, etc.) use a simple visual-level filter
(`IsOpen = true`, `Status = "Completed"`, …) to scope the count — still no DAX, just a static filter
condition on the visual.

## 4. What's intentionally NOT here

Compared to a fully analytical build, this shell has no:

- **SLA status logic** (Met / Breached / At Risk) — that's inherently time-relative (`NOW()`-based) DAX
- **Aging buckets** (0-1 / 1-3 / 3-7 / 7-14 / 14+ days) — same reason
- **Time intelligence** (MoM/QoQ/YoY, rolling windows, YTD, "previous period" comparisons)
- **KPI variance/trend arrows** on the cards (needs a comparison measure)

`docs/DAX_Measures_Reference.md` documents all of this as a **catalog of what to build**, with the
intended DAX for each — not because it's in the file, but as a starting point for whoever adds it. The
report pages already have the right cards/charts/tables laid out and titled for this logic (e.g., the
"Backlog & Status" page), so adding a measure to the model and swapping a card's field for it in Desktop
is a small, targeted change per KPI rather than a rebuild.

## 5. The HTML mockup (design reference)

Open `mockup/SaskTel_CommandCenter_Mockup.html` in any browser — no server needed. This one *does* carry
real interactive logic client-side (it's plain JavaScript, not a Power BI file), including the SLA/aging
business rules, so it's the best reference for what the fully-built-out version should look and feel
like: global date quick-filters, a live-updating reporting-period label, drill-through modals, and a
responsive mobile layout. Treat it as the target design; the `.pbip` is the real-Power-BI starting point
to build toward it.

## 6. Theme

`theme/SaskTel_CommandCenter_Theme.json` — full SaskTel palette (`#00539B` primary, `#0072CE` secondary,
`#00A651`/`#F2A900`/`#DA291C` status colors, `#FAFBFC` page background), Fluent-style rounded cards, soft
shadows. Already embedded in the report; also shipped standalone for use elsewhere via
**View → Themes → Browse for themes**.

## 7. Report pages

All 5 pages exist as real, populated Power BI visuals (not placeholders):

- **Overview** — status snapshot (Open/Urgent/On Hold/Cancelled counts), 4 KPI cards, a monthly trend
  line, By Type / By Status charts, an assignee snapshot table
- **Workload & Performance** — requests and avg handling time by assignee, workload distribution,
  agent leaderboard table
- **Backlog & Status** — open backlog by status, requests by type, cancellations trend, open worklist
- **Trends & Capacity** — volume by type over time, intake by day of week, open items by type, requests
  by priority
- **Request Detail** (drillthrough target) — request detail table, 4 performance-metric cards, lifecycle
  status-history table. Wire up the actual "right-click → Drillthrough" behavior via Desktop's Page
  Information → Drillthrough field well (a 30-second manual step; the page and visuals already exist).

## 8. Repointing to real data

Everything is written against the column names actually seen in your SharePoint lists' screenshots, so
swapping the source is a Power Query change, not a model rebuild — open each table's query in
**Transform data**, replace the `Source` step with **Get Data → SharePoint Online List**, and map columns
to what's already there (`requestor.Title → Requestor`, `assignee.Title → Assignee`,
`requestType.Value → RequestType`, etc.). This is entirely yours to do against your own SharePoint site —
nothing in this package references it.
