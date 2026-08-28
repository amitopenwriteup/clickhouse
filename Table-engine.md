# Lab: ClickHouse Table Engines
### MergeTree Family • ReplacingMergeTree • SummingMergeTree / AggregatingMergeTree • Distributed

**Environment:** ClickHouse Cloud (SQL console or `clickhouse-client` connected to your Cloud service)

---

## 0. Prerequisites & Setup

1. Log in to your ClickHouse Cloud service and open the **SQL console**, or connect via `clickhouse-client --host <your-host> --secure --password`.
2. Create a dedicated database for this lab so cleanup is easy:

```sql
CREATE DATABASE IF NOT EXISTS engines_lab;
USE engines_lab;
```

> **Cloud note:** ClickHouse Cloud services already run on a highly-available, replicated, shared-storage architecture behind the scenes. When you write `ENGINE = MergeTree()` in Cloud, it is automatically provisioned as a `SharedMergeTree` / `ReplicatedMergeTree`-equivalent under the hood — you get replication for free without specifying ZooKeeper/Keeper paths yourself. This is different from a self-managed on-prem cluster where you must write `ReplicatedMergeTree('/clickhouse/tables/{shard}/name', '{replica}')` explicitly. Syntax and behavior for this lab are identical either way; just know the engine name is doing more work for you in Cloud.

---

## 1. The MergeTree Family — Foundations

`MergeTree` is the base storage engine for nearly everything in ClickHouse. Data is stored in **parts**, sorted by an `ORDER BY` key, and background merges consolidate parts over time.

### 1.1 Create a basic MergeTree table

```sql
CREATE TABLE orders
(
    order_id     UInt64,
    customer_id  UInt32,
    order_date   Date,
    amount       Decimal(10, 2),
    status       String
)
ENGINE = MergeTree()
ORDER BY (customer_id, order_date)
PARTITION BY toYYYYMM(order_date);
```

Key clauses:
- **`ORDER BY`** — the sorting/primary key. Determines how data is physically sorted on disk and which sparse index is built. Choose columns you filter/aggregate on most.
- **`PARTITION BY`** — splits data into separate physical part-groups (commonly by month). Helps with data lifecycle (e.g., dropping old partitions) but is *not* a substitute for a good `ORDER BY`.

### 1.2 Insert sample data

```sql
INSERT INTO orders VALUES
(1, 101, '2026-01-05', 250.00, 'completed'),
(2, 102, '2026-01-06', 89.99,  'pending'),
(3, 101, '2026-02-10', 40.50,  'completed'),
(4, 103, '2026-02-11', 500.00, 'cancelled'),
(5, 102, '2026-02-14', 120.00, 'completed');
```

### 1.3 Inspect parts and merges

```sql
SELECT table, partition, name, rows, bytes_on_disk
FROM system.parts
WHERE table = 'orders' AND active = 1;
```

**Exercise 1:** Insert 3 more rows in a second `INSERT` statement, then re-run the query above. Notice a *new part* appears rather than the row being appended to the old one. Run:

```sql
OPTIMIZE TABLE orders FINAL;
```

Check `system.parts` again — parts should merge into fewer, larger parts.

---

## 2. ReplacingMergeTree — Deduplication on Merge

Use this when you need "upsert-like" behavior: the latest version of a row (by insertion order or a version column) should win, and older duplicates are removed **during background merges** (not immediately).

### 2.1 Create the table

```sql
CREATE TABLE customers_replacing
(
    customer_id UInt32,
    name        String,
    email       String,
    updated_at  DateTime
)
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY customer_id;
```

- The argument `updated_at` is the **version column** — on merge, ClickHouse keeps the row with the *highest* version per `ORDER BY` key.
- If you omit the version column, ClickHouse keeps the *last inserted* row for that key.

### 2.2 Insert an initial row, then an "update"

```sql
INSERT INTO customers_replacing VALUES
(101, 'Aditi Shah', 'aditi@old-email.com', '2026-01-01 10:00:00');

INSERT INTO customers_replacing VALUES
(101, 'Aditi Shah', 'aditi@new-email.com', '2026-03-15 09:30:00');
```

### 2.3 Query without merging — see both rows

```sql
SELECT * FROM customers_replacing;
```

You'll see **both** rows — deduplication has not happened yet because no merge has run.

