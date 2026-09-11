# ClickHouse Lab Guide: Filtering, Aggregation, and Sorting

This lab explains how ClickHouse actually processes a query internally — in what order filtering, grouping, and sorting happen, and why that order matters for performance.

---

## Step 0: Setup — Create a Large Table

```sql
CREATE TABLE trips
(
    trip_id         UInt32,
    passenger_count UInt8,
    trip_distance   Float32,
    fare_amount     Decimal32(2),
    pickup_date     Date
)
ENGINE = MergeTree
ORDER BY tuple();   -- deliberately no primary key for now

INSERT INTO trips
SELECT
    number AS trip_id,
    (number % 6) + 1 AS passenger_count,
    (number % 50) + 1 AS trip_distance,
    ((number % 200) + 10) AS fare_amount,
    toDate('2026-01-01') + (number % 90) AS pickup_date
FROM numbers(2000000);
```

**What's happening here:**

- `numbers(2000000)` generates 2 million rows out of thin air — a common trick for building test data without a real source.
- `ORDER BY tuple()` means **no sorting key at all**. This is done on purpose, so that later in the lab you can compare "no primary key" vs "has a primary key" and see the performance difference for yourself.
- Each column uses `number % X` to produce realistic-looking but random-ish spread values (e.g. `trip_distance` cycles between 1 and 50).

**Sanity check:**

```sql
SELECT count() FROM trips;
SELECT * FROM trips LIMIT 5;
```

---

## Step 1: The 4 Stages of a Query Pipeline (Concept)

Every time ClickHouse runs a query, it internally goes through up to four stages, always in this order:

1. **Read** the data from the table
2. **Filter** it (`WHERE`)
3. **Aggregate** it (`GROUP BY`, `sum()`, `avg()`, etc.)
4. **Sort** it (`ORDER BY`)

Understanding this order is the whole point of this lab. Once you know this, you can predict how fast or slow a query will be just by looking at it.

---

## Step 2: See the Pipeline with EXPLAIN

Take a query that has all three stages together — filtering, aggregation, and sorting:

```sql
SELECT
    passenger_count,
    avg(fare_amount) AS avg_fare
FROM trips
WHERE trip_distance > 25
GROUP BY passenger_count
ORDER BY avg_fare DESC;
```

Run `EXPLAIN` on it:

```sql
EXPLAIN indexes = 1
SELECT
    passenger_count,
    avg(fare_amount) AS avg_fare
FROM trips
WHERE trip_distance > 25
GROUP BY passenger_count
ORDER BY avg_fare DESC;
```

The output looks roughly like this:

```
Expression (Projection)
  Sorting (Sorting for ORDER BY)
    Aggregating
      Expression (Before GROUP BY)
        Filter (WHERE)
          ReadFromMergeTree (trips)
```

### The most important trick in this lab

**Read the EXPLAIN output from BOTTOM to TOP** — that is the real execution order:

| Order | Stage | What it does |
|---|---|---|
| 1st | `ReadFromMergeTree` | Data is read from the table |
| 2nd | `Filter (WHERE)` | `trip_distance > 25` filtering happens |
| 3rd | `Aggregating` | `GROUP BY passenger_count` happens |
| 4th | `Sorting` | Finally, sort by `avg_fare DESC` |

Remember: **EXPLAIN output should always be read bottom-to-top, not top-to-bottom.**

---

## Step 3: Filtering — Reduce Rows First

Filtering is the first real work ClickHouse does, and it's the most powerful lever for speed — because everything after it (aggregation, sorting) only has to deal with whatever survives the filter.

**Count without any filter:**

```sql
SELECT count() FROM trips;
```

**Count with a filter:**

```sql
SELECT count() FROM trips WHERE trip_distance > 25;
```

Since `trip_distance` is spread evenly from 1 to 50, roughly **half** the rows will match `> 25`. Compare the two counts to confirm this.

**Now compare execution time** (turn off the filesystem cache first, so the comparison is fair and not skewed by cached results):

```sql
SET enable_filesystem_cache = 0;

-- Full table scan
SELECT avg(fare_amount) FROM trips;

-- With filter
SELECT avg(fare_amount) FROM trips WHERE trip_distance > 25;
```

**Lesson:** The more rows a filter removes, the faster everything downstream runs — aggregation and sorting simply have less data left to chew through.

---

## Step 4: Aggregation — What Happens After Filtering

Aggregation (`GROUP BY`, `sum()`, `avg()`, `count()`, etc.) only ever runs on the rows that survived the filter stage — never on the full table.

**Simple filter, no aggregation:**

```sql
SELECT count() FROM trips WHERE trip_distance > 25;
```

**Filter + aggregation:**

```sql
SELECT
    passenger_count,
    count() AS trip_count,
    avg(fare_amount) AS avg_fare,
    sum(trip_distance) AS total_distance
FROM trips
WHERE trip_distance > 25
GROUP BY passenger_count
ORDER BY trip_count DESC;
```

