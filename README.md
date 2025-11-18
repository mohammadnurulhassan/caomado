# caomado


# NordicMart Sweden Expansion – Data Engineering Case

**Candidate:** Mohammad Nurul Hassan  
**Company:** Curamando / Eidra  
**Environment:** BigQuery (`eidra-df-case.df2025`)

---

## 1. Understanding the Case & Overall Approach

NordicMart is preparing to expand into the Swedish market and has started collecting early data from pilot stores and digital channels. Before business stakeholders and data scientists can rely on the data, the foundation must be validated, cleaned, and structured for analysis.

**Role:** Data Engineer with four main objectives:

1. Explore the datasets in BigQuery (`eidra-df-case.df2025`).
2. Identify and describe data-quality issues and suggest mitigations.
3. Clarify relationships between entities and present them in an ERD.
4. Propose an analytics-ready data model and explain its analytical value.

### 1.1 Working Steps

- **Exploration**
  - Previewed tables using `SELECT … LIMIT`.
  - Used row counts, distinct counts, and basic distributions to understand shape and scale.

- **Data Quality Assessment**
  - Checked for duplicate IDs, missing values, broken links, and suspicious numeric values.

- **Relationship Mapping**
  - Inferred links between `users`, `stores`, `products`, `transactions`, `web_sessions`, and `regions`.
  - Summarized these relationships in a conceptual ERD.

- **Modeling**
  - Proposed a star-schema model centered on transactions with supporting dimensions.
  - Added a separate fact table for online sessions.

- **Business Framing**
  - Aligned cleaned data with analytical questions relevant to NordicMart’s Swedish expansion.

> _Note: Permission to create tables/views was restricted (`bigquery.tables.create denied`), so all cleaning was demonstrated using SELECT-based queries._

---

## 2. Dataset Overview

All tables are in the following dataset:

- **Project:** `eidra-df-case`
- **Dataset:** `df2025`

### 2.1 Tables

| Table | Description |
|:------|:-------------|
| `df2025.products` | Product reference data. |
| `df2025.stores` | Store metadata including location and channel. |
| `df2025.users` | User and loyalty information. |
| `df2025.transactions` | Transaction-level data with sales measures. |
| `df2025.web_sessions` | Online web sessions data. |
| `df2025.regions` | Regions mapping table. |

### 2.2 Example Summary Query

SELECT 'products' AS table_name, COUNT() AS row_count FROM eidra-df-case.df2025.products
UNION ALL
SELECT 'stores', COUNT() FROM eidra-df-case.df2025.stores
UNION ALL
SELECT 'users', COUNT() FROM eidra-df-case.df2025.users
UNION ALL
SELECT 'transactions', COUNT() FROM eidra-df-case.df2025.transactions
UNION ALL
SELECT 'web_sessions', COUNT() FROM eidra-df-case.df2025.web_sessions
UNION ALL
SELECT 'regions', COUNT() FROM eidra-df-case.df2025.regions;


---

## 3. ERD & Relationships (Conceptual View)

### 3.1 Core Relationships

- **Users → Transactions** via `user_id`
- **Stores → Transactions** via `store_id`
- **Products → Transactions** via `product_id`
- **Regions → Users, Stores, Transactions** via `region_code`
- **Users → Web sessions** via `user_id`
- **Regions → Web sessions** via `region_code`

### 3.2 Business-Friendly ERD

Core tables:

- Customers → `users`
- Stores → `stores`
- Products → `products`
- Transactions → `transactions`
- Web Sessions → `web_sessions`
- Regions → `regions`

**Concept:**  
Transactions sit at the center connecting customers, stores, products, and regions.  
Online sessions connect customers (when identifiable) and regions, linking digital engagement with offline sales.

---

## 4. Data Quality Issues & Mitigations

### 4.1 Users (`df2025.users`)

**Checks:**

- Unique IDs.  
- Missing values across key fields.

**Findings:**

- `user_id` unique.  
- 16 users missing `first_name`.

**Mitigation:**

- Retain rows with missing names. Replace nulls via `COALESCE(first_name, 'Unknown')`.  
- For missing `region_code`, group under “Unknown”.

---

### 4.2 Stores (`df2025.stores`)

**Findings:**

- `store_id` unique.  
- Channel types distinguish store formats.  
- Most stores correctly map to regions.

**Mitigation:**

- Enforce valid `region_code` references.  
- Fix invalid codes or exclude from region-level metrics.

---

### 4.3 Products (`df2025.products`)

