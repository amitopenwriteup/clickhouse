# ClickHouse Lab Guide: Materialized Views — Incremental, Refreshable & Cascading

This lab covers three different types of Materialized Views (MVs) in ClickHouse, using a simple online store scenario: orders and customers.

---

## Step 0: Setup — The Scenario

We'll build three things on top of an `orders` table:

1. **Part A — Incremental MV**: a real-time daily revenue rollup per category.
2. **Part B — Refreshable MV**: a periodically-refreshed "top spending customers" report that needs a JOIN.
3. **Part C — Cascading MV**: chaining the daily rollup into a monthly rollup, then a yearly rollup.

**Create the database and base tables:**

```sql
CREATE DATABASE IF NOT EXISTS shop;
USE shop;

CREATE TABLE customers
(
    customer_id UInt32,
    name String,
    country String
)
ENGINE = MergeTree
ORDER BY customer_id;

CREATE TABLE orders
(
    order_id UInt64,
    customer_id UInt32,
    category String,
    order_time DateTime,
    amount Decimal(10, 2)
)
ENGINE = MergeTree
ORDER BY order_time;

INSERT INTO customers VALUES
    (1, 'Asha', 'IN'),
    (2, 'Ben', 'UK'),
    (3, 'Carla', 'ES');
```

`customers` is a small lookup table; `orders` is the stream of raw transactions everything else will be built from.

---

## Part A — Incremental Materialized View

**Goal:** keep a live "revenue per day per category" table always up to date, without ever re-scanning the whole `orders` table.

### Step 1: Target table using SummingMergeTree

```sql
CREATE TABLE daily_revenue
(
    day Date,
    category String,
    total_amount Decimal(18, 2),
    order_count UInt32
)
ENGINE = SummingMergeTree
ORDER BY (day, category);
```

`SummingMergeTree` automatically adds up numeric columns (like `total_amount` and `order_count`) for rows that share the same `ORDER BY` key — here, the same `(day, category)`.

### Step 2: The materialized view

```sql
CREATE MATERIALIZED VIEW daily_revenue_mv
TO daily_revenue
AS
SELECT
    toDate(order_time) AS day,
    category,
    sum(amount) AS total_amount,
    count() AS order_count
FROM orders
GROUP BY day, category;
```

**Important detail:** the `GROUP BY` columns (`day`, `category`) must match the target table's `ORDER BY` — this is required for `SummingMergeTree` to correctly merge rows together.

**How an incremental MV actually works:** it doesn't run continuously in the background. Instead, every time a block of rows is inserted into `orders`, ClickHouse runs this `SELECT` against just that new block, and inserts the result into `daily_revenue`.

### Step 3: Insert orders and watch it update

```sql
INSERT INTO orders VALUES
    (1, 1, 'electronics', now(), 120.00),
    (2, 2, 'books',       now(), 15.50),
    (3, 1, 'electronics', now(), 45.00);

SELECT day, category, sum(total_amount) AS revenue, sum(order_count) AS orders
FROM daily_revenue
GROUP BY day, category;
```

You should see `electronics` at **165.00** across 2 orders, and `books` at **15.50** across 1 order — computed instantly at insert time, not when you later query it.

### Step 4: The historical-data gotcha

```sql
INSERT INTO orders VALUES (4, 3, 'toys', now() - INTERVAL 3 DAY, 30.00);
```

This new row is picked up immediately, even though its `order_time` is 3 days in the past. **Key point:** the MV reacts to *when a row is inserted*, not what its timestamp says.

But there's a catch: if `orders` already had rows **before** the MV was created, those old rows will **never** appear in `daily_revenue` — unless you add `POPULATE` when creating the MV:

```sql
-- Only if backfilling existing data at creation time:
-- CREATE MATERIALIZED VIEW daily_revenue_mv TO daily_revenue AS ... POPULATE;
```

**Takeaway for Part A:** incremental MVs are great for real-time, single-table rollups, but they only ever see data inserted *after* they were created (unless you backfill with `POPULATE`).

