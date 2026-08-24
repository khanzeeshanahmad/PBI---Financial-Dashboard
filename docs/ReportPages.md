# Report Build-Out Guide

The `.Report` project ships all 6 pages fully built out with a professional,
CFO/executive-oriented layout: KPI card rows, trend and composition charts,
and a shared custom theme (`FinanceExecutiveTheme.json`, registered in
`report.json`). This doc lists what's seeded per page vs. what's still quick
to finish interactively in Power BI Desktop. Nothing here requires new DAX or
Power Query — every measure/column referenced already exists in the model.

## 1. Executive Summary — seeded: 8 KPI cards (Revenue YTD, Net Income YTD,
Gross Margin % YTD, Cash and Bank, Revenue YoY %, Net Income YoY %, Total AR
Outstanding, Total AP Outstanding), Revenue trend line chart (Year legend for
YoY), Revenue-by-Department donut, Year slicer (horizontal, top bar).
Recommended additions:
- A sparkline can be added to each KPI card via the card visual's built-in
  "trend line" formatting option (Format pane → Callout value → enable trend
  axis using Dim_Date[Date]).
- Conditional font color on the two YoY % cards (green if positive, red if
  negative) via Format pane → Callout value → conditional formatting rules.

## 2. P&L Statement — seeded: 5 KPI cards (Revenue, COGS, Gross Profit,
Operating Expenses, Net Income), table (Account_Category → Name → Actual
Amount (IS)), Revenue/COGS/Net Income trend line chart by month, Year and
Month slicers.
Recommended additions:
- Drill-down: set the table's row hierarchy to Account_Category → Name
  (already two levels) and enable "Show next level" / drill icons in the
  visual header.
- Add a bookmark-driven toggle (two identical trend charts, one filtered to
  MonthShort for MoM, one to Year for YoY) if a literal toggle button is
  wanted instead of just filtering via slicers.

## 3. Balance Sheet — seeded: 4 KPI cards (Total Assets, Total Liabilities,
Total Equity, Balance Sheet Check), table (Account_Category → Subcategory →
Name), Balance Sheet composition donut (`BS Category Amount` by
Account_Category).
Recommended additions:
- A 100% stacked bar of Assets vs. (Liabilities + Equity) by month as a
  second visual balance check, using the same cumulative-balance caveat as
  below.
- Note: none of the Balance Sheet measures are point-in-time/cumulative —
  they sum G/L activity within whatever date filter is applied. Trending them
  by month would show *period movement*, not the running balance. A true
  "Balance Sheet as of [date]" trend needs a new cumulative measure (e.g.
  `CALCULATE([Total Assets], FILTER(ALL(Dim_Date), Dim_Date[Date] <= MAX(Dim_Date[Date])))`)
  — left out of this seed since it changes the measure semantics and should
  be reviewed against how the business wants "as of" balances defined.

## 4. Budget vs Actual — seeded: 4 KPI cards (Budget Amount, Actual Amount,
Budget Variance, Budget Variance %), clustered column (Actual vs Budget by
department), variance table by account.
Recommended additions:
- Add a Year/Budget Name slicer (`Fact_Budget[Budget_Name]`) if more than one
  named budget/forecast exists.
- Conditional formatting (data bars or color scale) on `Budget Variance %` in
  the variance table — red/green by sign.

## 5. AR/AP Aging — seeded: 4 KPI cards (Total AR/AP Outstanding, AR/AP
Current), AR and AP aging-by-bucket column charts (using the new
`Dim_AgingBucket` disconnected table + `AR Aging Amount`/`AP Aging Amount`
dynamic measures, sorted Current → 90+ Days), bar chart AR by customer, table
AP by vendor.
Recommended additions:
- Drill-through: right-click a customer/vendor bar → "Drillthrough" → send to
  Transaction Detail, after adding Customer_No/Vendor_No as a drillthrough
  filter field on that page (Format pane → Drillthrough).
- Color the aging-bucket columns red-to-green (or a single warm-to-cool
  gradient) by bucket via conditional formatting on the column's fill, if a
  stronger "risk" visual read is wanted than the theme's flat accent color.

## 6. Transaction Detail — seeded: 4 KPI cards (Transaction Count, Total
Debit, Total Credit, Net Amount), Year and Month slicers (side by side), full
G/L entry table.
Recommended additions:
- Mark this page as a **drillthrough target**: Format pane → Drillthrough →
  add `Dim_ChartOfAccounts[No]`, `Dim_Customer[No]`, `Dim_Vendor[No]` as
  drillthrough fields as needed, matching what each summary page should pass
  through.
- Add a search/filter box (slicer with search) on `Document_No` or
  `Description` for ad-hoc lookups.

## Theme

`Financial Dashboard.Report/StaticResources/RegisteredResources/FinanceExecutiveTheme.json`
is a custom Power BI theme (navy/blue/gold corporate palette, Segoe UI
Semibold titles and callouts, card-style visuals with a subtle border and
drop shadow) applied report-wide via `report.json`'s
`themeCollection.customTheme` + a second `RegisteredResources` entry in
`resourcePackages`. Editing colors/fonts for the whole report only requires
touching this one file — Format pane → General won't be needed per visual.
To adjust it further in Desktop: **View → Themes → Customize current theme**,
or hand-edit the JSON (standard Power BI custom theme schema — dataColors,
background/foreground, good/neutral/bad, textClasses, visualStyles).

## Why interactive features are still left for Desktop

Drillthrough wiring, bookmarks, and conditional-formatting rules are stored
in PBIR as fairly deep, UI-generated JSON with details (bookmark target
GUIDs, per-visual filter-pane state) that are easy to get subtly wrong by
hand and hard to verify without Power BI Desktop itself. The visuals seeded
in this project use the same JSON shape and were built the same way, so
adding more of the same kind (another card, another slicer, another table
column) is safe to copy from the existing `visual.json` files — but the
above interactive features are genuinely faster and safer to add with the
Desktop UI, which generates guaranteed-valid definitions for you.
