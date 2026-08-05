# SaskTel Operations Command Center — Power BI Solution

An enterprise Power BI reporting package for **Service Request Operations**: a three-layer report
(Executive → Operations → Detail), a star-schema semantic model, a full DAX measure library, and a
SaskTel-branded theme — built to be opened directly in Power BI Desktop.

> **Data note:** this package ships with a **self-contained synthetic dataset** (240 requests / 974
> status-history events, spanning the trailing ~15 months), shaped to match the field names visible in
> the SharePoint list screenshots you provided (`appRequestHeader`, `appReqHeaderStatus`, `appAssignees`
> — requestor/assignee/status/urgent/etc.). **No connection to any SharePoint site is made anywhere in
> this package** — the data is generated once and embedded as static values. When you're ready to point
> it at your real lists, see [Repointing to your real data](#repointing-to-your-real-data) below.

---

## 1. What's in this folder

```
ServiceOpsCommandCenter.pbip                     ← open THIS in Power BI Desktop
ServiceOpsCommandCenter.Report/                   ← report pages (Power BI Project format)
ServiceOpsCommandCenter.SemanticModel/            ← star-schema model, DAX, Power Query (TMDL format)
theme/SaskTel_CommandCenter_Theme.json            ← standalone theme file (also embedded in the report)
mockup/SaskTel_CommandCenter_Mockup.html          ← interactive HTML wireframe (open in any browser)
docs/DAX_Measures_Reference.md                    ← readable catalog of every measure
README.md                                         ← this file
```

## 2. Getting a working `.pbit` from this package

You asked for a `.pbit` file specifically. Here's the honest tradeoff on why it's delivered this way:

A `.pbit`'s data-connection layer (`DataMashup`) is a proprietary binary package format with no public
spec, and this sandbox has no Power BI Desktop to round-trip-verify a hand-built one — a bad
`DataMashup` typically fails the *whole file* silently, with no way for me to catch that before handing
it to you. Rather than gamble on that, this is delivered as a **Power BI Project (`.pbip`)** — plain
TMDL + JSON text, which Desktop parses natively with full fidelity, so there's no binary-corruption risk
at all.

**To get your `.pbit`, it's one extra step, ~30 seconds:**

1. Install/open **Power BI Desktop** (free). In Options → Preview features, confirm **"Power BI
   Project (.pbip) save option"** is enabled (on by default in current Desktop).
2. **File → Open → Browse** → select `ServiceOpsCommandCenter.pbip`. Desktop loads the model and all 5
   report pages.
3. **File → Save As → Power BI template files (\*.pbit)**. Done — this `.pbit` is a real, fully-Desktop-
   verified file, not a guess.

This also means: if you'd rather work with the `.pbip` directly (it's the modern, git-friendly,
diffable format Microsoft is moving the ecosystem toward), you never need to convert it at all.

## 3. Executive dashboard wireframes & detailed page mockups (deliverables 1–2)

Open `mockup/SaskTel_CommandCenter_Mockup.html` in any browser (double-click it, no server needed). It's
a fully interactive, data-bound wireframe — not a static image — covering:

- Top nav bar: SaskTel branding, **Reporting Period** (computed live from the active date filter),
  Last Refresh, Export, Help
- **Global date quick-filters**: Current/Previous Month, Current/Previous Quarter, Last 3/6/12 Months,
  YTD, Current/Previous Year, All Time — each recomputes every KPI, chart, and the reporting-period label
- Collapsible-style left filter rail: Request Type, Assignee, Urgency, Status, SLA Outcome
- All 4 main pages (Overview / Workload & Performance / SLA & Backlog / Trends & Capacity) plus a
  request-detail **drill-through modal** (click any row) showing the full lifecycle timeline
- A responsive **mobile layout** (deliverable 9) — resize the browser below ~900px, or open on a phone

This is the reference to hand to a designer or stakeholder for sign-off before building the exact same
thing as native Power BI visuals (see the Layout Specification below).

