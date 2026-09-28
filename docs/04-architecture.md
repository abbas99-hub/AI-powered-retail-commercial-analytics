# Data Model Architecture

## Star schema

```
                         dim_dates (366 days, 2024)
                              │  (sales_date_only → full_date)
                              │
dim_customers ──────┐         │         ┌────────── dim_products
 (100,000 rows)      │         │         │           (210 rows)
                     ▼         ▼         ▼
              ┌──────────────────────────────────┐
              │      fact_sales_normalized        │
              │         (1,000,000 rows)          │
              └──────────────────────────────────┘
                     ▲                   ▲
                     │                   │
              dim_stores ──┐       dim_salespersons
              (500 rows)   │        (2,000 rows)
                            │              ▲
                            │  (inactive,   │
                            │  USERELATIONSHIP)
                            └───────────────┘
                     dim_campaigns (50 rows)
                     also → dim_dates via start_date_sk / end_date_sk
                     (both inactive — same cycle constraint as above)
```

## Tables

| Table | Rows | Grain |
|---|---|---|
| `fact_sales_normalized` | 1,000,000 | One row per transaction |
| `dim_customers` | 100,000 | One row per customer |
| `dim_products` | 210 | One row per SKU |
| `dim_stores` | 500 | One row per physical location |
| `dim_salespersons` | 2,000 | One row per staff member |
| `dim_campaigns` | 50 | One row per campaign |
| `dim_dates` | 366 | One row per calendar day, 2024 |
| `_Measures` | 0 (measure-only table) | Houses all 76 measures |

## Relationships

| From | To | Cardinality | Active? | Why |
|---|---|---|---|---|
| `fact_sales_normalized[customer_sk]` | `dim_customers[customer_sk]` | Many-to-1 | Active | Standard fact→dim |
| `fact_sales_normalized[product_sk]` | `dim_products[product_sk]` | Many-to-1 | Active | Standard fact→dim |
| `fact_sales_normalized[store_sk]` | `dim_stores[store_sk]` | Many-to-1 | Active | Standard fact→dim |
| `fact_sales_normalized[salesperson_sk]` | `dim_salespersons[salesperson_sk]` | Many-to-1 | Active | Standard fact→dim |
| `fact_sales_normalized[campaign_sk]` | `dim_campaigns[campaign_sk]` | Many-to-1 | Active | Standard fact→dim |
| `fact_sales_normalized[sales_date_only]` | `dim_dates[full_date]` | Many-to-1 | Active | Calculated column truncates the fact table's datetime to a pure date for a clean join key |
| `dim_stores[store_manager_sk]` | `dim_salespersons[salesperson_sk]` | Many-to-1 | **Inactive** | Would create a relationship cycle if active (see `02-data-quality-findings.md`) |
| `dim_campaigns[start_date_sk]` | `dim_dates[date_sk]` | Many-to-1 | **Inactive** | Same cycle constraint — `dim_campaigns` already reaches `dim_dates` via the fact table |
| `dim_campaigns[end_date_sk]` | `dim_dates[date_sk]` | Many-to-1 | **Inactive** | Same reason |

## Why `sales_date_only` exists

`fact_sales_normalized[sales_date]` is a full datetime (includes time-of-day, e.g.
`2024-01-02 12:13:14 AM`). `dim_dates[full_date]` is a pure date. Relating a datetime directly to
a date column doesn't produce a clean one-to-many join. The fix is a calculated column:

```dax
sales_date_only = DATEVALUE(fact_sales_normalized[sales_date])
```

This keeps the original timestamp intact (for any future time-of-day analysis) while giving the
model a clean join key. Validated by confirming zero unmatched fact rows after the relationship
was built — see `02-data-quality-findings.md`.

## Date table enrichment

`dim_dates` originally had only raw numeric columns (`year`, `month`, `day`, `weekday`, `quarter`).
Seven calculated columns were added for display and correct sorting:

| Column | Purpose | Sort-by |
|---|---|---|
| `month_name` | "January" | `month` |
| `month_short_name` | "Jan" | `month` |
| `weekday_name` | "Monday" (verified: `weekday`=1 is Monday, ISO convention) | `weekday` |
| `is_weekend` | Boolean, True for Sat/Sun | — |
| `quarter_name` | "Q1" | — |
| `year_month` | "Jan 2024" | `year_month_sort` (hidden) |
| `year_month_sort` | `202401` (hidden) | — |

`dim_dates` is marked as the model's official **Date Table** (full_date as the key column) —
required for the `DATESMTD`/`DATESQTD`/`DATESYTD`/`DATEADD` time-intelligence functions to behave
correctly.

## Fiscal calendar — intentionally not built

The model does not include fiscal year/quarter/month columns. No fiscal-year-start month was
confirmed with the business, and fabricating one would silently produce wrong numbers on any
fiscal report. If needed, this is a one-pass addition once the fiscal start month is known.
