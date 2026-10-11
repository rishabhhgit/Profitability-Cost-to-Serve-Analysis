# Power BI Documentation

Report file: `Power bi/Enterprise_Profitability_Final.pbix`

Pages: Executive Overview, Customer Profitability, Cost-to-Serve,
Customer Risk & Profit Concentration, Customer Deep Dive, Tooltip.

The report is bound to the SQL reporting layer (`rpt` schema). It must be
refreshed against a database built by running `SQL/01` … `SQL/10` in order.

---

## Status of the shipped screenshots vs. the `.pbix`

The PNG exports in `dashboard/` have been **regenerated from the corrected
reporting-layer output** (the same logic as `SQL/08` … `SQL/10`, independently
re-verified in Python against the raw CSVs). They show the corrected metrics:
26.4% True Contribution Margin, 13 At-Risk customers, 4,754 / 3,229 / 2,002 /
13 segment split. The original pre-correction exports are preserved in
`dashboard/legacy/` for comparison.

The `.pbix`'s data model still carries the **legacy** contribution metric. One
report-level bug *has* been fixed in the shipped file (the Risk-page card
below); everything else requires opening the file in Power BI Desktop and
refreshing it against the SQL reporting layer, because the true-contribution
inputs (`True_*` columns) do not exist in the model until then.

### What the `.pbix` currently displays

| Visual | Currently bound to | Shows | Should be |
|---|---|---|---|
| Contribution Margin % card | `Contribution Margin %` measure | 99.54% | **26.4%** (after SQL refresh + re-point) |
| Contribution Profit card | `Contribution Profit` measure | 3.56bn | **0.94bn** (after SQL refresh + re-point) |
| At-Risk / Loss-Making customer count | `Profitability_Status` | 1 | **13** (after the SQL refresh below) |
| Profitability segment bars | `Profitability_Segment` | 4,632 / 3,387 / 1,978 / 1 | 4,754 / 3,229 / 2,002 / **13** |
| Risk page "High Cost-to-Serve" card | `Cost_Efficiency_Status = 'High Cost-to-Serve'` | **2,163** (fixed in the shipped file) | 2,163 |
| Deep Dive detail table | customer-level measures | correct — see note below | no change |

Revenue (3.58bn), orders (227K), customers (10K), Pareto (5.042K) and
Cost-to-Serve driver cards are unaffected — they never depended on the
corrected metric.

Note on the Deep Dive figure: the detail table renders numbers with the
**Indian digit groupings** used by the report's locale, so customer 4960's
revenue appears as `18,36,805.19` — that is **1,836,805.19**, exactly the
verified value (87 completed orders, $21,113 per order), and its Contribution
Profit column `18,32,026.23` is 1,832,026.23. The table and the KPI cards on
the same page agree; there is nothing to fix here.

Note on the High Cost-to-Serve card: the card's *title* said
"High Cost-to-Serve" but its query filtered `Cost_Efficiency_Status =
'Efficient Cost-to-Serve'`, so it displayed the complementary 7,835. The
model column itself was always correct (2,163 High / 7,835 Efficient, matching
`SQL/08_Create_Reporting_Views.sql`). The filter in `Report/Layout` has been
patched in the shipped `.pbix`, so the card now reads **2,163** without any
refresh.

### Inspecting the shipped model without Power BI Desktop

The `DataModel` part is an XPress9-compressed Analysis Services backup. It was
decompressed and read with the `pbixray` Python library (read-only), which
confirms:

- **17 measures**, the key one being the legacy definition
  `Contribution Profit = [Total Revenue] - [Total Cost-to-Serve]` — Product
  Cost is never subtracted.
- 5 `rpt` view tables + a `Measurs` helper table; **no Product Cost / COGS
  column exists anywhere in the model**, so the true 26.4% margin cannot be
  recomputed from the shipped model — the COGS data has to come in with the
  SQL refresh.
- Relationships: `rpt vw_Cost_to_Serve` → `rpt vw_Customer_Profitability`
  (M:1 on `Customer_ID`), and both customer-level views → `rpt vw_Customer_Pareto`
  (1:1, bi-directional).

The model can therefore be *read* outside Desktop, but it should only be
*edited* in Desktop (measure edits must be processed by the engine).

### Measure / field names found in `Report/Layout`

These are the exact strings the report's visuals bind to, counted by
occurrence in the `Report/Layout` JSON:

| Reference in Layout | Occurrences | Action |
|---|---|---|
| `Contribution Margin %` | 37 | re-point to `True_Contribution_Margin_Pct` |
| `Contribution Profit` | 34 | re-point to `True_Contribution_Profit` |
| `Profitability_Status` | 15 | refresh only (SQL now computes it from the corrected metric) |
| `Pareto_Segment` | bound | refresh only |
| `Profit_Rank` | bound | refresh only (ranked on `True_Contribution_Profit`) |
| `Cumulative_Profit_Pct` | bound | refresh only |
| `Contribution_Margin_Pct` | bound | swap to `True_Contribution_Margin_Pct` |
| `Cost_to_Serve_Ratio` | bound | no change (never affected) |

71 of the report's field references point at the legacy contribution metric.

## Steps to correct the report

1. Run `SQL/01` … `SQL/10` against SQL Server so the `rpt` views include the
   `True_Contribution_Profit` / `True_Contribution_Margin_Pct` columns added by
   the project review (`SQL/08_Create_Reporting_Views.sql`,
   `SQL/09_Executive_KPIs.sql`).
2. Open the `.pbix` in Power BI Desktop → **Transform data** and refresh.
3. On the Executive Overview page, re-point the two profitability cards:
   - `Contribution Margin %` → `True_Contribution_Margin` (format: percentage)
   - `Contribution Profit` → `True_Contribution_Profit` (format: currency, millions)
4. Optionally rename the card titles to **True Contribution Margin %** and
   **True Contribution Profit** so the definition is unambiguous.
5. On Customer Profitability, Customer Risk & Profit Concentration and
   Customer Deep Dive pages, swap any remaining `Contribution_Profit` /
   `Contribution_Margin_Pct` fields for the `True_*` equivalents. The
   `Profitability_Status`, `Profitability_Segment`, `Pareto_Segment` and
   `Cumulative_Profit_Pct` fields need no field swap — they are now computed
   from the corrected metric in SQL — only a refresh is required.
   (The Risk-page "High Cost-to-Serve" card already points at the right value
   — its filter was corrected in the shipped file.)
6. Refresh all pages, then re-export the screenshots into `dashboard/`.

## Refresh notes

- The SQL views deliberately keep the legacy `Contribution_Profit` /
  `Contribution_Margin_Pct` columns so nothing in the existing model breaks;
  they are no longer used for any classification or ranking.
- `rpt.vw_Customer_Cost_to_Serve` is deprecated (it omits Storage cost and is
  not restricted to Completed orders). Do not build new visuals on it — use
  `rpt.vw_Customer_Profitability`.
- Storage cost in the model is an *allocation* (units-based), so Cost-to-Serve
  by customer is comparable across customers but is not an invoice-level cost.
- The regenerated PNGs in `dashboard/` are rendered directly from the
  corrected reporting-layer output (verified against the raw CSVs) — they are
  the authoritative previews until the `.pbix` is refreshed and re-exported.
