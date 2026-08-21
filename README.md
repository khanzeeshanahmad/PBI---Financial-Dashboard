# Power BI Financial Dashboard — Dynamics 365 Business Central

A Power BI Project (`.pbip`) that connects to **Dynamics 365 Business
Central (SaaS)** via the native Power BI connector and builds a star-schema
financial model + a 6-page report: Executive Summary, P&L Statement, Balance
Sheet, Budget vs Actual, AR/AP Aging, and Transaction Detail.

Fiscal year = calendar year (January–December) throughout, so all time
intelligence uses standard DAX (`TOTALYTD`, `SAMEPERIODLASTYEAR`, etc.) with
no fiscal-offset logic.

## Project layout

```
Financial Dashboard.pbip                    ← open this in Power BI Desktop
Financial Dashboard.SemanticModel/          ← TMDL model definition
  definition/
    expressions.tmdl                        ← BCEnvironment / BCCompanyName / dimension / incremental-refresh parameters
    tables/                                 ← Dim_Date, Dim_ChartOfAccounts, Dim_BusinessDimension1/2,
                                               Dim_Customer, Dim_Vendor, Fact_GLTransactions, Fact_Budget,
                                               Fact_CustLedgerEntries, Fact_VendorLedgerEntries, _Measures
                                               (each table's Power Query M source is in its `partition` block)
    relationships.tmdl
Financial Dashboard.Report/                 ← PBIR report definition (6 pages)
docs/
  DataDictionary.md                         ← every table, column, BC source field, and measure's DAX
  ReportPages.md                            ← what's seeded per page vs. recommended to finish in Desktop
```

## Prerequisites

- Power BI Desktop with **Power BI Project (.pbip) save** and **TMDL** in
  Preview features enabled (Options → Preview features) — needed to open
  this format at all.
- Access to the target Business Central SaaS tenant, environment, and
  company as a licensed user or via an Entra ID service account with
  Business Central access.
- A Power BI Pro/PPU license (or Premium capacity workspace) to publish and
  share.

## Authenticating through the connector (Azure AD / Entra)

1. Open `Financial Dashboard.pbip` in Power BI Desktop.
2. Go to **Transform data → Data source settings**, or just hit **Refresh** —
   Desktop will prompt for the Business Central connector's credentials the
   first time.
3. Choose **Organizational account** and sign in with the Entra ID (Azure
   AD) account that has access to the target BC tenant/environment/company.
   This is the same native "Dynamics 365 Business Central" connector you'd
   pick from Get Data, using OAuth against Entra — no API key, no raw OData
   endpoint/URL to configure.
4. Desktop caches the OAuth token for that data source; if you ever need to
   switch tenants, clear it via **File → Options and settings → Data source
   settings → Global permissions**.

## Setting environment / company (Sandbox vs Production)

All tables call `BusinessCentral.Contents()` and drill into
`Environment{[Name = BCEnvironment]}` → `Company{[Name = BCCompanyName]}`
using two model parameters instead of hardcoded literals, so **one place**
controls what every table pulls from:

1. In Desktop: **Home → Transform data → Manage parameters**.
2. Set `BCEnvironment` to the exact environment name shown in the BC admin
   center / connector Navigator (e.g. `Production` or `Sandbox`, or your
   tenant's actual sandbox name).
3. Set `BCCompanyName` to the exact company display name (e.g. `CRONUS USA,
   Inc.`).
4. If this company's Global Dimensions 1/2 aren't Department/Project, update
   `GlobalDimension1Code` / `GlobalDimension2Code` to match (check **Company
   Information** in BC for what's mapped to each Global Dimension slot).
5. Refresh. Because these are query parameters (not query text), you can
   also override them at the semantic model level in the Fabric/Power BI
   service (**Settings → Parameters**) per workspace, so a Dev workspace can
   point at Sandbox and a Prod workspace at Production without editing the
   .pbip at all.

See `docs/DataDictionary.md` for the full parameter list and every table's M
query / source BC entity.

## BC version compatibility

The Power BI connector's Navigator entity names can shift across Business
Central releases and localizations. This project was authored against a
current BC SaaS release (API `v2.0`-era). Flagged compatibility risks, also
called out inline in each table's `.tmdl` comment:

| Table | Entity name used | Known variant(s) |
|---|---|---|
| Dim_ChartOfAccounts | `Chart of Accounts` | `Account Category` / `Account Subcategory Descript.` may require a merge against a separate `G/L Account Category` table on older BC builds |
| Dim_Customer | `Customers` | Older exports: `Customer` (singular) |
| Dim_Vendor | `Vendors` | Older exports: `Vendor` (singular) |
| Fact_GLTransactions | `G/L Entries` | `General Ledger Entries`, or table name `G/L Entry` |
| Fact_Budget | `G/L Budget Entries` | — |
| Fact_CustLedgerEntries | `Cust. Ledger Entries` | — |
| Fact_VendorLedgerEntries | `Vendor Ledger Entries` | — |

If a query errors with "entity not found," open **Transform data → [table] →
Source** step, click the gear icon (or re-launch Get Data → Business
Central), and confirm the exact label your tenant's Navigator shows, then
update the literal in that one M step.

## Incremental refresh

`Fact_GLTransactions` (and `Fact_Budget`) already filter on the reserved
`RangeStart` / `RangeEnd` parameters. To finish enabling incremental refresh
after first publish:

1. In the Fabric/Power BI service workspace, or in Desktop before publishing:
   right-click **Fact_GLTransactions** in the Fields pane → **Incremental
   refresh**.
2. Suggested policy for posted G/L entries (immutable once posted, so change
   detection isn't needed): **Archive** 5 years, **Incrementally refresh**
   1 month, **Detect data changes**: off.
3. Apply the same policy to `Fact_Budget` if budget volume warrants it
   (usually not necessary at typical budget-entry volumes).

## Publishing and scheduled refresh

1. **File → Publish → Publish to Power BI** and choose the target workspace
   (Pro or Premium/Fabric capacity).
2. In the service, open the semantic model's **Settings**:
   - **Data source credentials**: sign in with the same (or a dedicated
     service) Entra ID account that has BC access — this is what scheduled
     refresh runs as.
   - **Parameters**: confirm/override `BCEnvironment`, `BCCompanyName`, and
     the dimension parameters for this workspace if it differs from what's
     in the .pbip (e.g. a Dev workspace pointed at Sandbox).
   - **Scheduled refresh**: enable, and set a cadence — daily is typical for
     GL-driven financials; increase frequency only if the business needs
     same-day figures and BC's own data volume/API limits support it.
3. If the workspace is on a Pro (non-Premium) capacity, an on-premises data
   gateway is **not** required for Business Central SaaS (it's a cloud
   connector) — only for gateway-bound sources, which this project doesn't
   use.
4. Re-share `docs/DataDictionary.md` with report consumers so measure
   definitions and BC field lineage are documented outside the model too.

## Extending the model

- **More dimensions**: BC only flattens Global Dimensions 1–2 onto ledger
  entries directly; a 3rd/4th dimension needs a merge against `Dimension Set
  Entry` filtered to the relevant `Dimension Set ID`s.
- **Multi-currency**: `Remaining Amount` on the AR/AP fact tables is LCY;
  add `Remaining Amt. (LCY)` vs. transaction-currency columns and a
  `Currency Exchange Rate` dimension if original-currency reporting is
  needed.
- **More measures / report visuals**: see `docs/DataDictionary.md` for the
  DAX pattern behind every existing measure, and `docs/ReportPages.md` for
  what's seeded per report page vs. quick to finish in Desktop (drillthrough
  wiring, bookmarks, extra cards/slicers).