### 2.4 Force deduplication for testing

```sql
OPTIMIZE TABLE customers_replacing FINAL;
SELECT * FROM customers_replacing;
```

Now only the row with `updated_at = 2026-03-15` remains.

> **Production tip:** Never rely on `OPTIMIZE ... FINAL` in production queries (it's expensive). Instead, query with:
> ```sql
> SELECT * FROM customers_replacing FINAL;
> ```
> or use `argMax()` to pick the latest row per key without forcing a merge:
> ```sql
> SELECT customer_id, argMax(name, updated_at) AS name, argMax(email, updated_at) AS email
> FROM customers_replacing
> GROUP BY customer_id;
> ```

**Exercise 2:** Insert a duplicate row for `customer_id = 101` with an *older* `updated_at` than the current winner. Confirm with `FINAL` that the newer row is still retained, not the most recently inserted one.

---

## 3. SummingMergeTree — Pre-Aggregated Sums

Use this for tables where you frequently need the **sum** of numeric columns grouped by a key — e.g., rolling metrics, counters, pre-aggregated fact tables.

### 3.1 Create the table

```sql
CREATE TABLE daily_sales
(
    sale_date    Date,
    product_id   UInt32,
    quantity     UInt32,
    revenue      Decimal(12, 2)
)
ENGINE = SummingMergeTree()
ORDER BY (sale_date, product_id);
```

- All numeric columns **not** in `ORDER BY` (here: `quantity`, `revenue`) are summed automatically during merges for rows sharing the same `ORDER BY` key.
- You can optionally specify which columns to sum: `SummingMergeTree((quantity, revenue))`.

### 3.2 Insert multiple rows for the same key

```sql
INSERT INTO daily_sales VALUES
('2026-03-01', 55, 3, 150.00),
('2026-03-01', 55, 2, 100.00),
('2026-03-01', 56, 1, 75.00);
```

### 3.3 Force merge and observe summing

```sql
OPTIMIZE TABLE daily_sales FINAL;
SELECT * FROM daily_sales ORDER BY sale_date, product_id;
```

Row `(2026-03-01, 55, ...)` should now show `quantity = 5`, `revenue = 250.00` — the two rows summed into one.

> As with `ReplacingMergeTree`, always add an explicit `SUM(...) ... GROUP BY` in your query, or query with `FINAL`, if you can't guarantee a merge has already happened.

**Exercise 3:** Add a `discount` column of type `Nullable(Decimal(12,2))`. Does it get summed automatically? (Hint: `SummingMergeTree` skips non-numeric and certain nullable/edge-case columns — check `system.parts` and the docs behavior, and compare to explicitly listing columns in the engine definition.)

---

## 4. AggregatingMergeTree — Arbitrary Aggregate Functions

`SummingMergeTree` only sums. For `avg`, `count distinct`, `uniq`, `max`, or custom combinators, use `AggregatingMergeTree` with **`AggregateFunction`** columns, typically fed via a **Materialized View**.

### 4.1 Create a raw events table (source)

```sql
CREATE TABLE page_views
(
    event_date  Date,
    user_id     UInt32,
    page        String
)
ENGINE = MergeTree()
ORDER BY (event_date, user_id);
```

### 4.2 Create the AggregatingMergeTree target table

```sql
CREATE TABLE page_views_agg
(
    event_date      Date,
    page            String,
    unique_users    AggregateFunction(uniq, UInt32),
    total_views     AggregateFunction(count)
)
ENGINE = AggregatingMergeTree()
ORDER BY (event_date, page);
```

### 4.3 Create a Materialized View to populate it

```sql
CREATE MATERIALIZED VIEW page_views_mv
TO page_views_agg
AS
SELECT
    event_date,
    page,
    uniqState(user_id) AS unique_users,
    countState()       AS total_views
FROM page_views
GROUP BY event_date, page;
```

- `uniqState(...)` / `countState(...)` produce the intermediate aggregate *state*, not the final value.
- The Materialized View fires automatically on every `INSERT` into `page_views`.

### 4.4 Insert data and query

```sql
INSERT INTO page_views VALUES
('2026-04-01', 1, '/home'),
('2026-04-01', 2, '/home'),
('2026-04-01', 1, '/home'),
('2026-04-01', 3, '/pricing');

SELECT
    event_date,
    page,
    uniqMerge(unique_users) AS unique_users,
    countMerge(total_views) AS total_views
FROM page_views_agg
GROUP BY event_date, page
ORDER BY page;
```

- `uniqMerge` / `countMerge` finalize the aggregate states into readable numbers.

**Exercise 4:** Add `avgState(...)`/`avgMerge(...)` for a numeric column (e.g., add a `duration_seconds` column to `page_views`) to compute average session duration per page.

---

## 5. Distributed Table Engine

The `Distributed` engine is a **stateless routing/proxy layer**: it stores no data itself, but forwards `SELECT` queries to underlying local tables across shards and merges results, and routes `INSERT`s to the correct shard by a sharding key.

> **Important for ClickHouse Cloud users:** Cloud's compute-storage separation means you generally do **not** need to manually create shards or `Distributed` tables to get horizontal scalability — queries already fan out across compute nodes against shared object storage via **parallel replicas**, and ClickHouse Cloud is actively rolling out multi-stage distributed query execution to push this further. The classic `Distributed` engine pattern below is essential to understand (it's still core ClickHouse architecture, used in self-managed/on-prem clusters and tested in interviews/exams), but on a single Cloud service you won't typically define your own multi-shard cluster by hand. If your Cloud environment exposes a named cluster (check `SELECT * FROM system.clusters`), you can still run the exercise as written.

