# ClickHouse Cloud Lab
### SQL Fundamentals · Joins & Aggregations · Filtering & Sorting Large Datasets

**Where to run this:** Your own ClickHouse Cloud service (e.g. "My first service") → **SQL console**.
**Duration:** ~60–75 minutes
**Data:** Generated entirely with SQL in Step 1 — no file uploads, no external URLs, no credentials needed. Fully self-contained, so it will work the same way every time you run it.

> ⚠️ If you see a banner like *"This service is idle. Wake your service to view table schema..."*, click it to wake the service before running any queries — an idle service needs a few seconds to spin back up.

---

## Agenda

| Time | Block |
|---|---|
| 0:00–0:10 | Setup: build a synthetic "large" dataset |
| 0:10–0:25 | Part 1 — SQL Fundamentals |
| 0:25–0:45 | Part 2 — Filtering & Sorting Large Datasets |
| 0:45–1:05 | Part 3 — Joins & Aggregations |
| 1:05–1:15 | Part 4 — Performance: proving it with `EXPLAIN` |

---

## Part 0 — Setup: Build the Dataset *(10 min)*

We'll create two tables that mimic a real e-commerce workload:
- **`customers`** — 100,000 customers
- **`orders`** — 5,000,000 orders referencing those customers

This is small by ClickHouse standards (which routinely handles billions of rows), but large enough to see real performance behavior and to make `LIMIT`, filtering, and sort-key habits matter.

