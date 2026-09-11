# ClickHouse Window Functions — Full Lab Guide

A step-by-step guide to understanding window functions in ClickHouse, using a simple sales dataset.

---

## Step 0: Setup

Create a table and load sample data.

```sql
CREATE TABLE sales
(
    region      String,
    salesperson String,
    sale_date   Date,
    amount      UInt32
)
ENGINE = MergeTree
ORDER BY (region, sale_date);

INSERT INTO sales (region, salesperson, sale_date, amount) VALUES
  ('East', 'Alice',  '2026-01-01', 100),
  ('East', 'Alice',  '2026-01-02', 150),
  ('East', 'Bob',    '2026-01-01', 200),
  ('East', 'Bob',    '2026-01-03', 50),
  ('West', 'Carol',  '2026-01-01', 300),
  ('West', 'Carol',  '2026-01-02', 120),
  ('West', 'Dave',   '2026-01-01', 90),
  ('West', 'Dave',   '2026-01-02', 60);
```

There are 8 rows total: 4 in the East region (Alice, Bob) and 4 in the West region (Carol, Dave).

Check the data:

```sql
SELECT * FROM sales ORDER BY region, salesperson, sale_date;
```

---

## Step 1: Window Functions vs GROUP BY

This is the single most important idea in the whole lab.

**GROUP BY** squashes many rows into one row per group. You lose the detail of individual rows.

```sql
SELECT region, sum(amount) AS total
FROM sales
GROUP BY region;
```

Result: only **2 rows** (East, West). The individual sales are gone — you can no longer see who sold what.

**A window function** computes a value across a group of related rows, but keeps every original row intact.

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    sum(amount) OVER (PARTITION BY region) AS region_total
FROM sales
ORDER BY region, salesperson, sale_date;
```

Result: still **8 rows**. Every row now also shows the total for its own region, right alongside it.

**Core idea:** window functions add extra context to each row instead of collapsing rows together.

**General syntax shape:**

```sql
<window_function>(...) OVER (
    PARTITION BY <column(s)>   -- optional: splits rows into groups
    ORDER BY <column(s)>       -- optional: defines the order of rows
    ROWS/RANGE BETWEEN ...     -- optional: defines exactly which rows count
)
```

Every window function in this lab is a variation of this same shape.

---

## Step 2: Ranking Functions

**Goal:** Within each region, rank salespeople by their highest single sale.

```sql
SELECT
    region,
    salesperson,
    amount,
    row_number() OVER (PARTITION BY region ORDER BY amount DESC) AS rn,
    rank()       OVER (PARTITION BY region ORDER BY amount DESC) AS rnk,
    dense_rank() OVER (PARTITION BY region ORDER BY amount DESC) AS drnk
FROM sales
ORDER BY region, amount DESC;
```

### What happens when two rows tie on `amount`?

| Function | Behavior on ties |
|---|---|
| `row_number()` | Always gives unique numbers (1, 2, 3...) even for ties. Which tied row gets which number is arbitrary unless you add a tiebreaker column. |
| `rank()` | Gives the **same** rank to tied rows, then **skips** the next number (e.g. 1, 1, 3). |
| `dense_rank()` | Gives the same rank to tied rows but does **not** skip the next number (e.g. 1, 1, 2). |

### Practical use: get the top sale per region

```sql
SELECT * FROM (
    SELECT
        region, salesperson, sale_date, amount,
        row_number() OVER (PARTITION BY region ORDER BY amount DESC) AS rn
    FROM sales
)
WHERE rn = 1;
```

**Why the subquery?** You cannot filter directly on a window function's result in the same `SELECT` — `WHERE` runs before window functions are calculated. So you compute `rn` in an inner query first, then filter on it from the outside.

---

## Step 3: Running / Cumulative Totals

**Goal:** Show a running total of sales per region, ordered by date.

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    sum(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date
    ) AS running_total
FROM sales
ORDER BY region, sale_date;
```

### The key concept

When `ORDER BY` is present inside `OVER(...)` but no frame is written, ClickHouse automatically uses this default frame:

