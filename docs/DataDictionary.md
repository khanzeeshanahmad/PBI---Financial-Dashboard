# Data Dictionary — Financial Dashboard (Business Central)

Semantic model: `Financial Dashboard.SemanticModel`. All import-mode tables are
sourced through the native **Dynamics 365 Business Central** Power BI
connector (`Dynamics365BusinessCentral.ApiContentsWithOptions`), parameterized
by environment and company — see `Parameters` below and the README for how to
point this at a different environment/company.

Column naming: every model-facing column name is sanitized from BC's native
field name — a trailing `.` is dropped and spaces/`/` become `_` (e.g. BC's
`G/L Account No.` is exposed in the model as `G_L_Account_No`). This avoids
Power Query M's bracket field-access syntax breaking on names with a literal
trailing period. The `BC source field` column below always shows BC's
original, unsanitized name; `sourceColumn` in the TMDL still points at that
original name since Power Query itself never renames these fields — only the
model-level column name changes.

Sign convention used throughout: G/L entries in Business Central post with
their natural debit/credit sign (credit-normal accounts — Income, Liabilities,
Equity — carry a negative `Amount`). Columns always keep BC's native sign so
they reconcile against BC's own G/L Entry list; DAX measures negate where
needed so Revenue, Total Liabilities, Total Equity, etc. read as positive
numbers on the reports.

---

## Parameters (`definition/expressions.tmdl`)

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `BCEnvironment` | Text | `DEMO` | BC environment name as shown in the BC admin center / connector Navigator. |
| `BCCompanyName` | Text | `Cronus - QMM` | BC company (legal entity) display name, exactly as it appears under the Environment node in the Navigator. |
| `BCApiVersion` | Text | `v2.0` | Documentation-only marker of which BC API/connector generation this project was authored against. |
| `GlobalDimension1Code` | Text | `DEPARTMENT` | Dimension Code mapped to this company's Global Dimension 1 slot (Company Information page). |
| `GlobalDimension2Code` | Text | `CUSTOMERGROUP` | Dimension Code mapped to Global Dimension 2 for this company. |
| `RangeStart` / `RangeEnd` | DateTime | 2015-01-01 / 2026-01-01 | Reserved names for Power BI Desktop's incremental refresh feature; used as the filter bounds in `Fact_GLTransactions` and referenced for `Fact_Budget`. |

Changing `BCEnvironment` or `BCCompanyName` and refreshing repoints **every**
table in the model — no query needs individual editing.

---

## Dimension tables

