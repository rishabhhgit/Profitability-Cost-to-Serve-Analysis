# KPI Dictionary

Canonical definitions for every profitability metric in this project, with the
values produced by the current dataset (250,000 orders; 227,243 Completed;
10,000 customers in the master file, 9,998 of which have at least one
Completed order).

All figures below are for **Completed orders only**, which is the analytical
population used by `rpt.vw_Cost_to_Serve`, `rpt.vw_Customer_Profitability`
and `rpt.vw_Executive_KPIs`.

---

## 1. Revenue and product cost

| Metric | Definition (SQL) | Verified value |
|---|---|---|
| Gross Revenue | `SUM(Quantity * Unit_Price)` | — |
| Discount Amount | `SUM(Discount_Amount)` | — |
| **Net Revenue** | Gross Revenue − Discount Amount | **3,576,677,378** |
| **Product Cost (COGS)** | `SUM(Quantity * Products.Unit_Cost)` | **2,615,275,667** |
| **Gross Profit** | Net Revenue − Product Cost | **961,401,712** |
| Gross Margin | Gross Profit / Net Revenue | **26.88%** |

Implemented in `rpt.vw_Revenue_Analytics` (`SQL/08_Create_Reporting_Views.sql`).

## 2. Cost-to-Serve

Cost-to-Serve is the operational cost of serving Completed orders. It has four
components:

| Component | Source table | Verified value | Share of Cost-to-Serve |
|---|---|---|---|
| Customer Service | `core.Customer_Service` | 9,696,715 | **58.68%** |
| Shipping | `core.Shipping_Costs` | 4,847,989 | **29.34%** |
| Storage (allocated) | `core.Storage_Costs` | 1,199,899 | 7.26% |
| Returns | `core.Returns` | 779,325 | 4.72% |
| **Total Cost-to-Serve** | — | **16,523,929** | **100%** |
| Cost-to-Serve Ratio | Total Cost-to-Serve / Net Revenue | **0.46%** | — |

Customer Service + Shipping = **88.02%** of Total Cost-to-Serve.

Storage is *allocated*, not directly tracked per order: monthly storage cost is
divided across the units of the same product shipped from the same warehouse in
the same month (`rpt.vw_Storage_Cost_Allocation`).

## 3. Contribution profit — two definitions

The project originally shipped a metric named `Contribution_Profit` that does
**not** subtract Product Cost. It is retained unchanged in the views so the
existing Power BI report keeps working, but it must not be used as a headline
profitability KPI.

| Metric | Formula | Verified value | Margin |
|---|---|---|---|
| `Contribution_Profit` (**legacy**) | Net Revenue − Cost-to-Serve | 3,560,153,450 | **99.54%** |
| `True_Contribution_Profit` (**canonical**) | Gross Profit − Cost-to-Serve | **944,877,783** | **26.42%** |

The legacy figure implies a 99.54% margin, which is not credible for a
distribution business. The corrected metric — `True_Contribution_Profit`,
introduced during project review — nets COGS first and is the metric used for
every business conclusion in `README.md` and `Insights.pdf`.

Canonical objects:

- `rpt.vw_Cost_to_Serve.True_Contribution_Profit`
- `rpt.vw_Customer_Profitability.True_Contribution_Profit`
- `rpt.vw_Executive_KPIs.True_Contribution_Profit` / `True_Contribution_Margin`

## 4. Customer-level KPIs

| KPI | Definition | Verified value |
|---|---|---|
| Total Customers (analysis population) | Distinct customers with ≥1 Completed order | **9,998** |
| Customers in master file | Rows in `core.Customers` | 10,000 |
| Total Orders (Completed) | Distinct Completed orders | **227,243** (of 250,000) |
| **Loss-Making Customers** | `True_Contribution_Profit < 0` | **13** (all SMB) |
| Legacy loss-making customers | `Contribution_Profit < 0` | 1 |
| Profit concentration | Customers needed for 80% of true contribution profit | ~50% of the base (≈5,042 customers) |
| Return Rate | Return Quantity / Units Sold | 2.89% |

## 5. Segmentation and risk

`rpt.vw_Customer_Profitability_Segmentation` and `rpt.vw_Customer_Risk_Analysis`
classify customers on **True_Contribution_Profit**:

- `At-Risk Customer` / `Loss-Making Customer` — `True_Contribution_Profit < 0`
- `Strategic Customer` — revenue ≥ average revenue **and** true contribution ≥ average true contribution
- `High-Cost Customer` — Cost-to-Serve ratio above the customer-base average (within the segmentation view's mutually exclusive classes)
- `Profitable Customer` — positive true contribution profit

Expected distribution on the current dataset (9,998 customers):

| Segment | Customers |
|---|---|
| Profitable Customer | 4,754 |
| Strategic Customer | 3,229 |
| High-Cost Customer | 2,002 |
| At-Risk Customer | 13 |

`rpt.vw_Customer_Risk_Analysis` applies the cost test as a **separate, independent
axis** (`Cost_Efficiency_Status`: `Cost_to_Serve_Ratio > AVG(ratio)`), so its counts
differ from the segmentation classes above:

| Cost Efficiency Status | Customers |
|---|---|
| High Cost-to-Serve | **2,163** |
| Efficient Cost-to-Serve | 7,835 |

`rpt.vw_Customer_Pareto` ranks customers on `True_Contribution_Profit`:
top 80% of profit = ≈5,042 customers (50.4% of the base), next 15% = 2,790,
bottom 5% = 2,166.

## 6. How these numbers were verified

Every figure above was reproduced independently of SQL Server by re-implementing
the view logic in Python directly over the raw CSVs in
`Dataset/Enterprise_CostToServe_Dataset.zip` (storage allocation, completed-order
filters, COGS join and customer roll-ups included). The reproduction also matches
the dashboard exports in `dashboard/` (3.58bn revenue, 227K orders,
16.52M Cost-to-Serve, 58.68% / 29.34% / 7.26% / 4.72% cost split).
