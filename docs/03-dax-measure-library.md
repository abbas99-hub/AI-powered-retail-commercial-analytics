# DAX Measure Library — 76 Measures, Fully Validated

Every measure below was pulled directly from the live semantic model (not written from
memory into this document) and independently validated against real data before being
considered complete — see `01-data-profiling.md` and `02-data-quality-findings.md` for the
evidence behind the ones marked with a note.

## Folder summary

| Folder | Count | Purpose |
|---|---|---|
| 01 Sales | 7 | Core revenue and transaction-level KPIs |
| 02 Customers | 11 | Customer base size, activity, retention, and segment-driven sales |
| 03 Products | 7 | Catalog coverage, ranking, and contribution analysis |
| 04 Stores | 7 | Network coverage, ranking, and contribution analysis |
| 05 Salespersons | 8 | Staff performance, ranking, and the store-manager relationship |
| 06 Campaigns | 7 | Campaign performance and budget efficiency (not ROI) |
| 07 Time Intelligence | 13 | MTD/QTD/YTD, period-over-period growth, rolling windows |
| 08 Advanced Analytics | 10 | Concentration, Pareto analysis, volatility, best/worst periods |
| 09 Data Quality | 6 | Live referential-integrity tripwires for an ongoing validation page |
| **Total** | **76** | |

## 01 Sales

| Measure | Description |
|---|---|
| `Total Sales` | Sum of all transaction values |
| `Total Transactions` | Row count of the fact table at transaction grain |
| `Distinct Transactions` | Distinct count of sales_id; data-quality safeguard - should equal Total Transactions at current 1:1 grain |
| `Average Transaction Value` | Average value per transaction |
| `Average Sales Per Customer` | Total sales divided by distinct customers present in filter context |
| `Average Sales Per Store` | Total sales divided by distinct stores present in filter context |
| `Average Sales Per Product` | Total sales divided by distinct products present in filter context |

## 02 Customers

| Measure | Description |
|---|---|
| `Total Customers` | Row count of dim_customers - all registered customers regardless of purchase activity |
| `Active Customers` | Distinct customers transacting within the CURRENT filter context (e.g. respects a date slicer) |
| `Customers With Purchases` | Distinct customers with at least one purchase EVER (ignores date filter, still respects other filters like segment/store) |
| `Customers Without Purchases` | Registered customers who have never made a purchase |
| `New Customers` | Proxy metric: customers whose FIRST-EVER transaction date falls within the current period filter. No account-signup date exists in the data, so this uses first purchase as the acquisition proxy. |
| `Repeat Customers` | Distinct customers with more than one transaction within the current filter context |
| `Repeat Purchase Rate` | Share of purchasing customers who bought more than once in context |
| `High Value Customer Sales` | Total sales attributable to customers in the 'High Value' segment (existing segment label in dim_customers) |
| `Churn Risk Customer Sales` | Total sales attributable to customers in the 'Churn Risk' segment (existing segment label in dim_customers) |
| `Customer Rank` | Dense rank of the current customer by Total Sales, highest = 1 |
| `Top Customer` | Top-spending customer's full name with customer_id appended for guaranteed uniqueness |

## 03 Products

| Measure | Description |
|---|---|
| `Total Products` | Row count of dim_products - full catalog regardless of sales activity |
| `Products Sold` | Distinct products with at least one sale EVER (ignores date filter, respects other filters) |
| `Products Never Sold` | Catalog products with zero sales in the entire dataset |
| `Product Rank` | Dense rank of the current product by Total Sales, highest = 1. Evaluated over ALL products regardless of visual-level filters. |
| `Top Product` | Name of the single highest-selling product in the current context |
| `Bottom Product` | Name of the lowest-selling product in the current context, excluding never-sold products |
| `Product Contribution %` | This product's/brand's/category's share of TOTAL company sales, regardless of grain shown in the visual - one generic reusable measure at every grain |

## 04 Stores

| Measure | Description |
|---|---|
| `Total Stores` | Row count of dim_stores - full store network regardless of activity |
| `Active Stores` | Distinct stores with at least one sale EVER |
| `Stores With Sales` | Distinct stores transacting within the CURRENT filter context |
| `Store Rank` | Dense rank of the current store by Total Sales, highest = 1, evaluated catalog-wide |
| `Top Store` | Name of the single highest-selling store, disambiguated with location and ID since store_name alone is shared by ~10 branches on average |
| `Bottom Store` | Name of the lowest-selling store (excluding zero-sales stores), disambiguated with location and ID |
| `Store Contribution %` | This store's share of TOTAL company sales, regardless of grain shown in the visual |

## 05 Salespersons

