# ClickHouse SummingMergeTree Lab - Explained

## Overview

This lab demonstrates **SummingMergeTree**, a specialized ClickHouse table engine that automatically **aggregates (sums) numeric values** for rows sharing the same ORDER BY key during background merges.

---

## What is SummingMergeTree?

**SummingMergeTree** is designed for:
- **Time-series data** with frequent inserts
- **Pre-aggregation at storage level** (not at query time)
- **Automatic summarization** of numeric columns during merges

### Key Concept

When ClickHouse performs a merge operation, rows with **identical ORDER BY keys** are combined into a single row, with numeric values **automatically summed together**.

**Comparison with Other Engines:**

| Engine | Behavior |
|--------|----------|
| **MergeTree** | Keeps all rows as-is |
| **ReplacingMergeTree** | Keeps the latest row (replacement) |
| **SummingMergeTree** | **Sums numeric columns** |
| **AggregatingMergeTree** | Complex aggregation states |

---

## Lab Walkthrough

### Step 1-3: Setup
```sql
CREATE DATABASE lab_summing;
USE lab_summing;

CREATE TABLE daily_sales (
    order_date Date,
    product_category String,
    total_sales Decimal(12,2)
)
ENGINE = SummingMergeTree()
ORDER BY (order_date, product_category);
```

**Critical:** `ORDER BY (order_date, product_category)` defines the grouping key.

### Step 4-5: Insert Data
```
2026-09-09 | Mobile | 100  ─┐
2026-09-09 | Mobile | 200  ├─ Same key → Will sum
2026-09-09 | Mobile | 300  ┘
2026-09-09 | Laptop | 500
```

**Before merge:** Multiple rows exist (ClickHouse splits data into parts).

### Step 6-7: Force Merge & Aggregate
```sql
OPTIMIZE TABLE daily_sales FINAL;
```

**After merge:**
```
2026-09-09 | Mobile | 600  (100 + 200 + 300)
2026-09-09 | Laptop | 500
```

### Step 8: Verification
The `GROUP BY` query validates that the merge result matches manual aggregation:
```sql
SELECT order_date, product_category, SUM(total_sales)
FROM daily_sales
GROUP BY order_date, product_category;
```

---

## How It Works (Under the Hood)

```
INSERTION PHASE (No aggregation yet)
├── Insert: (2026-09-09, Mobile, 100) → Part 1
├── Insert: (2026-09-09, Mobile, 200) → Part 2
├── Insert: (2026-09-09, Mobile, 300) → Part 2
└── Insert: (2026-09-09, Laptop, 500) → Part 2

MERGE PHASE (Background, automatic or OPTIMIZE)
├── Compare ORDER BY keys
├── Mobile (2026-09-09): 100 + 200 + 300 = 600
└── Laptop (2026-09-09): 500 (only one row)

RESULT: Single merged row per key
```

---

## Real-World Use Cases

### 1. **Website Analytics Dashboard**
```sql
CREATE TABLE page_views (
    event_date Date,
    page_id UInt32,
    views UInt32,
    clicks UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (event_date, page_id);
```

**Scenario:**
- Multiple servers log page views every second
- Each insert adds a row: `(2026-09-09, page_123, 1, 0)` per view
- SummingMergeTree automatically sums: `(2026-09-09, page_123, 5000, 250)`
- **Benefit:** Query-time performance improves, storage is optimized

---

### 2. **Financial Trading Volume Aggregation**
```sql
CREATE TABLE trade_volume (
    trade_date Date,
    symbol String,
    quantity Int64,
    revenue Decimal(18,2)
)
ENGINE = SummingMergeTree()
ORDER BY (trade_date, symbol);
```

**Scenario:**
- Millions of trades per day for each stock
- Instead of storing each trade separately, sum volumes per symbol per day
- Query `SELECT SUM(quantity) FROM trade_volume WHERE symbol = 'AAPL'` is instant (already pre-computed)

---

### 3. **IoT Sensor Data Aggregation**
```sql
CREATE TABLE sensor_metrics (
    timestamp Date,
    sensor_id String,
    temperature Float32,
    humidity Float32,
    error_count UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (timestamp, sensor_id);
```

**Scenario:**
- Thousands of IoT sensors sending readings every minute
- Instead of storing 1.44M rows/day per sensor, store 1 aggregated row
- `temperature` and `humidity` are automatically averaged (via SUM + manual division)
- `error_count` is summed: fault detection at a glance

---

### 4. **E-Commerce Sales Dashboard**
```sql
CREATE TABLE daily_revenue (
    event_date Date,
    store_id UInt16,
    product_category String,
    total_revenue Decimal(12,2),
    order_count UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (event_date, store_id, product_category);
```

**Scenario:**
- Chain of 100 stores, each with POS systems
- Each order creates a row: `(2026-09-09, store_5, 'Electronics', 250.00, 1)`
- After merges: `(2026-09-09, store_5, 'Electronics', 15000.00, 60)`
- Real-time dashboards show revenue instantly without scanning millions of rows

---

### 5. **Database Query Profiling**
```sql
CREATE TABLE query_stats (
    event_date Date,
    query_type String,
    execution_time_ms UInt32,
    rows_processed UInt64
)
ENGINE = SummingMergeTree()
ORDER BY (event_date, query_type);
```

- Sum total query time per type per day
- Identify performance bottlenecks in seconds

---

## Key Advantages

| Benefit | Impact |
|---------|--------|
| **Reduced Storage** | Multiple inserts → 1 summed row |
| **Fast Queries** | Pre-aggregated data, no GROUP BY needed |
| **Write-Heavy Workloads** | Handles millions of inserts/sec |
| **Automatic Aggregation** | No manual ETL/rollup needed |
| **Lower CPU Cost** | Queries don't recalculate sums |

---

## Important Notes ⚠️

### When Merge Happens
- Automatically: Background merge thread (~5-10 minutes by default)
- Manually: `OPTIMIZE TABLE table_name FINAL` (for testing)
- **In production:** Merges happen asynchronously; queries may return unmerged data

### What Gets Summed
```sql
-- Only numeric columns are summed:
-- UInt*, Int*, Float*, Decimal

-- String columns (like product_category) are **grouped**, not summed
-- Date columns (like order_date) are **grouped**, not summed
```

### Query Caveat
```sql
-- ❌ WRONG: Doesn't account for pre-merge rows
SELECT total_sales FROM daily_sales WHERE order_date = '2026-09-09';

-- ✅ CORRECT: Aggregates even unmerged rows
SELECT SUM(total_sales) FROM daily_sales 
WHERE order_date = '2026-09-09' 
GROUP BY product_category;
```

---

## Summary

**SummingMergeTree is ideal for:**
- High-volume metric collection (analytics, monitoring, trading)
- Pre-aggregated time-series data
- Dashboards requiring instant sums without GROUP BY overhead
- Systems where raw event granularity isn't needed

**Use ReplacingMergeTree if** you need only the latest state (user profiles, device status).  
**Use AggregatingMergeTree if** you need complex statistics (avg, percentiles, distinct counts).
