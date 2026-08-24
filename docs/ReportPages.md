# Report Build-Out Guide

The `.Report` project ships all 6 pages fully built out with a professional,
CFO/executive-oriented layout, redesigned per a `powerbi-report-design` skill
review — see `docs/DesignBrief.md` for the full brownfield critique and
rationale behind every layout/chart decision below. Every page reserves a
header band (page title left, slicers right-aligned) before content begins,
on a 24px margin / 16px gutter grid. This doc lists what's seeded per page vs.
what's still quick to finish interactively in Power BI Desktop. Nothing here
requires new DAX or Power Query — every measure/column referenced already
exists in the model.

## 1. Executive Summary — seeded: page title, Year slicer (horizontal, header
band), 4 KPI cards (Revenue YTD, Net Income YTD, Gross Margin % YTD, Cash and
Bank), Revenue trend line chart (Year legend for YoY), Revenue-by-Department
sorted bar chart.
Trimmed from an earlier 8-card version per the design review: YoY % cards were
dropped as redundant with the trend chart's Year-legend, and AR/AP outstanding
cards were dropped as already the dedicated subject of the AR/AP Aging page —
"KPI carpet-bombing" (>6 cards) is a flagged anti-pattern for a page meant to
be scanned in under 10 seconds. The Revenue-by-Department visual was changed
from a donut to a sorted bar chart, since department count is data-dependent
and unbounded — a donut risks becoming unreadable (or hitting the >5-slice
anti-pattern) at higher cardinality, and bars are the correct encoding for a
ranking question at any cardinality.
Recommended additions:
- A sparkline can be added to each KPI card via the card visual's built-in
  "trend line" formatting option (Format pane → Callout value → enable trend
  axis using Dim_Date[Date]).
- If a compact YoY read is wanted without a separate card, use the newer
  "Card" visual's Reference Label feature in Desktop (Format pane → Reference
  labels) to show a small delta/arrow under the main callout instead of a
  full second card.

## 2. P&L Statement — seeded: page title, Year and Month slicers (header
band), 5 KPI cards (Revenue, COGS, Gross Profit, Operating Expenses, Net
Income), table (Account_Category → Name → Actual Amount (IS)), Revenue/COGS/
Net Income trend line chart by month.
Recommended additions:
- Drill-down: set the table's row hierarchy to Account_Category → Name
  (already two levels) and enable "Show next level" / drill icons in the
  visual header.
- Add a bookmark-driven toggle (two identical trend charts, one filtered to
  MonthShort for MoM, one to Year for YoY) if a literal toggle button is
  wanted instead of just filtering via slicers.

## 3. Balance Sheet — seeded: page title, 4 KPI cards (Total Assets, Total
Liabilities, Total Equity, Balance Sheet Check), table (Account_Category →
Subcategory → Name), Balance Sheet composition donut (`BS Category Amount` by
Account_Category — kept as a donut, unlike the department chart, since it's
capped at exactly 3 slices and well within the readable range).
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

## 4. Budget vs Actual — seeded: page title, 4 KPI cards (Budget Amount,
Actual Amount, Budget Variance, Budget Variance %), clustered column (Actual
vs Budget by department), variance table by account.
Recommended additions:
- Add a Year/Budget Name slicer (`Fact_Budget[Budget_Name]`) if more than one
  named budget/forecast exists.
- Conditional formatting (data bars or color scale) on `Budget Variance %` in
  the variance table — red/green by sign.

## 5. AR/AP Aging — seeded: page title, 4 KPI cards (Total AR/AP Outstanding,
AR/AP Current), AR and AP aging-by-bucket column charts (using the
`Dim_AgingBucket` disconnected table + `AR Aging Amount`/`AP Aging Amount`
dynamic measures, sorted Current → 90+ Days), sorted bar chart of Total AR
Outstanding by customer, table of AP buckets by vendor.
The AR-by-customer chart was changed from a 5-series clustered bar (one
series per aging bucket, forcing a 5-color legend read) to a single sorted
bar of `Total AR Outstanding` — a much faster "who owes us the most" read,
and consistent with the same ranked-bar language used for department revenue
on Executive Summary. The bucket-level breakdown these series carried isn't
lost: the aging-by-bucket chart directly above already answers "how much is
at each risk stage," and the AP table still shows the full per-vendor bucket
breakdown for symmetry.
Recommended additions:
- Drill-through: right-click a customer/vendor bar → "Drillthrough" → send to
  Transaction Detail, after adding Customer_No/Vendor_No as a drillthrough
  filter field on that page (Format pane → Drillthrough).
- Color the aging-bucket columns on a green (Current) → red (90+ Days)
  semantic ramp instead of the theme's flat accent color — this is the
  textbook use case for a sequential/semantic palette since the bucket order
  is genuinely ordinal (best-to-worst collections risk). Left for Desktop's
  Format pane → Data colors rather than hand-authored, since it needs a
  per-category-value conditional-format override and no confirmed-safe PBIR
  JSON pattern for that exact shape was available this session — see
  `docs/DesignBrief.md`'s "Known deviations."

## 6. Transaction Detail — seeded: page title, Year and Month slicers (header
band), 4 KPI cards (Transaction Count, Total Debit, Total Credit, Net
Amount), full G/L entry table.
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
Semibold titles and callouts, card-style visuals with a subtle border —
no drop shadows, which the design review flagged as pure decoration)
applied report-wide via `report.json`'s
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
