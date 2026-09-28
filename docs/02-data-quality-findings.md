# Data Quality & Modeling Findings

Every finding below was caught by running live DAX queries against the actual 1,000,000-row fact
table and its dimensions — not assumed, not inferred from column names. This document exists so
the *evidence* behind each fix in the semantic model is traceable, not just the fix itself.

## Summary table

| # | Finding | Evidence | Fix | Risk if missed |
|---|---|---|---|---|
| 1 | `product_name` is not unique | 186 distinct names across 210 products | Always pair `product_name` with `brand` in any visual | Two unrelated SKUs silently merge in a "Top Products" table |
| 2 | `store_name` is not a unique identifier | 50 distinct names across 500 stores (10 branches/name avg.) | `Top Store`/`Bottom Store` measures return `name — location (store_id)` | A store ranking table looks like duplicate rows |
| 3 | Relationship cycle on `store_manager_sk` | `dim_stores → dim_salespersons` would create a second active path to the fact table (alongside the existing direct salesperson relationship) | Built the relationship **inactive**, used `USERELATIONSHIP()` for the one measure that needs it | Power BI would reject an active relationship, or silently produce wrong numbers if forced |
| 4 | Auto Date/Time silently owned the date relationship | Fact table's `sales_date` was relating to a hidden auto-generated date table, not the real `dim_dates` | Disabled Auto Date/Time in Desktop settings; added a `sales_date_only` calculated column and related it to `dim_dates[full_date]` directly | Every year-over-year/date-slicer visual filters against the wrong calendar |
| 5 | `campaign_budget` ≠ confirmed marketing spend | Every transaction is already tagged to *some* campaign — no non-campaign baseline exists to isolate incremental lift | Measure named `Sales to Budget Ratio`, explicitly **not** `ROI`, with a warning in its own description field | A budget-efficiency number gets presented to leadership as a return-on-investment figure it was never validated to be |

## Detail: finding 1 — Non-unique product names

```dax
EVALUATE ROW(
    "DistinctProductNames", DISTINCTCOUNT(dim_products[product_name]),
    "TotalProducts", COUNTROWS(dim_products)
)
```
**Result:** `186 | 210`

This was caught while validating the `Top Product` measure — the initial top-5 check (grouped by
name) didn't match the true top-5 by SKU, which is what surfaced the collision. The measures were
then rebuilt to rank at the correct grain (`dim_products`, one row per SKU) and any display
measure that returns a name now concatenates a disambiguating attribute.

## Detail: finding 3 — The relationship cycle

The brief called for `dim_stores[store_manager_sk] → dim_salespersons[salesperson_sk]` as a normal
active relationship. Mapping the existing relationship graph first:

```
fact_sales_normalized[salesperson_sk] → dim_salespersons[salesperson_sk]   (active)
fact_sales_normalized[store_sk]       → dim_stores[store_sk]              (active)
dim_stores[store_manager_sk]          → dim_salespersons[salesperson_sk]  (proposed)
```

Adding the third edge as active creates two simultaneous filter paths between
`dim_salespersons` and `fact_sales_normalized` — direct, and via `dim_stores`. Tabular models do
not allow two active paths between the same pair of tables. The relationship was built **inactive**
instead, and a single measure uses `USERELATIONSHIP()` to activate it only when needed:

```dax
Sales by Store Manager =
CALCULATE(
    [Total Sales],
    USERELATIONSHIP(dim_stores[store_manager_sk], dim_salespersons[salesperson_sk])
)
```

**Validation:** the top 5 results from this measure were cross-checked against the top 5 stores
from an entirely independent measure (`Store Rank`). They matched exactly — same five values, same
order — and all five people holding those "top managed store" positions carry the job title
`Manager` in `dim_salespersons[salesperson_role]`. That combination (numeric match + logical
consistency of role) is what makes this a validated fix, not just a formula that ran without
erroring.

## Detail: finding 5 — Why this isn't called ROI

```
True ROI = (Revenue − Marketing Spend) / Marketing Spend
```

Two conditions must hold for that formula to be meaningful, and neither did here:

1. `campaign_budget` must represent actual, attributable marketing spend — never confirmed against
   source data.
2. There must be a non-campaign baseline period to measure incremental lift against — but every
   single one of the 1,000,000 transactions is already tagged to one of the 50 campaigns. There is
   no "what would have happened anyway" control group in this dataset.

The measure is named `Sales to Budget Ratio` and its `description` property (visible in the model
itself, not just this document) explicitly warns against relabeling it as ROI.

## What this means for anyone extending this model

If you add real marketing spend data and a non-campaign baseline period, the `Sales to Budget
Ratio` measure can be safely evolved into a true ROI calculation — the underlying `Campaign Budget`
and `Total Sales` building blocks are already correct. Everything else above is intentionally
conservative: each fix trades a small amount of modeling elegance for a much larger reduction in
the risk of a wrong number reaching a stakeholder.
