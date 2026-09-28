# Data Profiling Report

Before any modeling or DAX work began, every table was profiled with live DAX queries against the
actual data — not assumed from a schema description. This is the baseline every later phase built
on and validated against.

## Row counts

| Table | Rows |
|---|---|
| `fact_sales_normalized` | 1,000,000 |
| `dim_customers` | 100,000 |
| `dim_products` | 210 |
| `dim_stores` | 500 |
| `dim_salespersons` | 2,000 |
| `dim_campaigns` | 50 |
| `dim_dates` | 366 |

## Fact table integrity

- `sales_id` distinct count = 1,000,000 → **no duplicate transaction IDs**
- Null `customer_sk` / `product_sk` / `store_sk` / `salesperson_sk` / `campaign_sk` / `sales_date` → **zero, across all five**
- `sales_date` range: **Jan 2 → Dec 26, 2024** (partial-year edges — no Jan 1 or Dec 27–31 activity)
- `total_amount` range: **500 → 4,999.98**; negative sales = 0; zero-value sales = 0

## Referential integrity

Every foreign key in the fact table was checked against its dimension table using `EXCEPT()` /
`VALUES()` — zero orphans found on all five relationships (customer, product, store, salesperson,
campaign), zero orphans on `dim_stores[store_manager_sk] → dim_salespersons`, zero orphans on
`dim_campaigns[start_date_sk / end_date_sk] → dim_dates`.

## Dimension key uniqueness

`customer_sk`, `product_sk`, `store_sk`, `salesperson_sk`, `date_sk`, and `full_date` all have
distinct counts equal to their table's row count — clean primary keys, no duplicates at the
surrogate-key level.

**But surrogate keys aren't the whole story** — see `02-data-quality-findings.md` for the two
non-obvious findings this profiling pass led to: `product_name` and `store_name` are *not* unique
identifiers, despite every surrogate key being clean.

## Value ranges and categorical cardinality

| Field | Distinct values |
|---|---|
| `customer_segment` | 10 |
| `store_type` | 3 |
| `category` | 6 |
| `brand` | 25 |
| `salesperson_role` | 4 |
| `campaign_budget` | 105,815 → 987,420 (range) |

## Data-quality checklist result

| Check | Result |
|---|---|
| Duplicate `sales_id` | None found |
| Null foreign keys | None found |
| Orphan foreign keys (all paths) | None found |
| Negative/zero sales | None found |
| Dimension PK uniqueness | Confirmed unique |
| Date coverage gaps (Jan 1, Dec 27–31 missing) | Present — minor, informational only |

The underlying data was genuinely clean. The real issues found during this project weren't data
quality — they were **model wiring** decisions (a broken date relationship via Auto Date/Time, an
unused store-manager relationship, non-unique display names) that only a live, query-based audit
surfaces. A schema-only review would have missed every one of them.