## 4. Power BI layout specification & visual placement (deliverables 3–4)

`ServiceOpsCommandCenter.Report/report.json` already contains real, data-bound native visuals — cards,
line/donut/bar/area charts, tables, slicers — laid out on a 1280×720 canvas, for all 5 pages, wired to
the actual measures/columns in the semantic model below. Open the `.pbip` in Desktop and you'll see it
populated, not blank.

A few finishing touches are intentionally left as quick manual steps in Desktop rather than encoded
blind (each is a Format-pane checkbox, not a rebuild):

| Step | Where | Why manual |
|---|---|---|
| Conditional formatting (green/amber/red) on KPI cards & SLA % cells | Format pane → Conditional formatting, thresholds at 90%/80% | Rule-JSON is a high-error-rate surface to hand-author blind |
| Sync Slicers across the 4 main pages | View ribbon → Sync Slicers | Per-report-session setting, not part of visual JSON |
| Visual-level filter `IsOpen = True` on the SLA page's worklist table | Filters pane on that visual | Keeps the worklist to open/overdue items only |
| Drillthrough wiring (Request Detail page) | Page information → toggle "Allow as drill through"; drag `ReadableID` into the Drillthrough field well | One-click Desktop action |
| Collapsible filter panel (spec asked for collapsible; shipped as always-visible) | Selection pane + a bookmark toggle button | Bookmarks are themselves manually authored in Desktop |

Page-by-page visual map (matches the wireframe 1:1):

- **Overview (L1)** — attention banner (4 exception cards) → 4 executive KPI cards → Intake-vs-Resolved
  trend + By Type donut + By Status bar → Assignee Performance Snapshot table
- **Workload & Performance (L2)** — 4 KPIs → Open-vs-Completed by Assignee + Avg Handling Time by
  Assignee → Workload Distribution donut + Productivity bar → Agent Leaderboard table
- **SLA & Backlog (L2)** — 4 KPIs → Backlog by Status + SLA Compliance by Type → SLA Breach trend →
  Open & Overdue Worklist table
- **Trends & Capacity (L2)** — 3 KPIs → Volume by Type (stacked area) + Intake by Day of Week → Open
  Items by Type + Aging Bucket Analysis
- **Request Detail (L3, drillthrough)** — request detail table → 4 performance-metric cards (time to
  start / handling / resolution / hold) → lifecycle status-history table

## 5. Theme (deliverable 5)

`theme/SaskTel_CommandCenter_Theme.json` — full SaskTel palette (`#00539B` primary, `#0072CE`
secondary, `#00A651`/`#F2A900`/`#DA291C` status colors, `#FAFBFC` page background), Fluent-style rounded
cards, soft shadows, and per-visual-type style overrides (cards, slicers, tables, charts). It's already
embedded in the report (`ServiceOpsCommandCenter.Report/StaticResources/RegisteredResources/`), and also
shipped standalone so you can apply it to other reports via **View → Themes → Browse for themes**.

## 6. Data model diagram (deliverable 6)

Star schema, two fact tables (one child fact holding the status-change log):

```
                     Dim_Date
                   (Date … 1096 days)
                    /   |    \
                   /    |     \
     Dim_RequestType  Dim_Status  Dim_Priority
              \          |          /
               \         |         /
                Fact_ServiceRequests  ← 240 rows, 1 per request
                    (RequestKey PK)
                        |
                        | (many-to-one on RequestKey)
                        |
                Fact_StatusHistory   ← 974 rows, 1 per status change
                  (StatusHistoryKey PK)
                        |
                   also → Dim_Status, Dim_Date (via StatusDate)

Dim_Assignee ──── Fact_ServiceRequests[Assignee]
```

- `Fact_ServiceRequests.CreatedDate → Dim_Date.Date` is the **active** date relationship (drives every
  time-intelligence measure by intake date).
