# E-Commerce Medallion Analytics Project

End-to-end data engineering pipeline built on **Databricks** that processes raw e-commerce data through a **medallion architecture** (Bronze → Silver → Gold), covering both **dimension** and **fact** processing.

## Project Structure

```
1_medaillion_arc/
├── 1dim_bronze.py                    # Dimension data → Bronze layer (raw ingestion)
├── Silver Layer Processing.py       # Bronze → Silver (cleansing & standardization)
├── dim_gold.py                       # Silver → Gold (star-schema dimensions)
└── medillian_processing_fact/
    ├── 1_fact_bronze.py              # Order items → Bronze layer
    ├── 2_fact_silver.py              # Fact cleansing, dedup & type casting
    └── 3_fact_gold.py                # Gold fact table + currency conversion
```

## Pipeline Layers

### Bronze (raw ingestion)
Reads CSVs from `/Volumes/ecommerece/source_data/raw/` (brands, category, products, customers, date, order_items) with explicit schemas, and appends source-file and ingestion-timestamp metadata columns.

Tables: `brz_brands`, `brz_category`, `brz_products`, `brz_customers`, `brz_calendar`, `brz_order_items`

### Silver (cleansing & standardization)

* Trimming and regex cleanup of codes and names
* Standardized category codes (GROCERY→GRCY, BOOKS→BKS, TOYS→TOY)
* Deduplication (category, products, customers, calendar)
* Anomaly fixes: `"Two"` → `2` quantities, `"305g"` → `305` grams, comma decimals (`22,2` → `22.2`), spelling fixes (Coton→Cotton, Alumium→Aluminum, Ruber→Rubber), negative ratings → absolute values
* Null handling: drop rows with null `customer_id`, fill null phones with `"Not Available"`
* Type casting: dates, timestamps (multiple formats), numerics

Tables: `slv_brands`, `slv_category`, `slv_products`, `slv_customers`, `slv_calendar`, `slv_order_items`

### Gold (BI-ready star schema)

* `gld_dim_products` — products enriched with brand and category names (COALESCE to 'Not Available')
* `gld_dim_customers` — customers enriched with country/state region mapping
* `gld_dim_date` — calendar with `date_id`, month name, weekend flag, quarter/week labels
* `gld_fact_order_items` — transactions with gross/discount/sale amounts, coupon flag, and INR normalization via fixed FX rates (as of 2025-10-15)

## How to Run

1. Use a Databricks workspace with Unity Catalog access to the `ecommerece` catalog.
2. Run the dimension flow in order: `1dim_bronze` → `Silver Layer Processing` → `dim_gold`.
3. Run the fact flow in order: `1_fact_bronze` → `2_fact_silver` → `3_fact_gold`.

Alternatively, run everything via the `dimension_processing` Lakeflow Job, which chains the fact notebooks (bronze → silver → gold).

## Data Quality Highlights

* Deduplication on business keys (`order_id`+`item_seq`, `category_code`, `date`)
* Mixed-format datetime parsing with fallbacks (`yyyy-MM-dd HH:mm:ss` and `dd-MM-yyyy HH:mm`)
* Currency normalization to INR for cross-region revenue comparability
* Sanity checks at the end of each gold table (row counts, DESCRIBE EXTENDED)