### Dim_Date
Calendar-generated table (`List.Dates`, not pulled from BC), marked as the
model's **official Date table** (`Model.MarkAsDateTable` / `dataCategory:
Time`), January–December, no fiscal offset. Spans January 1 of the earliest
year present in `Fact_GLTransactions[Posting_Date]` (computed via
`Date.StartOfYear(DateTime.Date(List.Min(Fact_GLTransactions[Posting_Date])))`,
so the calendar always starts exactly at the tenant's actual data, not a
hardcoded year) through the end of the current calendar year, at each
refresh. This makes `Dim_Date`'s M query depend on `Fact_GLTransactions`
evaluating first — expected and harmless (no circular reference, since
`Fact_GLTransactions` never references `Dim_Date`), but worth knowing if you
ever see Power Query's dependency graph and wonder why Dim_Date isn't a leaf
query anymore.

| Column | Type | Notes |
|---|---|---|
| Date | dateTime | Key. |
| Year | int64 | |
| Quarter | int64 | 1–4 |
| QuarterName | string | "Q1"–"Q4" |
| MonthNumber | int64 | 1–12, sort key for MonthName/MonthShort |
| MonthName | string | "January", sorted by MonthNumber |
| MonthShort | string | "Jan", sorted by MonthNumber |
| YearMonth | string | "2026-01" |
| YearQuarter | string | "2026-Q1" |
| Day | int64 | |
| DayOfWeekNumber | int64 | 1 (Mon) – 7 (Sun), sort key for DayName |
| DayName | string | sorted by DayOfWeekNumber |
| FiscalYear | int64 | Equal to Year (fiscal year = calendar year for this company). |

### Dim_ChartOfAccounts
Source: `WebServices` folder, entity `Chart_of_Accounts` (the legacy Chart of
Accounts page, published as a web service — its field names keep BC's
classic space-separated names even though the entity's own name has
underscores).

| Column | Type | BC source field | Notes |
|---|---|---|---|
| No | string | `No.` | Key. |
| Name | string | `Name` | |
| Account_Type | string | `Account Type` | Posting / Total / Begin-Total / End-Total / Heading. Filter to `Posting` for transactional rollups. |
| Income_Balance | string | `Income/Balance` | "Income Statement" or "Balance Sheet" — drives P&L vs. BS page filters. |
| Debit_Credit | string | `Debit/Credit` | |
| Account_Category | string | `Account Category` | Assets / Liabilities / Equity / Income / Cost of Goods Sold / Expense. Drives Revenue/COGS/OpEx measures directly. |
| Account_Subcategory | string | `Account Subcategory Descript.` | |
| Totaling | string | `Totaling` | |
| Indentation | int64 | `Indentation` | Preserves BC's COA hierarchy display order. |

### Dim_BusinessDimension1 / Dim_BusinessDimension2
Source: `v2.0` folder, entities `dimensions` and `dimensionValues` joined
together. `dimensionValues` only carries a `dimensionId` (GUID), not a plain
dimension code, so the query first finds the `dimensions` row whose `code`
equals `GlobalDimension1Code`/`GlobalDimension2Code`, takes its `id`, then
filters `dimensionValues` to that `dimensionId`. Generic table names because
dimension usage is company-specific — rename in Desktop to match what the
company actually tracks (e.g. `Dim_Department`, `Dim_Project`).

| Column | Type | Notes |
|---|---|---|
| Code | string | Key (from `dimensionValues.code`). Includes a synthetic blank-code row ("(No Department)" / "(No Customer Group)") so unassigned G/L entries still join cleanly. |
| Name | string | From `dimensionValues.displayName`. |
| Dimension_Code | string | Hidden; set to the `GlobalDimension1Code`/`GlobalDimension2Code` parameter value (every row in this table already belongs to that one dimension). |

Global Dimensions 1 and 2 are the only two dimensions BC flattens directly
onto G/L Entry / Cust. Ledger Entry / Vendor Ledger Entry rows. A third or
fourth company dimension needs its own bridge/merge (`dimensionSetLines` on
`v2.0`, or `DimensionSetEntries` under `WebServices`) if required.

### Dim_AccountCategory
Source: the BC **Advanced API browser** (not the curated entity list) —
`accountCategories` under the `microsoft/analytics/v1.0` publisher/group/
version path. Unrelated to any of the standard curated entities the other
tables use.

| Column | Type | Notes |
|---|---|---|
| id | string | Placeholder — verify against the tenant's actual schema. |
| code | string | Placeholder. |
| displayName | string | Placeholder. |

**Not verified against a live tenant.** This table's navigation path
(`Advanced → microsoft/analytics/v1.0 → accountCategories`) was confirmed
against a real Navigator, but its column schema wasn't — check Power Query
Editor after first refresh and correct the three columns above to match, then
add a relationship to `Dim_ChartOfAccounts[Account_Category]` on whichever
column turns out to be the real join key. Not yet wired into any measure or
relationship pending that verification.

### Dim_Customer
Source: `v2.0` folder, entity `customers` (Microsoft's standard, documented
API — not a web service).

| Column | Type | BC source field |
|---|---|---|
| No | string | `number` (key) |
| Name | string | `displayName` |
| Currency_Code | string | `currencyCode` (blank = local currency) |

Trimmed to the three fields with the highest confidence on this entity. If
your tenant's `customers` entity also exposes posting group, salesperson, or
address/country fields you want, add them the same way after checking the
exact field names in Power Query Editor's preview.

### Dim_Vendor
Source: `v2.0` folder, entity `vendors`.

| Column | Type | BC source field |
|---|---|---|
| No | string | `number` (key) |
| Name | string | `displayName` |
| Currency_Code | string | `currencyCode` |

---

## Fact tables

### Fact_GLTransactions
Source: `WebServices` folder, entity `G_LEntries` (posted General Ledger
entries, published as a legacy web service — field names keep BC's classic
space-separated names). Grain: one row per G/L Entry No. Filtered on
`RangeStart`/`RangeEnd` for incremental refresh.

| Column | Type | BC source field | Notes |
|---|---|---|---|
| Entry_No | int64 | `Entry No.` | Key. |
| G_L_Account_No | string | `G/L Account No.` | → Dim_ChartOfAccounts. |
| Posting_Date | dateTime | `Posting Date` | → Dim_Date. |
| Document_No | string | `Document No.` | |
| Document_Type | string | `Document Type` | |
| Description | string | `Description` | |
| Source_Code | string | `Source Code` | Which BC journal/process posted the entry. |
| Global_Dimension_1_Code | string | `Global Dimension 1 Code` | → Dim_BusinessDimension1. |
| Global_Dimension_2_Code | string | `Global Dimension 2 Code` | → Dim_BusinessDimension2. |
| Debit_Amount | double | `Debit Amount` | |
| Credit_Amount | double | `Credit Amount` | |
| Amount | double | `Amount` | Signed net (Debit − Credit); the column all P&L/BS measures sum. |
| User_ID | string | `User ID` | Hidden. |

### Fact_Budget
Source: `WebServices` folder, entity `G_LBudgetEntries`. Grain: one row per
budget entry.

| Column | Type | BC source field |
|---|---|---|
| Entry_No | int64 | `Entry No.` (key) |
| Budget_Name | string | `Budget Name` — a company may keep several named budgets/forecasts here |
| G_L_Account_No | string | `G/L Account No.` → Dim_ChartOfAccounts |
| Date | dateTime | `Date` → Dim_Date |
| Global_Dimension_1_Code / Global_Dimension_2_Code | string | → Dim_BusinessDimension1 / Dim_BusinessDimension2 |
| Amount | double | `Amount`, same sign convention as Fact_GLTransactions |

### Fact_CustLedgerEntries (AR)
Source: `WebServices` folder, entity `Cust_LedgerEntries`.

| Column | Type | BC source field | Notes |
|---|---|---|---|
| Entry_No | int64 | `Entry No.` | Key. |
| Customer_No | string | `Customer No.` | → Dim_Customer. |
| Posting_Date | dateTime | `Posting Date` | → Dim_Date. |
| Due_Date | dateTime | `Due Date` | Aging buckets key off this vs. today. |
| Document_Type / Document_No | string | | |
| Description | string | | |
| Currency_Code | string | | |
| Open | boolean | `Open` | TRUE while a balance remains outstanding; aging measures filter to `Open = TRUE`. |
| Amount | double | Original invoice/transaction amount (LCY). |
| Remaining_Amount | double | Outstanding balance (LCY) as of last refresh — what the aging buckets sum. |

### Fact_VendorLedgerEntries (AP)
Source: `WebServices` folder, entity `VendorLedgerEntries`. Same shape as
Fact_CustLedgerEntries, keyed by Vendor_No instead of Customer_No.

---

## Relationships (star schema, single-direction only)

| From (many) | To (one) |
|---|---|
| Fact_GLTransactions[Posting_Date] | Dim_Date[Date] |
| Fact_GLTransactions[G_L_Account_No] | Dim_ChartOfAccounts[No] |
| Fact_GLTransactions[Global_Dimension_1_Code] | Dim_BusinessDimension1[Code] |
| Fact_GLTransactions[Global_Dimension_2_Code] | Dim_BusinessDimension2[Code] |
| Fact_Budget[Date] | Dim_Date[Date] |
| Fact_Budget[G_L_Account_No] | Dim_ChartOfAccounts[No] |
| Fact_Budget[Global_Dimension_1_Code] | Dim_BusinessDimension1[Code] |
| Fact_Budget[Global_Dimension_2_Code] | Dim_BusinessDimension2[Code] |
| Fact_CustLedgerEntries[Customer_No] | Dim_Customer[No] |
| Fact_CustLedgerEntries[Posting_Date] | Dim_Date[Date] |
| Fact_VendorLedgerEntries[Vendor_No] | Dim_Vendor[No] |
| Fact_VendorLedgerEntries[Posting_Date] | Dim_Date[Date] |

No bidirectional relationships are used; each fact table has exactly one
active path to Dim_Date, so there is no ambiguity to resolve with a
weak/inactive relationship.

---

## Measures (`_Measures` table)

All monetary measures are formatted `$#,0.00;($#,0.00)`; percentages
`0.0%;-0.0%`. Time-intelligence measures rely on Dim_Date being marked as the
Date table — standard `TOTALYTD` / `TOTALQTD` / `TOTALMTD` /
`SAMEPERIODLASTYEAR` work with no fiscal-offset argument because fiscal year
= calendar year.