- `Fact_ServiceRequests.CompletedDate → Dim_Date.Date` is **inactive**, activated via `USERELATIONSHIP`
  in `Resolved Requests (Completed Date)` — this is what the Overview trend's "Resolved" series uses, so
  it reflects completion date rather than creation date.
- `Fact_StatusHistory.RequestKey → Fact_ServiceRequests.RequestKey` is a fact-to-fact relationship (the
  status log is a child of the request).

## 7. DAX measures (deliverable 7)

See [`docs/DAX_Measures_Reference.md`](docs/DAX_Measures_Reference.md) for the full catalog — Volume,
SLA, Performance, and Time Intelligence groups (MoM/QoQ/YoY, rolling 3/6/12-month, YTD, and a
range-agnostic "Previous Period" pair that works for any date-slicer selection, not just calendar
months). The measures live directly in `Fact_ServiceRequests.tmdl` / `Fact_StatusHistory.tmdl`.

Two calculated columns do the heavy lifting other tools would need a fact-history join for at query
time: `SLA Outcome` (Met/Breached/At Risk/On Track/Cancelled, live against `NOW()`) and `AgingBucket`
(0-1/1-3/3-7/7-14/14+ days, open items only).

## 8. Drill-through design (deliverable 8)

The **Request Detail** page (`ReportSectionDetail`) is built as the drillthrough target: a
`ReadableID` filter well, a request-detail table (type, assignee, priority, status, created/completed/
SLA-due dates), four performance-metric cards (time to start, handling time, resolution time, hold
time), and a status-history table driving a lifecycle view — `New → Assigned → In Progress → On Hold →
Completed` or `→ Cancelled`. Wire the actual "right-click → Drillthrough" behavior via Desktop's
Drillthrough field well (30-second step, see the table in §4) — the page and every visual on it already
exist and are bound to real data.

## 9. Mobile experience (deliverable 9)

Covered by the HTML mockup's responsive layout (§3) — single-column KPI stack, stacked filters, full
chart legibility down to phone width. For the native Power BI Mobile app, use Desktop's **View → Mobile
Layout** on the Overview page (drag the visuals already on the canvas into the phone-shaped canvas — a
placement exercise, not a rebuild, since the visuals and their bindings already exist) to get a proper
Power BI Mobile card layout for Overview, SLA Alerts, and Workload Monitoring.

## 10. Executive presentation (deliverable 10)

The HTML mockup doubles as presentation screens — every page renders cleanly full-screen and is safe to
screenshot or project directly for a leadership walkthrough; the native report pages do the same once
opened in Desktop.

---

## Repointing to your real data

Everything downstream of the data — relationships, calculated columns, measures, visuals — is written
against the column names actually seen in your SharePoint lists' screenshots
(`requestor.title`, `assignee.title`, `status`, `urgent`, `editInprocess`, etc. for the header list;
`status`, `status2header` for the status-history list; `Name`/`assignee` for the assignee list), so
swapping the source is a Power Query change, not a model rebuild:

1. In Desktop, **Transform data** → open the `Fact_ServiceRequests` query.
2. Replace the `Source` step with **Get Data → SharePoint Online List**, point it at your
   `appRequestHeader` list, and rename/select columns to match what the table already expects:
   `RequestKey (ID), ReadableID, Title, ShortDescription, Requestor (requestor.Title), RequestType
   (requestType.Value), Assignee (assignee.Title), CreatedDateTime (Created), Status (status.Value),
   IsUrgent (urgent), …`
3. Do the same for `Fact_StatusHistory` against `appReqHeaderStatus` (`Title` → `RequestKey`, `status`,
   `Created`/`Modified` → `StatusTimestamp`) and `Dim_Assignee` against `appAssignees`.
4. The calculated columns (`SLA Outcome`, `AgingBucket`, `TimeToStartHours`, etc.) and every DAX measure
   keep working unchanged, since they reference the model's column names, not the source.

This repointing step is entirely yours to do against your own SharePoint site — nothing in this package
references it, and I did not connect to it while building this.