| Measure | Description |
|---|---|
| `Total Salespersons` | Row count of dim_salespersons - full staff roster regardless of activity |
| `Active Salespersons` | Distinct salespersons with at least one sale EVER |
| `Salesperson Rank` | Dense rank of the current salesperson by Total Sales, highest = 1 |
| `Top Salesperson` | Name of the top-selling salesperson, with ID appended since salesperson_name has 23 collisions across 2,000 staff |
| `Bottom Salesperson` | Name of the lowest-selling salesperson (excluding zero-sales staff), with ID appended for disambiguation |
| `Salesperson Contribution %` | This salesperson's/role's share of TOTAL company sales |
| `Sales by Store Manager` | Sales attributable to the stores each person MANAGES (not sold personally). Uses the inactive dim_stores->dim_salespersons relationship via USERELATIONSHIP. |
| `Average Sales Per Salesperson` | Average sales per salesperson in current filter context |

## 06 Campaigns

| Measure | Description |
|---|---|
| `Campaign Budget` | Total budget across campaigns in current filter context |
| `Sales to Budget Ratio` | Sales-to-budget ratio. EXPLICITLY NOT TRUE ROI - campaign_budget is not confirmed as actual attributable marketing spend, and no non-campaign baseline exists to isolate incremental lift. |
| `Campaign Rank` | Dense rank of the current campaign by Total Sales, highest = 1 |
| `Top Campaign` | Name of the highest-selling campaign in the current context |
| `Bottom Campaign` | Name of the lowest-selling campaign (excluding zero-sales campaigns) |
| `Campaign Contribution %` | This campaign's share of TOTAL company sales |
| `Total Campaigns` | Total number of campaigns in the catalog |

## 07 Time Intelligence

| Measure | Description |
|---|---|
| `Sales MTD` | Month-to-date sales as of the latest date in current filter context |
| `Sales QTD` | Quarter-to-date sales as of the latest date in current filter context |
| `Sales YTD` | Year-to-date sales as of the latest date in current filter context |
| `Previous Month Sales` | Sales for the same period, shifted back one month |
| `Previous Quarter Sales` | Sales for the same period, shifted back one quarter |
| `Previous Year Sales` | Sales for the same period, shifted back one year. Returns BLANK on current data - dataset covers only 2024. |
| `MoM Growth %` | Month-over-month growth rate |
| `QoQ Growth %` | Quarter-over-quarter growth rate |
| `YoY Growth %` | Year-over-year growth rate. BLANK on current data - see Previous Year Sales. |
| `YTD Growth %` | Year-to-date growth rate vs. the same YTD period last year. BLANK on current data. |
| `Running Total Sales` | Cumulative sales from the start of available data through the latest date in current context |
| `Rolling 7 Day Sales` | Trailing 7-day sales ending on the latest date in current context |
| `Rolling 30 Day Sales` | Trailing 30-day sales ending on the latest date in current context |

## 08 Advanced Analytics

| Measure | Description |
|---|---|
| `Top 10 Product Contribution %` | Share of total sales from the top 10 products by sales |
| `Top 10 Store Contribution %` | Share of total sales from the top 10 stores by sales |
| `Top 10 Customer Contribution %` | Share of total sales from the top 10 customers by spend |
| `Pareto Cumulative %` | Cumulative % of total sales contributed by products up to and including the current product's rank - draws the classic Pareto (80/20) curve |
| `Average Daily Sales` | Total sales divided by the number of calendar days in the current filter context |
| `Sales Volatility` | Population standard deviation of day-level sales within current context |
| `Best Sales Day` | Calendar date with the single highest sales total in current context |
| `Worst Sales Day` | Calendar date with the lowest non-zero sales total in current context |
| `Best Month` | Calendar month (Year-Month label) with the highest sales total |
| `Worst Month` | Calendar month (Year-Month label) with the lowest non-zero sales total |

## 09 Data Quality

| Measure | Description |
|---|---|
| `Orphan Customer Keys` | Fact rows whose customer_sk has no match in dim_customers. Should always be 0. |
| `Orphan Product Keys` | Fact rows whose product_sk has no match in dim_products. Should always be 0. |
| `Orphan Store Keys` | Fact rows whose store_sk has no match in dim_stores. Should always be 0. |
| `Orphan Salesperson Keys` | Fact rows whose salesperson_sk has no match in dim_salespersons. Should always be 0. |
| `Orphan Campaign Keys` | Fact rows whose campaign_sk has no match in dim_campaigns. Should always be 0. |
| `Duplicate Transaction Count` | Difference between Total Transactions and Distinct Transactions. Should always be 0. |