**P&L**
| Measure | DAX (summary) |
|---|---|
| Revenue | `-SUM(Amount)` where Account_Category = "Income" |
| COGS | `SUM(Amount)` where Account_Category = "Cost of Goods Sold" |
| Gross Profit | `Revenue − COGS` |
| Gross Margin % | `DIVIDE(Gross Profit, Revenue)` |
| Operating Expenses | `SUM(Amount)` where Account_Category = "Expense" |
| Operating Income | `Gross Profit − Operating Expenses` |
| Net Income | `Revenue − COGS − Operating Expenses` |

**Balance Sheet**
| Measure | DAX (summary) |
|---|---|
| Total Assets | `SUM(Amount)` where Account_Category = "Assets" |
| Total Liabilities | `-SUM(Amount)` where Account_Category = "Liabilities" |
| Total Equity | `-SUM(Amount)` where Account_Category = "Equity" |
| Balance Sheet Check | `Total Assets − Total Liabilities − Total Equity` (validation; non-zero before year-end close is expected — see the measure's comment) |

**Cash**
| Measure | Notes |
|---|---|
| Cash and Bank | Sums Amount for Assets whose Account_Subcategory contains "Cash" or "Bank" |
| Working Capital (Simplified) | `Total Assets − Total Liabilities`; a directional proxy only — see the measure's comment for why a true current/non-current split needs an added account mapping |

**Time intelligence** — full YTD/QTD/MTD/PY/YoY set for Revenue and Net
Income; YTD + PY/YoY% for Gross Profit, COGS, Operating Expenses, Operating
Income, and Gross Margin %. Same `TOTALYTD([Base], Dim_Date[Date])` /
`CALCULATE([Base], SAMEPERIODLASTYEAR(Dim_Date[Date]))` pattern extends
directly to any other base measure not already wrapped.

**Budget vs Actual**
| Measure | DAX (summary) |
|---|---|
| Actual Amount (IS) | `SUM(Amount)` where Income_Balance = "Income Statement" (native BC sign, not flipped, to match Budget Amount's convention) |
| Budget Amount | `SUM(Fact_Budget[Amount])` |
| Budget Variance | `Actual Amount (IS) − Budget Amount` |
| Budget Variance % | `DIVIDE(Budget Variance, ABS(Budget Amount))` |

**AR Aging** (all filter `Fact_CustLedgerEntries[Open] = TRUE`, compare
`Due_Date` to `TODAY()`): Total AR Outstanding, AR Current, AR 1-30 Days, AR 31-60
Days, AR 61-90 Days, AR 90+ Days.

**AP Aging**: same bucket set against `Fact_VendorLedgerEntries`.

**Executive KPIs**
| Measure | DAX (summary) | Notes |
|---|---|---|
| Transaction Count | `COUNTROWS(Fact_GLTransactions)` | Row count of posted G/L entries in the current filter context. |
| Total Debit | `SUM(Fact_GLTransactions[Debit_Amount])` | |
| Total Credit | `SUM(Fact_GLTransactions[Credit_Amount])` | |
| Net Amount | `SUM(Fact_GLTransactions[Amount])` | Same as Debit − Credit; used as a plain net-movement KPI on Transaction Detail. |

**Balance Sheet composition**
| Measure | DAX (summary) | Notes |
|---|---|---|
| BS Category Amount | `ABS(SUM(Amount))` where Income_Balance = "Balance Sheet" | Dynamic by whatever `Dim_ChartOfAccounts[Account_Category]` value is in context (Assets/Liabilities/Equity) — feeds the Balance Sheet composition donut. Absolute value only for a size-of-slice visual; use the signed Total Assets/Liabilities/Equity measures for anything that needs the real sign. |

**Aging buckets by chart** (`Dim_AgingBucket` — a small disconnected table, not related to any fact table)
| Column | Type | Notes |
|---|---|---|
| Bucket | string | "Current", "1-30 Days", "31-60 Days", "61-90 Days", "90+ Days" — a static `DATATABLE()`, not sourced from BC. |
| SortOrder | int64 | Hidden; sort-by column so the buckets chart in aging order, not alphabetically. |

| Measure | DAX (summary) |
|---|---|
| AR Aging Amount | `SWITCH(SELECTEDVALUE(Dim_AgingBucket[Bucket]), "Current", [AR Current], "1-30 Days", [AR 1-30 Days], … )` |
| AP Aging Amount | Same pattern against the AP bucket measures. |

These two dynamic measures only return a value when exactly one `Bucket` is in context (i.e. on an axis/legend built from `Dim_AgingBucket[Bucket]`) — they're not meant to be dropped on a card by themselves.

---

## Report pages

Every page uses a shared custom theme (`Financial Dashboard.Report/StaticResources/RegisteredResources/FinanceExecutiveTheme.json`, registered in `report.json`'s `themeCollection.customTheme`) for a consistent corporate palette (navy/blue/gold), card-style visuals with subtle borders and shadows, and bold Segoe UI Semibold titles/KPI values. See `docs/ReportPages.md` for the full per-page visual list.

| Page | Purpose | Key visuals |
|---|---|---|
| Executive Summary | KPI snapshot for leadership | 8 KPI cards (Revenue/Net Income/Gross Margin %/Cash, plus Revenue YoY %/Net Income YoY %/Total AR/Total AP), Revenue trend line chart (legend = Year for YoY), Revenue-by-Department donut, Year slicer |
| P&L Statement | Account-category rollup + trend | 5 KPI cards (Revenue/COGS/Gross Profit/OpEx/Net Income), table by Account_Category → Name, Revenue/COGS/Net Income trend line by month, Year/Month slicers |
| Balance Sheet | Assets/Liabilities/Equity rollup | 4 KPI cards (Total Assets/Liabilities/Equity, Balance Sheet Check), table by Account_Category → Subcategory → Name, Balance Sheet composition donut |
| Budget vs Actual | Variance by department/account | 4 KPI cards (Budget/Actual/Variance/Variance %), clustered column (Actual vs Budget by department), variance table by account |
| AR/AP Aging | Aging buckets | 4 KPI cards (Total AR/AP Outstanding, AR/AP Current), AR and AP aging-by-bucket column charts, bar chart AR by customer, table AP by vendor |
| Transaction Detail | Drill-through target | 4 KPI cards (Transaction Count, Total Debit/Credit, Net Amount), Year and Month slicers, full G/L entry table |

The seeded visuals cover each page's primary chart/table; additional cards,
slicers, and the cross-page drill-through wiring (right-click a summary
visual → *Drillthrough*) are quick to finish directly in Power BI Desktop —
see `docs/ReportPages.md` for the full recommended visual list per page.
