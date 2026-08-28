# ClickHouse Cloud Lab
### Window Functions · Materialized Views · Array & Nested Data · Approximate Aggregates

**Where to run this:** Your own ClickHouse Cloud service → **SQL console**.
**Data:** Generated entirely with SQL in Setup — no file uploads, no external URLs, no credentials. Fully self-contained.

> ⚠️ If your service shows *"This service is idle. Wake your service..."*, click it and wait a few seconds before running queries.

---

## Setup — Build the Dataset

### Step 0 — Clean slate

This lab uses a wider `orders` schema than earlier labs (it adds `tags` and a nested `items` structure), so drop any existing tables first:

```sql
DROP TABLE IF EXISTS mv_high_value_orders;
DROP TABLE IF EXISTS high_value_orders;
DROP TABLE IF EXISTS mv_daily_country_revenue;
DROP TABLE IF EXISTS daily_country_revenue;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customers;
```

### Step 1 — `customers`

```sql
CREATE TABLE customers
(
    customer_id UInt32,
    name        String,
    country     LowCardinality(String),
    signup_date Date
)
ENGINE = MergeTree
ORDER BY customer_id;

INSERT INTO customers
SELECT
    number + 1 AS customer_id,
    concat('Customer_', toString(number + 1)) AS name,
    arrayElement(
        ['USA','UK','India','Germany','Brazil','Japan','Canada','Australia'],
        (number % 8) + 1
    ) AS country,
    toDate('2019-01-01') + toIntervalDay(number % 1800) AS signup_date
FROM numbers(20000);
```

### Step 2 — `orders`, with a `tags` array and a `items` Nested structure

```sql
CREATE TABLE orders
(
    order_id    UInt64,
    customer_id UInt32,
    order_date  DateTime,
    amount      Decimal(10,2),
    status      LowCardinality(String),
    tags        Array(String),
    items       Nested
    (
        product  String,
        quantity UInt32,
        price    Decimal(10,2)
    )
)
ENGINE = MergeTree
ORDER BY (customer_id, order_date);
```

`items` is a **Nested** column: internally it's three arrays (`items.product`, `items.quantity`, `items.price`) that are kept the same length per row — one entry per line item in the order.

```sql
INSERT INTO orders
(
    order_id, customer_id, order_date, amount, status, tags,
    items.product, items.quantity, items.price
)
SELECT
    number + 1 AS order_id,
    (number % 20000) + 1 AS customer_id,
    toDateTime('2022-01-01 00:00:00') + toIntervalSecond(number * 5 + (rand() % 1000)) AS order_date,
    CAST(round((rand() % 200000) / 100.0, 2) AS Decimal(10,2)) AS amount,
    arrayElement(['pending','shipped','delivered','cancelled','refunded'], (rand() % 5) + 1) AS status,
    arrayFilter(x -> (rand() % 2) = 0, ['sale','gift','express','bulk','first_order']) AS tags,
    arrayMap(i -> concat('Product_', toString(1 + (rand() % 50))), idx) AS `items.product`,
    arrayMap(i -> 1 + (rand() % 5), idx)                                AS `items.quantity`,
    arrayMap(i -> CAST(round((rand() % 10000) / 100.0, 2) AS Decimal(10,2)), idx) AS `items.price`
FROM
(
    SELECT number, range(1 + (rand() % 4)) AS idx
    FROM numbers(200000)
);
```

**Note:** The `idx` array is computed once per row in the subquery, then reused by all three `arrayMap` calls. This is deliberate — if you called `range(...)` separately for each of `items.product`, `items.quantity`, `items.price`, each would get its own random length and ClickHouse would reject the insert ("Nested arrays must have same length").

### Step 3 — Sanity check

```sql
SELECT count() FROM customers;
SELECT count() FROM orders;

SELECT order_id, tags, items.product, items.quantity, items.price
FROM orders
LIMIT 5;
```

---

## Part 1 — Window Functions

### Step 4 — Running total per customer

```sql
SELECT
    customer_id,
    order_date,
    amount,
    sum(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total
FROM orders
WHERE customer_id IN (10, 20, 30)
ORDER BY customer_id, order_date;
```

### Step 5 — Ranking within a group

```sql
SELECT
    customer_id,
    order_date,
    amount,
    row_number() OVER (PARTITION BY customer_id ORDER BY order_date)      AS order_seq,
    rank()       OVER (PARTITION BY toDate(order_date) ORDER BY amount DESC) AS rank_in_day
FROM orders
WHERE customer_id IN (10, 20, 30)
ORDER BY customer_id, order_date;
```

**Note:** `row_number()`, `rank()`, and `dense_rank()` behave exactly as in standard SQL:2003 window syntax — a 1:1 transfer if you know Postgres or Snowflake.

### Step 6 — Moving average with a frame