---

## Part B — Refreshable Materialized View

**Goal:** build a "top customers by spend" report that needs a `JOIN` between `orders` and `customers`. Because a customer's total can be affected by *any* historical order (not just the newest insert), this is a better fit for a refreshable MV than an incremental one.

### Step 1: Target table

```sql
CREATE TABLE top_customers
(
    customer_id UInt32,
    name String,
    country String,
    total_spent Decimal(18, 2),
    updated_at DateTime
)
ENGINE = MergeTree
ORDER BY customer_id;
```

### Step 2: The refreshable MV

```sql
CREATE MATERIALIZED VIEW top_customers_mv
REFRESH EVERY 30 SECOND TO top_customers
AS
SELECT
    c.customer_id AS customer_id,
    c.name        AS name,
    c.country     AS country,
    sum(o.amount) AS total_spent,
    now()         AS updated_at
FROM orders AS o
LEFT JOIN customers AS c ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.name, c.country;
```

**Important detail:** every selected column is explicitly aliased with `AS`, even ones like `c.customer_id AS customer_id` that might look redundant. This matters because once a `JOIN` is involved, ClickHouse may keep the qualified name (`c.customer_id`) as the output column name unless you rename it explicitly — and that name has to exactly match the target table's column name (`customer_id`), or you'll get a `THERE_IS_NO_COLUMN` error.

**Key difference from Part A:** this view does **not** react to individual inserts. Instead, it completely re-runs the whole query on a timer — here, every 30 seconds — and overwrites `top_customers` with the fresh result.

### Step 3: Force an immediate refresh and check results

```sql
SYSTEM REFRESH VIEW top_customers_mv;

SELECT * FROM top_customers ORDER BY total_spent DESC;
```

You should see **Asha** (customer 1) leading with **165.00**, from the two electronics orders.

### Step 4: Check refresh status

```sql
SELECT database, view, status, last_success_time, next_refresh_time
FROM system.view_refreshes;
```

This system table tells you when the view last ran successfully and when it will run next — useful for debugging "why isn't my data updating."

### Step 5: Change the schedule

```sql
ALTER TABLE top_customers_mv MODIFY REFRESH EVERY 5 MINUTE;
```

You can change the refresh interval at any time without recreating the view.

### Step 6 (optional): APPEND mode for time-series snapshots

If instead of overwriting the latest totals each time, you wanted a **history** of totals over time (e.g., "what was total spend every 30 seconds"), you'd add `APPEND`:

```sql
-- Illustrative only — creates a growing snapshot log instead of replacing the table:
-- CREATE MATERIALIZED VIEW spend_snapshots_mv
-- REFRESH EVERY 30 SECOND APPEND TO spend_snapshots
-- AS SELECT now() AS ts, customer_id, sum(amount) AS total FROM orders GROUP BY customer_id;
```

**Takeaway for Part B:** refreshable MVs trade real-time freshness for the ability to run full `JOIN`s and complex logic on a schedule, instead of per-row.

---

## Part C — Cascading Materialized Views

**Goal:** chain rollups together — a monthly summary built from the daily rollup (Part A), and a yearly summary built from the monthly one — without ever touching the raw `orders` table again.

### Step 1: Monthly target table and MV, sourced from `daily_revenue`

```sql
CREATE TABLE monthly_revenue
(
    domain_month Date,
    category String,
    sumTotal AggregateFunction(sum, Decimal(18,2))
)
ENGINE = AggregatingMergeTree
ORDER BY (domain_month, category);

CREATE MATERIALIZED VIEW monthly_revenue_mv
TO monthly_revenue
AS
SELECT
    toStartOfMonth(day) AS domain_month,
    category,
    sumState(total_amount) AS sumTotal
FROM daily_revenue
GROUP BY domain_month, category;
```

**Key idea:** this MV's source is `daily_revenue` — the *target table from Part A* — not `orders` directly. Whenever `daily_revenue_mv` inserts a new block into `daily_revenue`, this monthly view automatically fires in turn.