### 5.1 Check available clusters

```sql
SELECT cluster, shard_num, replica_num, host_name
FROM system.clusters;
```

In ClickHouse Cloud this will typically show a single logical cluster representing your service's replicas rather than multiple independent shards.

### 5.2 Local table (per node/shard)

```sql
CREATE TABLE events_local
(
    event_time DateTime,
    user_id    UInt64,
    event_type String
)
ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

### 5.3 Distributed table on top

```sql
CREATE TABLE events_all AS events_local
ENGINE = Distributed(
    'default',        -- cluster name (replace with your cluster from system.clusters)
    'engines_lab',     -- database
    'events_local',    -- underlying local table
    intHash64(user_id) -- sharding key expression
);
```

### 5.4 Insert and query through the Distributed table

```sql
INSERT INTO events_all VALUES
(now(), 1001, 'pageview'),
(now(), 1002, 'click'),
(now(), 1003, 'pageview');

SELECT * FROM events_all ORDER BY event_time;
```

- `SELECT` from `events_all` fans out to every shard's `events_local` and merges results.
- `INSERT` into `events_all` hashes `user_id` and routes each row to the owning shard.

### 5.5 Choosing a sharding key (discussion)

| Sharding key | Effect |
|---|---|
| `rand()` | Best even distribution; no data colocation |
| `intHash64(user_id)` | Even distribution + colocates a user's own rows on one shard |
| `tenant_id` directly | Risk of hot shards if tenants are uneven in size |

**Exercise 5:** If your Cloud `system.clusters` only shows one shard, explain (in a sentence or two) why a `Distributed` table over it behaves like a pass-through to a single local table, and what would need to change (multi-shard, self-managed cluster) to see real query fan-out.

---

## 6. Wrap-Up Comparison

| Engine | Purpose | Dedup/Aggregation timing | Typical use case |
|---|---|---|---|
| `MergeTree` | General-purpose sorted storage | N/A | Base engine for almost everything |
| `ReplacingMergeTree` | Keep latest row per key | On merge (or with `FINAL`) | Slowly-changing dimension tables, upserts |
| `SummingMergeTree` | Sum numeric columns per key | On merge (or with `FINAL`/`GROUP BY`) | Rolling counters, pre-aggregated facts |
| `AggregatingMergeTree` | Arbitrary aggregate functions per key | On merge, finalized with `...Merge()` | Real-time dashboards via Materialized Views |
| `Distributed` | Query routing/proxy across shards | N/A (no storage) | Horizontal scale-out on self-managed clusters; less needed on Cloud due to shared storage |

## 7. Cleanup

```sql
DROP DATABASE IF EXISTS engines_lab;
```

---

### Further Reading
- ClickHouse docs: MergeTree family, ReplacingMergeTree, SummingMergeTree, AggregatingMergeTree, Distributed engine
- ClickHouse Cloud docs: SharedMergeTree architecture and parallel replicas