### Step 1 — Create and populate `customers`

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
FROM numbers(100000);
```

### Step 2 — Create and populate `orders`

```sql
CREATE TABLE orders
(
    order_id    UInt64,
    customer_id UInt32,
    order_date  DateTime,
    amount      Decimal(10,2),
    status      LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (customer_id, order_date);

INSERT INTO orders
SELECT
    number + 1 AS order_id,
    (number % 100000) + 1 AS customer_id,
    toDateTime('2022-01-01 00:00:00') + toIntervalSecond(number * 3 + (rand() % 1000)) AS order_date,
    CAST(round((rand() % 200000) / 100.0, 2) AS Decimal(10,2)) AS amount,
    arrayElement(
        ['pending','shipped','delivered','cancelled','refunded'],
        (rand() % 5) + 1
    ) AS status
FROM numbers(5000000);
```

### Step 3 — Sanity check

```sql
SELECT count() FROM customers;
SELECT count() FROM orders;

SELECT * FROM customers LIMIT 5;
SELECT * FROM orders LIMIT 5;
```

**Note:** `orders` is sorted on disk by `(customer_id, order_date)`. That's not decorative — it's the physical layout ClickHouse uses to skip data, and it's the backbone of Part 2 and Part 4.

---

## Part 1 — SQL Fundamentals *(15 min)*

### Step 4 — Basic `SELECT`, column projection

```sql
SELECT customer_id, name, country
FROM customers
LIMIT 10;
```

### Step 5 — Filtering with `WHERE`

```sql
SELECT name, country, signup_date
FROM customers
WHERE country = 'India'
LIMIT 10;
```

### Step 6 — `ORDER BY` and `LIMIT` together

```sql
SELECT name, country, signup_date
FROM customers
WHERE country = 'Germany'
ORDER BY signup_date DESC
LIMIT 5;
```

### Step 7 — `DISTINCT` and simple expressions

```sql
SELECT DISTINCT country
FROM customers
ORDER BY country;

SELECT
    order_id,
    amount,
    amount * 0.9 AS amount_after_discount
FROM orders
LIMIT 10;
```

### Step 8 — `IN` lists and boolean combinations

```sql
SELECT order_id, customer_id, amount, status
FROM orders
WHERE status IN ('cancelled', 'refunded')
  AND amount > 500
LIMIT 20;
```

---

## Part 2 — Filtering & Sorting Large Datasets *(20 min)*

This is where ClickHouse stops feeling like "just SQL" and starts feeling like a columnar, sort-key-driven engine.

### Step 9 — Filter on the sort-key prefix (cheap)

`orders` is ordered by `(customer_id, order_date)`. Filtering on `customer_id` lets ClickHouse jump straight to the relevant data blocks instead of scanning the whole table.

```sql
SELECT order_id, order_date, amount, status
FROM orders
WHERE customer_id = 4242
ORDER BY order_date DESC
LIMIT 10;
```

**Note:** Because `order_date` is the *second* sort-key column, rows for `customer_id = 4242` are already stored in date order — this `ORDER BY` is nearly free.

### Step 10 — Filter on a non-sort-key column (more expensive)

```sql
SELECT order_id, customer_id, amount, status
FROM orders
WHERE status = 'refunded'
ORDER BY amount DESC
LIMIT 10;
```

**Note:** `status` isn't in the sort key, so ClickHouse can't skip blocks based on it the way it can for `customer_id` — it has to check `status` across a wider range of granules. We'll measure this difference directly in Part 4.

### Step 11 — Range filters (dates and numbers)

```sql
SELECT order_id, customer_id, order_date, amount
FROM orders
WHERE order_date >= '2022-06-01' AND order_date < '2022-07-01'
  AND amount BETWEEN 100 AND 500
ORDER BY order_date
LIMIT 20;
```

### Step 12 — `LIMIT` caps rows *returned*, not rows *scanned*

```sql
-- This still has to look at every matching row to know which 5 are "biggest" —
-- LIMIT only trims the final output.
SELECT order_id, customer_id, amount
FROM orders
WHERE status = 'pending'
ORDER BY amount DESC
LIMIT 5;
```

**Note:** A common misconception is that `LIMIT` makes a query cheap. It bounds the result set, not the scan. Keep this in mind before running `ORDER BY ... LIMIT` on an unfiltered table.

### Step 13 — Sorting a small aggregated result (cheap either way)

```sql
SELECT country, count() AS c
FROM customers
GROUP BY country
ORDER BY c DESC;
```

**Note:** Once data is aggregated down to a handful of rows, sort cost is irrelevant — the expensive part was the scan/aggregation that came before it.

---

## Part 3 — Joins & Aggregations *(20 min)*

### Step 14 — Basic aggregation: revenue per status

```sql
SELECT
    status,
    count()           AS num_orders,
    sum(amount)       AS total_amount,
    avg(amount)       AS avg_amount
FROM orders
GROUP BY status
ORDER BY total_amount DESC;
```

### Step 15 — Conditional aggregation ("pivot" without `PIVOT`)

```sql
SELECT
    customer_id,
    sum(status = 'delivered')  AS delivered_count,
    sum(status = 'cancelled')  AS cancelled_count,
    sum(status = 'refunded')   AS refunded_count
FROM orders
WHERE customer_id IN (1, 2, 3, 4, 5)
GROUP BY customer_id
ORDER BY customer_id;
```

**Note:** `sum(<boolean expression>)` is a clean ClickHouse idiom — booleans coerce to `0`/`1`, so this counts matching rows per group without a `CASE WHEN`.

### Step 16 — `HAVING`: filter on aggregated results

```sql
SELECT
    customer_id,
    count()     AS num_orders,
    sum(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING num_orders > 60
ORDER BY total_spent DESC
LIMIT 20;
```

### Step 17 — Join customers and orders: revenue by country

```sql
SELECT
    c.country,
    count()            AS num_orders,
    sum(o.amount)      AS total_revenue,
    avg(o.amount)      AS avg_order_value
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.customer_id
GROUP BY c.country
ORDER BY total_revenue DESC;
```

**Note:** Both `orders` and `customers` are already at their natural grain here (one row per order, one row per customer), so this is a straightforward 1-to-many join — no pre-aggregation needed before joining.

### Step 18 — Filter-before-join for a heavier query

When you only care about a subset of customers, filter *before* the join so the join itself works on a smaller table.

```sql
WITH high_value_customers AS
(
    SELECT customer_id, sum(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING total_spent > 5000
)
SELECT
    c.name,
    c.country,
    h.total_spent
FROM high_value_customers AS h
INNER JOIN customers AS c ON h.customer_id = c.customer_id
ORDER BY h.total_spent DESC
LIMIT 20;
```

**Note:** `high_value_customers` is computed and reduced to a small set *first*; the join only has to match that small set against `customers`, not the full 5,000,000-row `orders` table.

### Step 19 — Approximate vs. exact distinct counts

```sql
SELECT
    country,
    uniq(customer_id)      AS approx_distinct_customers,  -- fast, ~1–2% error
    uniqExact(customer_id) AS exact_distinct_customers     -- exact, more CPU/memory
FROM customers
GROUP BY country
ORDER BY country;
```

**Note:** At real-world scale (billions of rows), `uniq()` is usually the right default. Reach for `uniqExact()` only when you need an exact number and can afford the extra cost.

### Step 20 — Window functions: running total per customer

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

---

## Part 4 — Performance: Proving It With `EXPLAIN` *(10 min)*

### Step 21 — Compare a sort-key filter vs. a non-sort-key filter

```sql
-- Filters on customer_id — the FIRST sort-key column
EXPLAIN indexes = 1
SELECT count() FROM orders WHERE customer_id = 4242;

-- Filters on status — NOT part of the sort key
EXPLAIN indexes = 1
SELECT count() FROM orders WHERE status = 'refunded';
```

**Note:** Compare the `Granules` line in each plan. The `customer_id` filter should read dramatically fewer granules than the `status` filter, even though both target the same table — this is the sort key from Step 2 paying off in a way you can measure, not just take on faith.

### Step 22 — Query-safety habits, even on your own service

Even outside a shared Playground, it's good practice to bound heavier ad-hoc queries — especially while exploring a table you don't know well yet:

```sql
SELECT status, count() AS c
FROM orders
GROUP BY status
LIMIT 100
SETTINGS
    max_execution_time = 30,
    max_rows_to_read = 1000000000;
```

---

## Wrap-Up

**What you covered:**
- Core SQL: `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, `DISTINCT`, `IN`
- Why the MergeTree `ORDER BY` (sort key) makes some filters far cheaper than others
- `GROUP BY`, conditional aggregation, `HAVING`, `uniq()` vs `uniqExact()`
- Joining pre-aggregated data, filter-before-join, and a basic window function
- Reading `EXPLAIN indexes = 1` to confirm performance claims instead of assuming them

**Cleanup (optional):**

```sql
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customers;
```

**Where to go next:**
- Re-run Part 4's `EXPLAIN` comparisons against a table with a *different* `ORDER BY` to see how the sort key choice changes which filters are "cheap."
- Try loading one of ClickHouse Cloud's built-in sample datasets (via the console's sample-data import) and repeat Parts 1–3 against real data instead of synthetic data.