```sql
SELECT
    customer_id,
    order_date,
    amount,
    avg(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3
FROM orders
WHERE customer_id IN (10, 20, 30)
ORDER BY customer_id, order_date;
```

### Step 7 — Previous/next row with `lagInFrame` / `leadInFrame`

```sql
SELECT
    customer_id,
    order_date,
    amount,
    lagInFrame(amount, 1)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_amount,
    leadInFrame(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_amount
FROM orders
WHERE customer_id IN (10, 20, 30)
ORDER BY customer_id, order_date;
```

**Note:** `lagInFrame` / `leadInFrame` are ClickHouse's window-function equivalents of `LAG`/`LEAD`. They only look within the current window frame — widen the frame (or drop the `ROWS BETWEEN` clause entirely) if a lookup seems to return `NULL`/`0` unexpectedly.

---

## Part 2 — Materialized Views

A ClickHouse materialized view is really a standing `INSERT ... SELECT` trigger: whenever rows are inserted into the source table, the view's query runs against *just those new rows* and the result is inserted into a target table. It does **not** re-scan the whole source table on every insert, and it does **not** automatically reflect updates/deletes to already-inserted source rows.

### Step 8 — A simple filtering view

Keep a live, always-up-to-date table of high-value orders.

```sql
CREATE TABLE high_value_orders
(
    order_id    UInt64,
    customer_id UInt32,
    order_date  DateTime,
    amount      Decimal(10,2)
)
ENGINE = MergeTree
ORDER BY (customer_id, order_date);

CREATE MATERIALIZED VIEW mv_high_value_orders
TO high_value_orders
POPULATE
AS
SELECT order_id, customer_id, order_date, amount
FROM orders
WHERE amount > 1500;
```

`POPULATE` backfills the view from existing rows at creation time. (In production, on a table taking live writes, `POPULATE` can race with concurrent inserts — fine here since nothing else is writing to `orders` yet.)

```sql
SELECT count() FROM high_value_orders;
```

### Step 9 — An incremental aggregation view

This is the more powerful pattern: pre-aggregate as data arrives, instead of aggregating billions of rows on every query. It needs an `AggregatingMergeTree` target table and `*State` / `*Merge` function pairs.

```sql
CREATE TABLE daily_country_revenue
(
    day          Date,
    country      LowCardinality(String),
    revenue      AggregateFunction(sum, Decimal(10,2)),
    orders_count AggregateFunction(count)
)
ENGINE = AggregatingMergeTree
ORDER BY (day, country);

CREATE MATERIALIZED VIEW mv_daily_country_revenue
TO daily_country_revenue
POPULATE
AS
SELECT
    toDate(o.order_date) AS day,
    c.country             AS country,
    sumState(o.amount)    AS revenue,
    countState()          AS orders_count
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.customer_id
GROUP BY day, country;
```

Query it with the matching `*Merge` combinators:

```sql
SELECT
    day,
    country,
    sumMerge(revenue)      AS total_revenue,
    countMerge(orders_count) AS total_orders
FROM daily_country_revenue
GROUP BY day, country
ORDER BY day, country
LIMIT 20;
```

**Note:** `revenue` and `orders_count` are stored as intermediate aggregation *states*, not final numbers — that's what lets ClickHouse merge partial results correctly across background-merged parts. `sumMerge`/`countMerge` finish the aggregation at query time.

### Step 10 — Prove it updates live

```sql
INSERT INTO orders (order_id, customer_id, order_date, amount, status)
VALUES (9999999, 42, now(), 2500.00, 'pending');
```

(`tags` and `items.*` weren't listed, so they default to empty arrays — that's fine for this check.)

```sql
-- Should now include order 9999999
SELECT * FROM high_value_orders WHERE order_id = 9999999;

-- Should reflect the new order's amount for today / customer 42's country
SELECT day, country, sumMerge(revenue) AS total_revenue, countMerge(orders_count) AS total_orders
FROM daily_country_revenue
WHERE day = today()
GROUP BY day, country
ORDER BY country;
```

---

## Part 3 — Array & Nested Data Functions

### Step 11 — Basic array functions on `tags`

```sql
SELECT order_id, tags, length(tags) AS num_tags
FROM orders
LIMIT 10;

-- Does this order have a specific tag?
SELECT order_id, tags
FROM orders
WHERE has(tags, 'gift')
LIMIT 10;

-- Any of / all of
SELECT order_id, tags
FROM orders
WHERE hasAny(tags, ['sale', 'express'])
LIMIT 10;
```

### Step 12 — Flattening an array: `arrayJoin()` vs. `ARRAY JOIN`

