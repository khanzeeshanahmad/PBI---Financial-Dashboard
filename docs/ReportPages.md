# Report Build-Out Guide

The `.Report` project already contains all 6 pages with a working primary
visual (or two) on each, built from real fields in the semantic model. This
doc lists what's seeded vs. what's quick to add in Power BI Desktop to reach
the full spec in the brief. Nothing here requires new DAX or Power Query —
every measure/column referenced already exists in the model.

## 1. Executive Summary — seeded: 4 KPI cards, Revenue trend line chart (Year
legend for YoY), Year slicer.
Recommended additions:
- Net Income trend line chart (mirror the Revenue trend visual)
- Card comparing `Net Income YoY %` next to the Net Income card (conditional
  formatting: green if positive)
- A sparkline can be added to each KPI card via the card visual's built-in
  "trend line" formatting option (Format pane → Callout value → enable trend
  axis using Dim_Date[Date]).

## 2. P&L Statement — seeded: table (Account Category → Name → Actual Amount
(IS)), Year slicer, Month slicer.
Recommended additions:
- Add `Operating Expenses`, `Gross Profit`, `Net Income` as extra table
  columns or as a small multiples matrix by MonthShort for a MoM view.
- Drill-down: set the table's row hierarchy to Account Category → Name (already
  two levels) and enable "Show next level" / drill icons in the visual header.
- Add a bookmark-driven toggle (two identical tables, one filtered to
  MonthShort = selected month for MoM, one to Year for YoY) if a literal
  toggle button is wanted instead of just filtering via slicers.

## 3. Balance Sheet — seeded: table (Account Category → Subcategory → Name,
Assets/Liabilities/Equity totals).
Recommended additions:
- A waterfall or 100% stacked bar of Total Assets vs (Liabilities + Equity)
  for a visual balance check next to the `Balance Sheet Check` measure.

## 4. Budget vs Actual — seeded: clustered column (Actual vs Budget by
department), variance table by account.
Recommended additions:
- Add a Year/Budget Name slicer (`Fact_Budget[Budget Name]`) if more than one
  named budget/forecast exists.
- Conditional formatting (data bars or color scale) on `Budget Variance %` in
  the variance table.

## 5. AR/AP Aging — seeded: AR bar chart by customer, AP table by vendor.
Recommended additions:
- Mirror a bar chart for AP and a table for AR customer detail (currently one
  of each; duplicating the pattern for the other side takes minutes).
- Add a "Total AR Outstanding" / "Total AP Outstanding" card pair at the top.
- Drill-through: right-click a customer/vendor bar → "Drillthrough" → send to
  Transaction Detail, after adding Customer No./Vendor No. as a drillthrough
  filter field on that page (Format pane → Drillthrough).

## 6. Transaction Detail — seeded: full G/L entry table, Year slicer.
Recommended additions:
- Mark this page as a **drillthrough target**: Format pane → Drillthrough →
  add `Dim_ChartOfAccounts[No.]`, `Dim_Customer[No.]`, `Dim_Vendor[No.]` as
  drillthrough fields as needed, matching what each summary page should pass
  through.
- Add a search/filter box (slicer with search) on `Document No.` or
  `Description` for ad-hoc lookups.

## Why these are left for Desktop rather than hand-authored

Drillthrough wiring, bookmarks, and conditional-formatting rules are stored
in PBIR as fairly deep, UI-generated JSON with details (bookmark target
GUIDs, per-visual filter-pane state) that are easy to get subtly wrong by
hand and hard to verify without Power BI Desktop itself. The visuals seeded
in this project use the same JSON shape and were built the same way, so
adding more of the same kind (another card, another slicer, another table
column) is safe to copy from the existing `visual.json` files — but the
above interactive features are genuinely faster and safer to add with the
Desktop UI, which generates guaranteed-valid definitions for you.