**Findings:**

- `product_id` unique and category available.  

**Mitigation:**

- Standardize textual values (case consistency).  
- Ensure all key products are categorized.

---

### 4.4 Transactions (`df2025.transactions`)

**Findings:**

- Central fact table with occasional missing links.  
- Some negative or zero values (returns).  
- Missing foreign keys to dimension tables.

**Mitigation:**

- Treat negative values as returns.  
- Exclude or flag unlinked rows.  
- Define grain: transaction-level or line-level.

---

### 4.5 Web Sessions (`df2025.web_sessions`)

**Findings:**

- Some sessions missing user_id or region.  
- Inconsistent durations.

**Mitigation (SELECT example):**
```sql
SELECT
  session_id,
  COALESCE(user_id, 'unknown_user') AS user_id_clean,
  COALESCE(page, 'unknown_page') AS page_clean,
  timestamp_utc,
  COALESCE(region_code, 'Unknown') AS region_code_clean,
  CASE
    WHEN duration_seconds IS NULL OR duration_seconds < 0 THEN 0
    ELSE duration_seconds
  END AS duration_seconds_clean
FROM `eidra-df-case.df2025.web_sessions`;





---

## 5. Proposed Analytics-Ready Data Model

### 5.1 Fact Tables

#### `fact_transactions`
- **Grain:** One row per transaction.
- **Keys:** `transaction_id`, `user_id`, `store_id`, `product_id`, `region_code`, `date_key`
- **Measures:** `quantity`, `gross_amount_sek`, derived metrics like price per unit.

#### `fact_web_sessions`
- **Grain:** One row per web session.
- **Keys:** `session_id`, `user_id` (nullable), `region_code`, `date_key`
- **Measures:** `duration_seconds`, derived engagement metrics.

---

### 5.2 Dimension Tables

| Dimension | Key | Notes |
|:-----------|:----|:------|
| `dim_user` | `user_id` | Clean names, demographics. |
| `dim_store` | `store_id` | Store metadata and channel type. |
| `dim_product` | `product_id` | Product name, category, brand. |
| `dim_region` | `region_code` | Region name, optional metrics. |
| `dim_date` | `date_key` | Calendar and time attributes. |

---

### 5.3 Analytical Benefits

- Consistent slicing between offline and online data.
- Enables queries such as:
  - Sales by region, store type, or product category.
  - Customer behavior by segment.
  - Correlation between online sessions and in-store transactions.




---

## 6. Example Analytical Value

### 6.1 Sales by Region and Month

SELECT
  r.region_name,
  d.year,
  d.month,
  SUM(f.gross_amount_sek) AS total_sales_sek
FROM `...fact_transactions` AS f
JOIN `...dim_region` AS r
  ON f.region_code = r.region_code
JOIN `...dim_date` AS d
  ON f.date_key = d.date_key
GROUP BY
  r.region_name,
  d.year,
  d.month
ORDER BY
  d.year,
  d.month,
  total_sales_sek DESC;




### 6.2 Sessions vs. Transactions by Region

WITH tx AS (
  SELECT
    region_code,
    COUNT(*) AS transactions_count
  FROM `...fact_transactions`
  GROUP BY region_code
),
sess AS (
  SELECT
    region_code,
    COUNT(*) AS sessions_count
  FROM `...fact_web_sessions`
  GROUP BY region_code
)
SELECT
  r.region_name,
  tx.transactions_count,
  sess.sessions_count
FROM tx
JOIN sess USING (region_code)
JOIN `...dim_region` AS r USING (region_code);





This comparison highlights regions where marketing yields high engagement but limited conversion — valuable insight for market optimization.

---

## 7. Conclusion & Next Steps

The NordicMart datasets provide a coherent view of customers, stores, products, transactions, web sessions, and regions.  
Through SQL-based exploration in BigQuery:

- Validated table structures and relationships.
- Identified missing fields and broken references.
- Proposed mitigation using `COALESCE`, `CASE WHEN`, and SELECT-based cleaning.

### Proposed Model Outcome

- Core facts: `fact_transactions`, `fact_web_sessions`
- Shared dimensions: `dim_user`, `dim_store`, `dim_product`, `dim_region`, `dim_date`
- Enables multi-dimensional analysis of sales and engagement.

### Next Steps

1. Materialize cleaned and modeled views in BigQuery.  
2. Schedule ETL refresh pipelines.  
3. Build dashboards or self-service reports on this analytics layer.

---





