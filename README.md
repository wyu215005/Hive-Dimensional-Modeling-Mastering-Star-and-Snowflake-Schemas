# Hive Dimensional Modeling: Mastering Star and Snowflake Schemas

> **"Storage is cheap, but joins are expensive. That's why we model."**
>
> 存储很便宜，但关联查询很贵——这就是我们需要建模的原因。

A deep dive into data modeling: Fact/Dimension table design, SCD (Slowly Changing Dimension) patterns, and the practical application of Surrogate Keys in Hive environments.

---

## Table of Contents

1. [I. Visual Introduction — Star vs. Snowflake](#i-visual-introduction--star-vs-snowflake)
2. [II. Core Modules](#ii-core-modules)
   - [Module 1 — Fact vs. Dimension Tables](#module-1--fact-vs-dimension-tables)
   - [Module 2 — The SCD Challenge (Slowly Changing Dimensions)](#module-2--the-scd-challenge-slowly-changing-dimensions)
   - [Module 3 — Surrogate Keys in Hive](#module-3--surrogate-keys-in-hive)
   - [Module 4 — Performance Tuning](#module-4--performance-tuning)
3. [III. Interactive Summary — Modeling Self-Check Checklist](#iii-interactive-summary--modeling-self-check-checklist)

---

## I. Visual Introduction — Star vs. Snowflake

### Star Schema (星型模式)

In a **Star Schema**, a central Fact table connects directly to every Dimension table. The shape looks like a star — one hub, many spokes. Dimensions are fully **denormalized**: all attributes live in a single table, even if that means repeating values.

```
                    ┌─────────────────┐
                    │   dim_date      │
                    │─────────────────│
                    │ date_sk (PK)    │
                    │ full_date       │
                    │ year / quarter  │
                    │ month / week    │
                    └────────┬────────┘
                             │
┌─────────────────┐          │          ┌─────────────────┐
│   dim_customer  │          │          │   dim_product   │
│─────────────────│          │          │─────────────────│
│ customer_sk(PK) │          │          │ product_sk (PK) │
│ customer_name   │          │          │ product_name    │
│ city            ├──────────┤          │ category        │
│ country         │          │          │ brand           │
│ segment         │          │          │ unit_price      │
└────────┬────────┘          │          └────────┬────────┘
         │                   │                   │
         │          ┌────────▼────────┐          │
         └──────────►   fact_sales    ◄──────────┘
                    │─────────────────│
                    │ sale_sk (PK)    │
                    │ date_sk (FK)    │
                    │ customer_sk(FK) │
                    │ product_sk (FK) │
                    │ store_sk (FK)   │
                    │ quantity        │
                    │ amount          │
                    │ discount        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   dim_store     │
                    │─────────────────│
                    │ store_sk (PK)   │
                    │ store_name      │
                    │ city / state    │
                    │ region          │
                    └─────────────────┘
```

**Pros:** Fast queries (fewer joins), simple structure, ideal for Hive's MPP architecture.  
**Cons:** Data redundancy in dimension tables; harder to maintain when an attribute (e.g., a product category hierarchy) changes frequently.

---

### Snowflake Schema (雪花模式)

A **Snowflake Schema** normalizes dimension tables into sub-dimensions. Repeated values are extracted into separate lookup tables, forming a "snowflake" shape.

```
┌──────────────┐    ┌──────────────────┐    ┌────────────────┐
│  dim_brand   │    │   dim_product    │    │  dim_category  │
│──────────────│    │──────────────────│    │────────────────│
│ brand_sk(PK) ◄────│ product_sk (PK)  ├────► category_sk(PK)│
│ brand_name   │    │ product_name     │    │ category_name  │
│ country      │    │ brand_sk    (FK) │    │ department     │
└──────────────┘    │ category_sk (FK) │    └────────────────┘
                    │ unit_price       │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   fact_sales     │
                    │──────────────────│
                    │ sale_sk     (PK) │
                    │ product_sk  (FK) │
                    │ customer_sk (FK) │
                    │ date_sk     (FK) │
                    │ quantity         │
                    │ amount           │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐    ┌──────────────────┐
                    │  dim_customer    │    │    dim_city      │
                    │──────────────────│    │──────────────────│
                    │ customer_sk (PK) ├────► city_sk     (PK) │
                    │ customer_name    │    │ city_name        │
                    │ city_sk     (FK) │    │ state / country  │
                    └──────────────────┘    └──────────────────┘
```

**Pros:** No data redundancy; dimension sub-tables are easy to update in isolation.  
**Cons:** More joins per query — costly in Hive. Harder to optimize with Map-side Joins.

---

### At a Glance: Star vs. Snowflake

| Dimension            | Star Schema ⭐         | Snowflake Schema ❄️       |
|----------------------|------------------------|---------------------------|
| **Structure**        | Denormalized           | Normalized (sub-dims)     |
| **Query Speed**      | ✅ Faster              | ⚠️ Slower (more joins)    |
| **Storage**          | ⚠️ More redundancy     | ✅ Less redundancy         |
| **Maintenance**      | ⚠️ Update many rows    | ✅ Update sub-dim only     |
| **Hive Suitability** | ✅ Recommended         | ⚠️ Use with caution        |
| **Best For**         | OLAP / BI dashboards   | Frequently changing attrs |

---

## II. Core Modules

---

### Module 1 — Fact vs. Dimension Tables

#### What Is a Grain? (粒度：建模的基石)

Before writing a single `CREATE TABLE`, you must answer: **What does one row represent?**

This is called the **Grain** — the most atomic level of detail captured by the fact table.

> ❌ Wrong grain: "Sales data"  
> ✅ Correct grain: "One line item on one sales order, for one product, on one day"

Getting the grain wrong means your aggregations will produce incorrect results. Define it first. Write it in a comment at the top of your DDL. Never compromise on it.

---

#### E-Commerce Case Study: `fact_sales` and `dim_product`

Imagine an e-commerce platform. Here is what each table stores:

**`fact_sales` — Measurement / Event Table**

```sql
-- Grain: One row = one product line on one order
CREATE TABLE fact_sales (
    sale_sk        BIGINT     COMMENT 'Surrogate key (PK)',
    order_id       STRING     COMMENT 'Business key from source system',
    date_sk        BIGINT     COMMENT 'FK → dim_date',
    customer_sk    BIGINT     COMMENT 'FK → dim_customer',
    product_sk     BIGINT     COMMENT 'FK → dim_product',
    store_sk       BIGINT     COMMENT 'FK → dim_store',
    -- Additive measures (can SUM across any dimension)
    quantity       INT        COMMENT 'Units sold',
    unit_price     DECIMAL(10,2),
    discount       DECIMAL(10,2),
    net_amount     DECIMAL(10,2) COMMENT 'quantity * unit_price - discount',
    -- Semi-additive (SUM only across some dims)
    inventory_snapshot INT    COMMENT 'Stock level at time of sale'
)
PARTITIONED BY (sale_date STRING)  -- Partition by date for performance
STORED AS ORC;
```

> **Rule of Thumb for Measures:**
> - **Additive** — can `SUM` across all dimensions (e.g., `net_amount`, `quantity`)
> - **Semi-Additive** — can `SUM` across some dims but not others (e.g., inventory balance)
> - **Non-Additive** — cannot `SUM` at all (e.g., ratios, percentages — store the components instead)

---

**`dim_product` — Descriptive Context Table**

```sql
-- SCD Type 2: track full history of product attribute changes
CREATE TABLE dim_product (
    product_sk       BIGINT     COMMENT 'Surrogate key (PK) — never changes',
    product_id       STRING     COMMENT 'Business key from source ERP system',
    product_name     STRING,
    category         STRING,
    sub_category     STRING,
    brand            STRING,
    supplier         STRING,
    unit_cost        DECIMAL(10,2),
    -- SCD Type 2 tracking columns
    effective_date   STRING     COMMENT 'Row valid from (YYYY-MM-DD)',
    expiration_date  STRING     COMMENT 'Row valid until (9999-12-31 = current)',
    is_current       TINYINT    COMMENT '1 = current record, 0 = historical'
)
STORED AS ORC;
```

**Key principle:** The Fact table stores *what happened* (numbers, events). The Dimension table stores *context* (who, what, where, when, why). Never mix them.

---

### Module 2 — The SCD Challenge (Slowly Changing Dimensions)

#### Why SCD Matters

Real-world data changes. A customer moves cities. A product changes its category. A salesperson transfers regions. The question is: **do you overwrite the old value, or do you keep history?**

There are three main SCD types:

| Type | Strategy | History Preserved? | Hive-Friendly? |
|------|----------|-------------------|----------------|
| **Type 0** | Never update | No (ignore changes) | ✅ Simple |
| **Type 1** | Overwrite old value | No | ✅ `INSERT OVERWRITE` |
| **Type 2** | Add a new row (拉链表) | **Yes — full history** | ⚠️ Requires workaround |
| **Type 3** | Add a "previous value" column | Partial | ✅ Simple |

---

#### SCD Type 2 in Hive — The "Zipper Table" Pattern (拉链表)

**The Pain Point:** Standard relational databases implement SCD Type 2 with `UPDATE` statements. Hive does **not support row-level updates** (in non-ACID tables). This is the central challenge.

**The Solution:** Use `INSERT OVERWRITE` + `UNION ALL` to rebuild the dimension table each time a change is detected.

---

**Step 1: Define the dimension table with SCD Type 2 columns**

```sql
CREATE TABLE dim_customer (
    customer_sk     BIGINT      COMMENT 'Surrogate key',
    customer_id     STRING      COMMENT 'Business key (stable)',
    customer_name   STRING,
    city            STRING,
    state           STRING,
    country         STRING,
    segment         STRING,
    -- SCD Type 2 audit columns
    effective_date  STRING      COMMENT 'Date this row became valid',
    expiration_date STRING      COMMENT '9999-12-31 means currently active',
    is_current      TINYINT     COMMENT '1 = active, 0 = expired'
)
STORED AS ORC;
```

---

**Step 2: Create a staging table for incoming changes**

```sql
CREATE TABLE stg_customer (
    customer_id     STRING,
    customer_name   STRING,
    city            STRING,
    state           STRING,
    country         STRING,
    segment         STRING,
    updated_date    STRING      COMMENT 'Date of change from source'
);
```

---

**Step 3: Apply SCD Type 2 logic via `INSERT OVERWRITE` + `UNION ALL`**

```sql
-- SCD Type 2 merge logic in Hive (no UPDATE needed)
INSERT OVERWRITE TABLE dim_customer
SELECT
    -- Assign new surrogate keys to brand-new or changed rows
    ROW_NUMBER() OVER (ORDER BY customer_id, effective_date) AS customer_sk,
    customer_id,
    customer_name,
    city,
    state,
    country,
    segment,
    effective_date,
    expiration_date,
    is_current
FROM (
    -- Part A: Historical rows that are NOT affected by today's changes
    --         Keep them exactly as-is (expired records and unchanged current records)
    SELECT
        d.customer_id,
        d.customer_name,
        d.city,
        d.state,
        d.country,
        d.segment,
        d.effective_date,
        -- Expire rows that match an incoming update
        CASE
            WHEN s.customer_id IS NOT NULL AND d.is_current = 1
                THEN s.updated_date  -- set expiration to the change date
            ELSE d.expiration_date
        END AS expiration_date,
        CASE
            WHEN s.customer_id IS NOT NULL AND d.is_current = 1 THEN 0
            ELSE d.is_current
        END AS is_current
    FROM dim_customer d
    LEFT JOIN stg_customer s
        ON d.customer_id = s.customer_id
       AND d.is_current = 1

    UNION ALL

    -- Part B: New rows from the staging table (the "new version" of changed records)
    SELECT
        s.customer_id,
        s.customer_name,
        s.city,
        s.state,
        s.country,
        s.segment,
        s.updated_date         AS effective_date,
        '9999-12-31'           AS expiration_date,
        1                      AS is_current
    FROM stg_customer s
) combined;
```

---

**Step 4: Query point-in-time snapshots**

```sql
-- Who was the customer on 2023-06-15?
SELECT *
FROM dim_customer
WHERE customer_id = 'C-1001'
  AND effective_date  <= '2023-06-15'
  AND expiration_date >  '2023-06-15';

-- Current state of all customers
SELECT *
FROM dim_customer
WHERE is_current = 1;
```

---

**Snapshot Table Alternative (Type 4 / Periodic Snapshot)**

For very large dimensions where full-history `UNION ALL` becomes expensive, consider a **daily snapshot** approach:

```sql
-- Append today's full dimension snapshot partitioned by snapshot_date
INSERT INTO dim_customer_snapshot PARTITION (snapshot_date = '2024-01-15')
SELECT customer_id, customer_name, city, state, country, segment
FROM dim_customer
WHERE is_current = 1;
```

This trades storage for simpler, faster time-travel queries.

---

### Module 3 — Surrogate Keys in Hive

#### The Problem with Auto-Increment in a Distributed System

In a traditional RDBMS, surrogate keys are simple: `AUTO_INCREMENT` or `SERIAL`. In Hive (a distributed system), there is **no built-in auto-increment** mechanism. Each mapper/reducer runs independently — there is no shared counter.

Why do we need surrogate keys at all?

- Business keys change (a product code can be reassigned)
- Business keys may not be unique across source systems
- They decouple the warehouse from source system key schemes
- They are essential for SCD Type 2 to distinguish row versions

---

#### Option A: `ROW_NUMBER()` Window Function

```sql
-- Generate surrogate keys using ROW_NUMBER()
CREATE TABLE dim_product AS
SELECT
    ROW_NUMBER() OVER (ORDER BY product_id, effective_date) AS product_sk,
    product_id,
    product_name,
    category,
    brand,
    effective_date,
    expiration_date,
    is_current
FROM dim_product_staging;
```

**✅ Pros:**
- Produces clean, sequential integers
- Easy to understand and debug

**⚠️ Cons:**
- **Non-deterministic across runs** — row numbers can shift if source data order changes
- In SCD Type 2 scenarios, re-running can assign different `product_sk` values to the same logical row, breaking fact table foreign keys
- Requires a full table scan + sort — expensive on large datasets

> **Mitigation:** Always base the `ORDER BY` on a stable, unique business key + `effective_date` combination. Store surrogate key mappings in a separate lookup table and only generate new keys for genuinely new rows.

---

#### Option B: `MD5(business_key)` Hash-Based Keys

```sql
-- Generate a deterministic surrogate key via MD5 hashing
CREATE TABLE dim_customer AS
SELECT
    CONV(SUBSTR(MD5(customer_id), 1, 15), 16, 10) AS customer_sk,  -- hex → bigint
    customer_id,
    customer_name,
    city,
    segment,
    effective_date,
    expiration_date,
    is_current
FROM dim_customer_staging;
```

For SCD Type 2 (where the same `customer_id` may have multiple rows with different time periods), include the `effective_date` in the hash:

```sql
CONV(SUBSTR(MD5(CONCAT(customer_id, '|', effective_date)), 1, 15), 16, 10) AS customer_sk
```

**✅ Pros:**
- **Fully deterministic** — same inputs always produce the same key
- No coordination needed across distributed nodes
- Idempotent — safe to re-run ETL pipelines

**⚠️ Cons:**
- Hash collisions are theoretically possible (extremely rare with MD5 at 15 hex digits ≈ 60-bit space)
- Resulting integers are large and non-sequential — minor impact on index performance
- Harder to read / debug in ad-hoc queries

---

#### Option C: Surrogate Key Mapping Table (Best Practice for Production)

```sql
-- Dedicated surrogate key management table
CREATE TABLE surrogate_key_map (
    entity_type      STRING   COMMENT 'e.g., product, customer',
    business_key     STRING   COMMENT 'Natural key from source',
    effective_date   STRING,
    surrogate_key    BIGINT,
    created_date     STRING
)
STORED AS ORC;

-- Lookup or insert: join new data against the map, generate keys only for unknowns
INSERT INTO surrogate_key_map
SELECT
    'product'        AS entity_type,
    s.product_id     AS business_key,
    s.effective_date AS effective_date,
    -- Offset new keys beyond the current maximum to avoid collisions
    (SELECT COALESCE(MAX(surrogate_key), 0) FROM surrogate_key_map
      WHERE entity_type = 'product') + ROW_NUMBER() OVER (ORDER BY s.product_id)
                     AS surrogate_key,
    CURRENT_DATE()   AS created_date
FROM stg_product s
LEFT JOIN surrogate_key_map m
    ON  m.entity_type  = 'product'
    AND m.business_key = s.product_id
    AND m.effective_date = s.effective_date
WHERE m.surrogate_key IS NULL;  -- Only generate keys for rows not already mapped
```

This pattern is the most robust for production Hive pipelines where key stability is critical.

---

#### Surrogate Key Strategy Comparison

| Strategy | Deterministic | Distributed-Safe | Readable | SCD Type 2 Ready |
|----------|:---:|:---:|:---:|:---:|
| `ROW_NUMBER()` | ❌ | ⚠️ (with care) | ✅ | ⚠️ |
| `MD5(biz_key)` | ✅ | ✅ | ❌ | ✅ (with date) |
| Mapping Table | ✅ | ✅ | ✅ | ✅ |

---

### Module 4 — Performance Tuning

#### Why Star Schema Is Preferred in Hive

Hive processes queries by translating them into MapReduce (or Tez/Spark) jobs. Each `JOIN` typically requires a **shuffle** — data is moved across the network between nodes. Minimizing joins directly reduces shuffle overhead.

| Schema | Join Depth | Shuffle Operations | Query Latency |
|--------|-----------|-------------------|---------------|
| Star ⭐ | 1 level (fact → dim) | Low | ✅ Fast |
| Snowflake ❄️ | 2–4 levels (fact → dim → sub-dim) | High | ⚠️ Slower |

---

#### Technique 1: Map-Side Join (Broadcast Join)

When one side of a join is small enough to fit in memory, Hive can **broadcast** it to all mappers, eliminating the shuffle entirely.

```sql
-- Enable map-side join (auto-convert when small table < threshold)
SET hive.auto.convert.join = true;
SET hive.mapjoin.smalltable.filesize = 25000000;  -- 25 MB threshold

-- Explicit hint (older Hive versions)
SELECT /*+ MAPJOIN(d) */
    f.sale_sk,
    f.amount,
    d.product_name,
    d.category
FROM fact_sales f
JOIN dim_product d
    ON f.product_sk = d.product_sk
   AND d.is_current = 1;
```

**Star Schema advantage:** Each dimension table is self-contained and typically small enough for a Map-side Join. In a Snowflake Schema, you must chain joins to sub-dimension tables, often breaking the size threshold.

---

#### Technique 2: Partition Pruning

Partition the fact table by date. Hive will skip partitions that don't match the `WHERE` clause:

```sql
-- Without partitioning: full table scan
SELECT SUM(net_amount) FROM fact_sales WHERE sale_date = '2024-01-15';

-- With PARTITIONED BY (sale_date STRING): only one partition is read ✅
SELECT SUM(net_amount) FROM fact_sales WHERE sale_date = '2024-01-15';
```

---

#### Technique 3: ORC + Columnar Storage

```sql
-- Prefer ORC with Snappy compression for analytical workloads
CREATE TABLE fact_sales (...)
STORED AS ORC
TBLPROPERTIES (
    'orc.compress' = 'SNAPPY',
    'orc.stripe.size' = '67108864',   -- 64 MB stripes
    'orc.row.index.stride' = '10000'
);
```

ORC stores data column-by-column, so a query reading only `quantity` and `net_amount` never touches `discount` or `order_id` — massive I/O savings.

---

#### Technique 4: Bucketing for Join Optimization

When joins between large tables are unavoidable (e.g., fact-to-fact or large dimension joins), bucketing both tables on the join key ensures aligned data placement:

```sql
CREATE TABLE fact_sales (...)
CLUSTERED BY (product_sk) INTO 256 BUCKETS
STORED AS ORC;

CREATE TABLE dim_product (...)
CLUSTERED BY (product_sk) INTO 256 BUCKETS
STORED AS ORC;

-- Enable bucket-map join
SET hive.optimize.bucketmapjoin = true;
```

---

#### Performance Tuning Summary

```
Query is slow?
     │
     ├─► Is the fact table full-scanned?
     │       └─► Add DATE partition → Partition Pruning
     │
     ├─► Is a dimension join slow?
     │       └─► Is dim table < 25 MB? → Map-side Join (MAPJOIN hint)
     │           Is dim table > 25 MB? → Bucket both tables → Bucket-Map Join
     │
     ├─► Are you reading too many columns?
     │       └─► Use ORC/Parquet columnar format
     │
     └─► Are you doing multi-level joins (snowflake)?
             └─► Consider flattening into a Star Schema dimension
```

---

## III. Interactive Summary — Modeling Self-Check Checklist

Use this checklist to evaluate the quality of any dimensional model before deploying to production.

### 🏗️ Foundation

- [ ] **Grain is defined** — written in plain English as a comment in the DDL: *"One row = …"*
- [ ] **Grain is the most atomic** — you have not summarized or pre-aggregated in the fact table
- [ ] **Fact table contains only measures and foreign keys** — no descriptive text columns
- [ ] **Dimensions contain no measures** — no totals, counts, or sums in dimension tables
- [ ] **All foreign keys in the fact table have a corresponding dimension** — no orphaned FKs

### 🗂️ Schema Design

- [ ] **Schema type chosen for the right reason:**
  - Star Schema → chosen for query performance and simplicity (preferred for Hive)
  - Snowflake Schema → chosen specifically to reduce update anomalies on high-churn attributes
- [ ] **Dimensions are conformed** — `dim_date`, `dim_customer` are shared across all fact tables with identical keys and definitions
- [ ] **Degenerate dimensions** (e.g., `order_id`, `invoice_number`) are stored directly in the fact table, not in a separate dim with a single column

### 🔑 Surrogate Keys

- [ ] **Every dimension table has a surrogate key (SK)** — not the business key as PK
- [ ] **Surrogate key strategy is chosen and documented:**
  - `ROW_NUMBER()` with stable `ORDER BY` → simple pipelines
  - `MD5(biz_key [+ effective_date])` → distributed, idempotent pipelines
  - Mapping table → production-grade, key stability guaranteed
- [ ] **Fact table foreign keys reference surrogate keys**, not business keys
- [ ] **A "unknown member" row exists** (SK = -1 or 0) in each dimension to handle NULL foreign keys gracefully

### 🔄 SCD (Slowly Changing Dimensions)

- [ ] **SCD type is documented per dimension:**
  - Type 0 → static / never changes (e.g., `dim_date`)
  - Type 1 → overwrite (history not needed)
  - Type 2 → zipper table with `effective_date` / `expiration_date` / `is_current`
- [ ] **SCD Type 2 dimensions have three mandatory columns:**
  - `effective_date STRING` — when this row became valid
  - `expiration_date STRING` — when this row expired (`'9999-12-31'` = still active)
  - `is_current TINYINT` — `1` for the active record, `0` for history
- [ ] **ETL logic uses `INSERT OVERWRITE` + `UNION ALL`** (not `UPDATE`) for SCD Type 2 in Hive
- [ ] **Point-in-time queries are tested** — joining fact to SCD Type 2 dim on date range returns exactly one row per business key

### ⚡ Hive Performance

- [ ] **Fact table is partitioned** by the most common filter column (usually `sale_date`)
- [ ] **Dimension tables use ORC or Parquet** format
- [ ] **`hive.auto.convert.join = true`** is set for Map-side Joins on small dimensions
- [ ] **Frequently joined large dimensions are bucketed** on their surrogate key
- [ ] **`is_current = 1` filter** is applied on SCD Type 2 dimensions in all current-state queries (prevents row duplication in joins)
- [ ] **Execution plan reviewed** with `EXPLAIN` before deploying new queries

### 📋 Documentation & Governance

- [ ] **Each table has a `COMMENT`** explaining its purpose and grain
- [ ] **Each column has a `COMMENT`** explaining its business meaning
- [ ] **ETL pipeline is idempotent** — re-running produces the same result (no duplicate rows)
- [ ] **Data lineage is documented** — source system → staging → dimension/fact is traceable
- [ ] **Sample data volume per table** is estimated and documented

---

## Quick Reference Card

```
DIMENSIONAL MODELING IN HIVE — QUICK REFERENCE
═══════════════════════════════════════════════

FACT TABLE                    DIMENSION TABLE
──────────────────────────    ──────────────────────────
✓ Surrogate key (PK)          ✓ Surrogate key (PK)
✓ Foreign keys to dims        ✓ Business key
✓ Additive measures           ✓ Descriptive attributes
✓ Grain = 1 business event    ✓ SCD Type + audit cols
✓ Partitioned by date         ✓ Stored as ORC
✗ No descriptive text         ✗ No measures or totals

SCD TYPE 2 — ZIPPER TABLE     SURROGATE KEY OPTIONS
──────────────────────────    ──────────────────────────
effective_date  STRING        ROW_NUMBER() OVER (...)
expiration_date STRING           → Simple, non-deterministic
is_current      TINYINT(1)    MD5(biz_key|eff_date)
                                 → Deterministic, distributed
Hive Pattern:                 Mapping Table
  INSERT OVERWRITE               → Production-grade, stable
  SELECT ... FROM
  (historical UNION ALL new)

PERFORMANCE CHECKLIST
──────────────────────────────────────────────────────
1. Partition fact table by date
2. Use ORC/Parquet + Snappy compression
3. Enable hive.auto.convert.join for small dims
4. Bucket large dims on SK for bucket-map join
5. Always filter SCD Type 2 dims with is_current=1
6. Prefer Star Schema to minimize join depth
7. Run EXPLAIN on all new queries before prod
```

---

## License

This project is open for learning and reference. Contributions, corrections, and improvements are welcome via pull request.

---

*Built with ❤️ for data engineers navigating the Hive ecosystem.*