The second query takes a bit longer because, after filtering, it still has to run `GROUP BY`, `avg()`, and `sum()` on the remaining rows. But crucially, this work happens **only on the filtered subset**, not on all 2 million rows.

**Check the EXPLAIN output to see the structure:**

```sql
EXPLAIN indexes = 1
SELECT
    passenger_count,
    count() AS trip_count,
    avg(fare_amount) AS avg_fare
FROM trips
WHERE trip_distance > 25
GROUP BY passenger_count;
```

```
Expression
  Aggregating          <- GROUP BY + count()/avg() happen here
    Expression (Before GROUP BY)
      Filter (WHERE)    <- filtering has already happened
        ReadFromMergeTree (trips)
```

Again, read bottom to top: filtering happens first, and only its output feeds into `Aggregating`.

---

## Step 5: Sorting — Happens Last

Add an `ORDER BY` and see where sorting fits into the pipeline:

```sql
EXPLAIN indexes = 1
SELECT
    passenger_count,
    avg(fare_amount) AS avg_fare
FROM trips
WHERE trip_distance > 25
GROUP BY passenger_count
ORDER BY avg_fare DESC;
```

The `Sorting` step appears at the **top** of the EXPLAIN output, which — remember, read bottom-to-top — means it's the **last** step to actually run.

**What this means in practice:**

> Sorting only happens after the data has already been filtered and aggregated.

This is why sorting is usually cheap, even on huge tables — by the time it runs, filtering and `GROUP BY` have already shrunk the dataset down to a small number of rows. **If a query feels slow, the real bottleneck is almost always the filter or aggregation stage — not sorting.**

**Try sorting alone** (no aggregation, just raw rows in order):

```sql
SELECT trip_id, fare_amount
FROM trips
ORDER BY fare_amount DESC
LIMIT 10;
```

This *can* still be slow, because ClickHouse may need to sort all 2 million rows before picking the top 10. This is exactly why `LIMIT` matters — it lets ClickHouse optimize for "just give me the top N" instead of fully sorting everything.

---

## Step 6: Use a Primary Key to Speed Up Filtering Itself

So far, filtering has meant "read everything, then throw away what doesn't match." A good primary key changes that — it lets ClickHouse skip reading irrelevant data in the first place.

**Create a new table, this time with a real primary key (via ORDER BY):**

```sql
CREATE TABLE trips_with_pk
(
    trip_id         UInt32,
    passenger_count UInt8,
    trip_distance   Float32,
    fare_amount     Decimal32(2),
    pickup_date     Date
)
ENGINE = MergeTree
ORDER BY (pickup_date, passenger_count);

INSERT INTO trips_with_pk SELECT * FROM trips;
```

**Run the same filter query against both tables and compare:**

```sql
-- OLD table (no primary key) — full table scan
EXPLAIN indexes = 1
SELECT count() FROM trips
WHERE pickup_date = '2026-02-01';

-- NEW table (has a primary key) — only relevant granules
EXPLAIN indexes = 1
SELECT count() FROM trips_with_pk
WHERE pickup_date = '2026-02-01';
```

**What you'll see:**

- The `trips_with_pk` query's EXPLAIN output will include an **"Indexes"** section showing something like `Granules: X/Y` — meaning ClickHouse only had to read X granules out of the total Y, instead of scanning the whole table.
- The plain `trips` query won't show this at all, since it has no primary key to check against — it has no choice but to scan everything.

**Why this matters:**

> A primary key makes the FILTER stage itself faster. ClickHouse knows in advance which granules could possibly contain matching data, without having to read the entire table to find out.

This connects back to Step 3: filtering is already the most important lever for speed, and a well-chosen primary key makes filtering itself dramatically cheaper.

---

## Wrap-Up Challenge

Try writing these two queries yourself, then use `EXPLAIN` to verify the Filter → Aggregate → Sort order.

**1. Total fare per passenger group, filtered by distance, sorted by total fare:**

> "Show total fare_amount per passenger_count group, but only for trips where distance is greater than 10, sorted by total fare in descending order."

**2. Check granule pruning on the primary-key table:**

> "On the trips_with_pk table, show data for just one day (e.g. '2026-01-15') and check how many Granules are read in the EXPLAIN output."

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Query pipeline order | Read → Filter → Aggregate → Sort, always in that order |
| Reading EXPLAIN | Always read from **bottom to top** — that's the real execution order |
| Filtering | Happens first; the more rows it removes, the less work everything after it has to do |
| Aggregation | Only runs on rows that survived the filter, never the whole table |
| Sorting | Happens last, after filtering and aggregation have already shrunk the data |
| LIMIT | Helps avoid sorting the entire dataset when you only need the top N rows |
| Primary Key | Lets the filter stage itself skip irrelevant data (fewer granules read), instead of just filtering after a full scan |
