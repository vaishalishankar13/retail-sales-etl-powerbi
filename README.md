# Retail Sales ETL Pipeline and Analytics Dashboard
**Python | Pandas | MySQL | Power BI**

An end-to-end data project: it cleans 25,000 raw retail transaction rows, loads them into a MySQL star schema safely and repeatably, proves the load is complete with a source-to-warehouse reconciliation, and serves the data in a Power BI dashboard.

PowerBI dashboard- (docs/img/dashboard.png)

## Results at a glance

| Metric | Value |
|---|---|
| Raw rows read | 25,000 |
| Rows loaded | 24,156 |
| Rows rejected | 844 (505 negative discount/tax, 339 exact duplicates) |
| Suspect rows loaded and flagged (`amount_paid` does not match formula) | 534 |
| Net revenue loaded | 14,582,770.99 |
| Net revenue rejected | 447,028.09 (3.0% of raw) |
| Reconciliation checks | 4 of 4 pass (row counts and revenue, source vs warehouse) |
| Orders / customers / average order value | 14,015 / 5,000 / 1,040.51 |

## Tech stack
- **Python** (Pandas, mysql-connector-python, logging) for extract, transform and load
- **MySQL 8** (InnoDB) for the warehouse: staging table, star schema, foreign keys, indexes, generated column, view, window functions
- **Power BI** for the data model, DAX measures and dashboard
- **Git / GitHub** for version control


## Data model (star schema)

ER diagram- (docs/img/ER_diagram.png)

- **Fact table `fact_retail_sales`:** one row per order line. Primary key is `(order_id, order_line_item)`.
- **Dimensions:** `dim_store`, `dim_product`, `dim_customer`, linked to the fact table with foreign keys.
- **`net_revenue`** is a generated column: `quantity * unit_price - store_discount - payment_gateway_discount`. Every report therefore uses the same revenue definition.
- **Indexes** on order date, store, product and customer. 
- **`stg_retail_sales`** is a staging table, emptied on every run.

## What the pipeline does

**Extract**
- Reads every column as text, so Pandas cannot guess types and one bad cell cannot crash the run.
- Handles the Excel BOM, malformed lines (reported, not silently dropped) and header spacing/case.
- Fails fast if an expected column is missing.

**Transform**

| Issue | Handling |
|---|---|
| Whitespace, mixed case, `null` / `N/A` / blank text | Standardised, missing values become real NULLs |
| Malformed emails | Set to NULL, row kept |
| Mixed, invalid or future dates | Converted to NULL, row kept, count logged |
| Negative quantity, negative discount or tax, missing keys, oversize values | **Rejected** to `rejected_rows.csv` with a reason |
| Duplicate order lines | One row per `(order_id, order_line_item)`; last one kept |
| Missing discount or tax | Treated as 0 |
| `amount_paid` differs from `qty x price - discounts + tax` | Loaded but **flagged** in `suspect_rows.csv` |

**Load**
1. Create the database and tables if missing (safe to re-run).
2. Bulk insert into staging in batches of 10,000 rows (`executemany`).
3. Upsert dimensions first, then the fact table (`INSERT ... ON DUPLICATE KEY UPDATE`), so foreign keys are always satisfied and re-running the same file creates no duplicates.
4. Run the reconciliation audit **before** committing. Any failure rolls the whole load back, and the warehouse stays exactly as it was.

**Reconciliation audit**
- `raw rows = loaded rows + rejected rows`
- `loaded rows = rows found in the warehouse`
- the same two checks on net revenue (to the cent)
- plus a data-quality snapshot: NULL dates, NULL emails, date range

## Dashboard (Power BI)

The model reads the fact and dimension tables from MySQL, with a Date table related to `order_date`.

- KPI cards: Net Revenue, Orders, Average Order Value
- Monthly net revenue trend
- Net revenue by category and top 5 cities
- Year slicer and a key-insights text box

```
Net Revenue         = SUM(fact_retail_sales[net_revenue])
Orders              = DISTINCTCOUNT(fact_retail_sales[order_id])
Average Order Value = DIVIDE([Net Revenue], [Orders])
```

Orders are counted as distinct `order_id` because one order can have several lines. Totals in Power BI were checked against the pipeline's audit output.

## Key insights

- **Electronics generates 78.0% of net revenue** (11.38M of 14.58M), followed by Home (12.0%), Apparel (7.9%) and Food (2.1%). The business depends heavily on one category.
- **Revenue is flat year over year.** 2025 net revenue (2.72M) is only 0.6% above 2024 (2.71M) and 2.0% above 2021 (2.67M). Growth would have to come from new customers, higher order value or new stores.
- **Top cities are closely matched.** Denver (1.90M), Boston, Austin, Seattle and San Francisco (1.82M) are within about 4.5% of each other, so no single city is a concentration risk.
- **Discounting is consistent.** The highest-discount stores run at only 5.1 to 5.4% of gross sales.
- **Data quality cost:** cleaning removed 3.0% of raw revenue. The 505 rows with negative discount or tax should be reviewed with the data owner, since they may be legitimate refunds.



## Design decisions
- **Read as text, then convert:** one malformed value becomes a NULL instead of stopping the pipeline.
- **Staging table and a single upsert:** faster than row by row, and safe to re-run.
- **Reject with a reason:** every removed row can be traced and explained.
- **Flag, don't delete, suspicious amounts:** a wrong total may still be a real sale.
- **One transaction, audit before commit:** the warehouse never ends up half loaded.

## Assumptions and limitations
- For duplicate keys, the last row is kept (assumes the latest row is the correction).
- Negative quantities, discounts and tax are treated as corrupt. If they are real returns or refunds, they should be loaded as negative revenue instead.
- The whole file is processed in memory, which is fine for tens of thousands of rows but would need chunking at much larger sizes.
- The pipeline is run manually and Power BI uses Import mode, so the report needs a refresh after each load.

## Possible next steps
- Unit tests for the transform step (pytest)
- Load-history table and skipping files that were already loaded
- Scheduling (Task Scheduler or cron) with failure alerts
- Docker Compose for a one-command setup
- Pipeline-health and store-performance pages in the dashboard