**Why `sumState()` and `AggregateFunction`:** instead of storing a final summed number, this table stores a *partial aggregate state*. This is ClickHouse's way of allowing further merging later (in Step 2) without losing precision — it's a common pattern for multi-level rollups.

### Step 2: Yearly target table and MV, sourced from `monthly_revenue`

```sql
CREATE TABLE yearly_revenue
(
    year UInt16,
    category String,
    total UInt64
)
ENGINE = SummingMergeTree
ORDER BY (year, category);

CREATE MATERIALIZED VIEW yearly_revenue_mv
TO yearly_revenue
AS
SELECT
    toYear(domain_month) AS year,
    category,
    sumMerge(sumTotal) AS total
FROM monthly_revenue
GROUP BY year, category;
```

**Why `sumMerge()` here, not `sum()`:** the block forwarded from `monthly_revenue` contains partial `AggregateFunction` states (because `monthly_revenue` uses `AggregatingMergeTree`), not a plain final number. So this view has to first **merge** those partial states (`sumMerge`) before it can re-aggregate them into `yearly_revenue`.

### Step 3: Trigger the whole chain with one insert

```sql
INSERT INTO orders VALUES
    (5, 2, 'books', '2024-06-15 10:00:00', 20.00),
    (6, 2, 'books', '2024-06-20 10:00:00', 10.00),
    (7, 3, 'toys',  '2023-01-05 10:00:00', 50.00);
```

This single insert automatically flows through the entire chain:

```
orders → daily_revenue_mv → daily_revenue → monthly_revenue_mv → monthly_revenue → yearly_revenue_mv → yearly_revenue
```

You didn't have to run anything else manually — each MV triggers the next one as soon as its source table receives a new block.

### Step 4: Verify each stage

```sql
-- Daily
SELECT day, category, sum(total_amount) FROM daily_revenue GROUP BY day, category ORDER BY day;

-- Monthly (must use sumMerge — this column stores partial aggregate state)
SELECT domain_month, category, sumMerge(sumTotal) AS total
FROM monthly_revenue
GROUP BY domain_month, category
ORDER BY domain_month;

-- Yearly
SELECT year, category, sum(total) AS total
FROM yearly_revenue
GROUP BY year, category
ORDER BY year;
```

You should see `books` split across two months in 2024, and `toys` showing up in 2023 — each level correctly aggregating up from the level below it.

**Takeaway for Part C:** cascading MVs let you build a hierarchy of rollups (daily → monthly → yearly) where each level is derived automatically from the one below it, without ever re-reading the raw source table.

---

## Cleanup

```sql
DROP DATABASE shop;
```

---

## Summary — What Each Part Demonstrated

| Part | Type | How It Works | Best For |
|---|---|---|---|
| **A** | Incremental | Fires per-insert on `orders`; `SummingMergeTree` merges partial sums | Real-time, single-table rollups |
| **B** | Refreshable | Re-runs the full `JOIN` query on a timer via `REFRESH EVERY ...` | Complex JOINs, periodic batch-style reports |
| **C** | Cascading | One MV's target table becomes the next MV's source table | Multi-level rollups (daily → monthly → yearly) |

### Gotchas to Remember

- **Incremental MVs never see pre-existing rows** unless created with `POPULATE`.
- **In JOINs inside incremental MVs, only the left-most (source) table triggers the view** — inserting into the "joined" table won't trigger anything.
- **Cascading MVs forward the newly computed block, not the fully-merged final state** of the intermediate table — this is why you need `xState`/`xMerge` functions (like `sumState`/`sumMerge`) when chaining through `AggregatingMergeTree` tables.
- **Refreshable MVs trade real-time freshness for full JOIN support and simpler semantics** — check `system.view_refreshes` to see when they last ran.
- **Always alias every column explicitly in a JOIN-based MV's SELECT**, even ones that seem obvious — otherwise ClickHouse may keep a qualified name like `c.customer_id`, which won't match your target table's plain column name and will throw a `THERE_IS_NO_COLUMN` error.