```
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

This means each row's window is "everything from the start of the partition up to and including this row" — which is exactly a running/cumulative total.

**Compare this to Step 1's `region_total`,** which had no `ORDER BY` inside `OVER(...)`. There, the default frame is the *entire partition*, so every row showed the same, unchanging total.

### This contrast is the most important takeaway of the lab:

| Syntax | Behavior |
|---|---|
| `OVER (PARTITION BY x)` | Whole-partition aggregate — same value for every row |
| `OVER (PARTITION BY x ORDER BY y)` | Running/cumulative aggregate — value grows row by row |

### Other running aggregates you can try the same way:

```sql
avg(amount)   OVER (PARTITION BY region ORDER BY sale_date) AS running_avg
count()       OVER (PARTITION BY region ORDER BY sale_date) AS running_count
max(amount)   OVER (PARTITION BY region ORDER BY sale_date) AS running_max
```

---

## Step 4: `lag()` and `lead()`

**Goal:** For each salesperson, compare today's sale to their previous sale.

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    lag(amount, 1) OVER (
        PARTITION BY salesperson
        ORDER BY sale_date
    ) AS previous_amount,
    amount - lag(amount, 1) OVER (
        PARTITION BY salesperson
        ORDER BY sale_date
    ) AS change_from_previous
FROM sales
ORDER BY salesperson, sale_date;
```

### Notes

- `lag(col, N)` looks back N rows within the partition (default N = 1).
- `lead(col, N)` looks forward N rows within the partition.
- The **first row** in each partition has no previous row, so `previous_amount` and `change_from_previous` will be `NULL` there.
- You can supply a default value instead of `NULL`:

```sql
lag(amount, 1, 0) OVER (...)   -- returns 0 instead of NULL
```

### To see the next sale instead, use `lead()`:

```sql
lead(amount, 1) OVER (PARTITION BY salesperson ORDER BY sale_date)
```

---

## Step 5: Moving Average with an Explicit Frame

**Goal:** Compute a 2-row moving average of `amount` per region, ordered by date (current row + previous row).

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    avg(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date
        ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS moving_avg_2
FROM sales
ORDER BY region, sale_date;
```

### Frame syntax reference

| Frame | Meaning |
|---|---|
| `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` | Current row + 1 row before it (2-row window) |
| `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` | Current row + 2 rows before it (3-row window) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | From start of partition to current row (same as Step 3's default — running total) |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | Current row to end of partition (reverse running total) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | The entire partition (same as Step 1's default — whole-partition aggregate) |

This table makes it clear: the "defaults" you saw in Steps 1 and 3 aren't magic — they're just two specific frames from this same list, applied automatically depending on whether `ORDER BY` is present.

---

## Step 6: Wrap-Up Challenge

**Goal:** For each region, show each salesperson's sale, what percentage of the region's total that sale represents, and whether it was an increase or decrease compared to that salesperson's previous sale.

Try writing it yourself first, then check below.

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    round(100.0 * amount / sum(amount) OVER (PARTITION BY region), 1)
        AS pct_of_region_total,
    amount - lag(amount, 1, amount) OVER (
        PARTITION BY salesperson ORDER BY sale_date
    ) AS change_vs_previous
FROM sales
ORDER BY region, salesperson, sale_date;
```

### How this combines everything you learned

- **`pct_of_region_total`** reuses the Step 1 pattern: `sum(amount) OVER (PARTITION BY region)` with no `ORDER BY`, so it gives a fixed, whole-region total to divide each row's amount by.
- **`change_vs_previous`** reuses the Step 4 `lag()` pattern, but the default value passed is `amount` itself (not `0`). This makes the first row's "change" come out to `0` (`amount - amount`), instead of `NULL`.

---

## Summary Table

| Step | Concept | Syntax Added |
|---|---|---|
| 1 | Window functions keep all rows; GROUP BY collapses them | `OVER (PARTITION BY x)` |
| 2 | Ranking rows within a group | `+ ORDER BY y` on ranking functions |
| 3 | Running totals — adding `ORDER BY` changes the default frame | `+ ORDER BY y` on aggregate functions |
| 4 | Looking at neighboring rows | `lag()` / `lead()` with optional default value |
| 5 | Controlling exactly which rows are included | `ROWS BETWEEN ... AND ...` |
| 6 | Combining multiple window functions in one query | All of the above together |

**The one rule to remember:**

```
OVER (PARTITION BY x)              → same value for every row in the group
OVER (PARTITION BY x ORDER BY y)   → value builds up row by row
OVER (... ROWS BETWEEN a AND b)    → you control exactly how many rows are included
```