```sql
-- arrayJoin() as an expression: one output row per tag
SELECT arrayJoin(tags) AS tag, count() AS c
FROM orders
GROUP BY tag
ORDER BY c DESC;

-- ARRAY JOIN clause: usually clearer once you need more than one column
SELECT tag, count() AS c
FROM orders
ARRAY JOIN tags AS tag
GROUP BY tag
ORDER BY c DESC;
```

### Step 13 — Working with `items` (Nested = arrays kept in sync)

```sql
SELECT order_id, items.product, items.quantity, items.price
FROM orders
LIMIT 5;
```

Flatten the nested line items into one row per item, in lockstep:

```sql
SELECT order_id, product, quantity, price
FROM orders
ARRAY JOIN items.product AS product, items.quantity AS quantity, items.price AS price
LIMIT 20;
```

Now that it's flattened, aggregate it like any other table:

```sql
SELECT
    product,
    sum(quantity)         AS total_units_sold,
    sum(quantity * price) AS total_revenue
FROM orders
ARRAY JOIN items.product AS product, items.quantity AS quantity, items.price AS price
GROUP BY product
ORDER BY total_revenue DESC
LIMIT 20;
```

### Step 14 — Per-order line-item math without flattening

Sometimes you want an answer *per order*, not per line item — `arrayMap`/`arraySum` operate on the arrays directly.

```sql
SELECT
    order_id,
    arraySum(items.quantity) AS total_items,
    arraySum(
        arrayMap((q, p) -> q * p, items.quantity, items.price)
    ) AS items_total
FROM orders
LIMIT 10;
```

**Note:** `arrayMap` with a two-argument lambda `(q, p) -> ...` walks two same-length arrays in parallel — exactly what a Nested column guarantees you.

---

## Part 4 — Approximate Aggregate Functions

At billion-row scale, exact distinct counts and exact quantiles get expensive. ClickHouse's approximate functions trade a small, bounded error for large speed/memory wins.

### Step 15 — Distinct counts: `uniq()` vs. `uniqExact()`

```sql
SELECT
    uniq(customer_id)      AS approx_distinct_customers,   -- HyperLogLog-family, fast
    uniqExact(customer_id) AS exact_distinct_customers      -- exact, more CPU/memory
FROM orders;
```

```sql
-- Distinct products actually sold, after flattening the nested items
SELECT
    uniqCombined(product) AS approx_distinct_products,  -- adaptive: exact for small cardinalities, HLL beyond a threshold
    uniqExact(product)    AS exact_distinct_products
FROM orders
ARRAY JOIN items.product AS product;
```

### Step 16 — Quantiles: `quantile()` vs. `quantileExact()`

```sql
SELECT
    quantile(0.5)(amount)               AS median_approx,
    quantileExact(0.5)(amount)          AS median_exact,
    quantiles(0.25, 0.5, 0.75, 0.95)(amount) AS approx_quartiles_and_p95
FROM orders;
```

**Note:** `quantile()` (and its relatives like `quantileTDigest()`) use sketch algorithms — approximate but far cheaper on large datasets. `quantileExact()` sorts the actual values, which is precise but memory-heavy at scale.

### Step 17 — Most frequent values: `topK()`

```sql
-- Most common order statuses, without a full GROUP BY
SELECT topK(3)(status) AS top_statuses
FROM orders;

-- Most frequently ordered products, after flattening
SELECT topK(5)(product) AS top_products
FROM orders
ARRAY JOIN items.product AS product;
```

**Note:** `topK(N)(col)` returns its top-N approximation using a fixed-size sketch, so memory usage doesn't grow with the number of distinct values — useful when `col` could have millions of distinct values but you only care about the handful at the top.

---

## Wrap-Up

**What you covered:**
- Window functions: running totals, ranking, moving averages, `lagInFrame`/`leadInFrame`
- Materialized views: a simple filtering view, and an incremental `AggregatingMergeTree` view with `*State`/`*Merge` functions — plus proof that both update live on insert
- Array functions (`has`, `hasAny`, `length`, `arrayJoin`, `arrayMap`, `arraySum`) and Nested data, including flattening with `ARRAY JOIN`
- Approximate aggregates: `uniq`/`uniqCombined` vs. `uniqExact`, `quantile` vs. `quantileExact`, and `topK`

**Cleanup (optional):**

```sql
DROP TABLE IF EXISTS mv_high_value_orders;
DROP TABLE IF EXISTS high_value_orders;
DROP TABLE IF EXISTS mv_daily_country_revenue;
DROP TABLE IF EXISTS daily_country_revenue;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customers;
```

**Where to go next:**
- Try a `ReplacingMergeTree` or `SummingMergeTree` target table instead of `AggregatingMergeTree`, and compare when each pattern fits.
- Compare `quantileTDigest()` and `quantileExact()` timing on a larger `orders` table (bump `numbers(...)` up in Setup) to see the approximation payoff grow with data size.
