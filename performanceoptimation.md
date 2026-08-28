# Lab: ClickHouse Indexing, Profiling & Query Tuning
### Primary/Skip Indexes • EXPLAIN & Query Profiling • Compression Codecs • Query Tuning

**Environment:** ClickHouse Cloud (SQL console or `clickhouse-client`)

---

## 0. Setup

```sql
CREATE DATABASE IF NOT EXISTS tuning_lab;
USE tuning_lab;
```

We'll load a reasonably large synthetic dataset so index/compression effects are actually visible (small tables fit in one or two granules and won't show much difference).

```sql
CREATE TABLE http_logs
(
    event_time  DateTime,
    domain      String,
    status      UInt16,
    method      String,
    path        String,
    bytes_sent  UInt32,
    user_id     UInt64
)
ENGINE = MergeTree()
ORDER BY (domain, event_time);

INSERT INTO http_logs
SELECT
    now() - randUniform(0, 86400 * 30) AS event_time,
    ['example.com','api.example.com','shop.example.com','cdn.example.com'][1 + rand() % 4] AS domain,
    [200, 200, 200, 301, 404, 500][1 + rand() % 6] AS status,
    ['GET','GET','GET','POST','PUT'][1 + rand() % 5] AS method,
    concat('/path/', toString(rand() % 500)) AS path,
    rand() % 50000 AS bytes_sent,
    rand() % 1000000 AS user_id
FROM numbers(5000000);
```

> **Cloud note:** `EXPLAIN`, `system.query_log`, `system.parts`, and index mechanics all work identically on ClickHouse Cloud. Query profiling via `system.query_log`/`system.trace_log` is fully available; just remember Cloud may sample or rate-limit very verbose trace logging by default on smaller service tiers.

---

## 1. The Primary Index (Sparse Index)

ClickHouse's primary index is **not** a traditional B-tree — it's a **sparse index**: one entry per **granule** (default 8,192 rows), storing the `ORDER BY` key value at the start of that granule.

### 1.1 Inspect primary index size and granule counts

```sql
SELECT
    table,
    formatReadableSize(sum(primary_key_bytes_in_memory)) AS pk_size_in_memory,
    sum(marks) AS total_marks
FROM system.parts
WHERE table = 'http_logs' AND active
GROUP BY table;
```

Each "mark" corresponds to one granule boundary.

### 1.2 See the primary index in action with EXPLAIN

```sql
EXPLAIN indexes = 1
SELECT count()
FROM http_logs
WHERE domain = 'api.example.com'
  AND event_time >= now() - INTERVAL 7 DAY;
```

Look for a section like:

```
Indexes:
  PrimaryKey
    Keys: domain, event_time
    Condition: and((domain in ['api.example.com', 'api.example.com']), (event_time in [...]))
    Parts: X/Y
    Granules: A/B
```

`Granules: A/B` tells you how many granules were actually read (`A`) out of the total (`B`) — this is the single most important number for judging index effectiveness.

### 1.3 The leading-column rule

The primary index only prunes granules for **leading columns** of `ORDER BY`, used left-to-right without gaps.

```sql
-- Efficient: domain is the leading ORDER BY column
EXPLAIN indexes = 1
SELECT count() FROM http_logs WHERE domain = 'shop.example.com';

-- Inefficient: status is NOT part of ORDER BY at all — full scan of all granules
EXPLAIN indexes = 1
SELECT count() FROM http_logs WHERE status = 500;
```

**Exercise 1:** Run both `EXPLAIN` statements above and compare the `Granules: A/B` ratio. Then run the actual queries and compare `read_rows` in `system.query_log` (see Section 3).

---

## 2. Skip Indexes (Data-Skipping / Secondary Indexes)

Skip indexes extend pruning to **non-leading / non-ORDER-BY columns** by storing lightweight per-granule summaries (min/max, a value set, or a bloom filter).

| Type | Best for | Notes |
|---|---|---|
| `minmax` | Range filters on numeric/date columns not in `ORDER BY` | Cheapest; stores per-granule min & max |
| `set(N)` | Low-cardinality equality filters | Stores up to `N` distinct values per granule |
| `bloom_filter` | High-cardinality equality filters | Probabilistic; some false positives, no false negatives |
| `ngrambf_v1` / `tokenbf_v1` | Substring / word matching in strings | Bloom filter over n-grams or tokens |
| `hypothesis` | Precomputed boolean/condition flags | Experimental; stores per-granule true/false/mixed for an expression |

### 2.1 Add a minmax skip index

```sql
ALTER TABLE http_logs ADD INDEX idx_bytes_minmax (bytes_sent) TYPE minmax GRANULARITY 4;
ALTER TABLE http_logs MATERIALIZE INDEX idx_bytes_minmax;
```

`GRANULARITY 4` groups every 4 granules into one index block — a trade-off between index size and pruning precision.

### 2.2 Add a set index for low-cardinality equality

```sql
ALTER TABLE http_logs ADD INDEX idx_status_set (status) TYPE set(10) GRANULARITY 4;
ALTER TABLE http_logs MATERIALIZE INDEX idx_status_set;
```

### 2.3 Add a bloom filter for high-cardinality equality

```sql
ALTER TABLE http_logs ADD INDEX idx_user_bloom (user_id) TYPE bloom_filter GRANULARITY 4;
ALTER TABLE http_logs MATERIALIZE INDEX idx_user_bloom;
```

### 2.4 Confirm skip indexes are used

```sql
EXPLAIN indexes = 1
SELECT count() FROM http_logs WHERE status = 500;
```

You should now see a second block in the output:

```
Skip
  Name: idx_status_set
  Description: set GRANULARITY 4
  Parts: X/Y
  Granules: A/B
```

**Exercise 2:** Re-run the `status = 500` query's `EXPLAIN` from Section 1.3 before and after adding `idx_status_set`. Quantify the improvement in `Granules: A/B`.

### 2.5 Inspect skip index storage cost

```sql
SELECT
    table,
    name,
    formatReadableSize(data_compressed_bytes) AS index_size
FROM system.data_skipping_indices
WHERE table = 'http_logs';
```

> **Trade-off:** every skip index adds write-time overhead (must be maintained on every insert/merge) and disk space. Only add indexes that measurably reduce granules read for real query patterns — verify with `EXPLAIN`, don't guess.

---

## 3. Query Profiling & EXPLAIN — Deeper Dive

### 3.1 EXPLAIN variants

```sql
-- Logical query plan (default)
EXPLAIN SELECT domain, count() FROM http_logs GROUP BY domain;

-- Physical/pipeline plan — shows actual execution steps and parallelism
EXPLAIN PIPELINE SELECT domain, count() FROM http_logs GROUP BY domain;

-- Index usage — the most useful for tuning
EXPLAIN indexes = 1
SELECT domain, count() FROM http_logs
WHERE event_time >= today() - 7
GROUP BY domain;

-- Estimated cost/row counts without running the query
EXPLAIN ESTIMATE
SELECT domain, count() FROM http_logs WHERE domain = 'api.example.com';
```

### 3.2 Profiling actual executed queries via system.query_log

Run a query, then inspect it:

```sql
SELECT domain, count() AS c, avg(bytes_sent) AS avg_bytes
FROM http_logs
WHERE status = 500
GROUP BY domain
ORDER BY c DESC;
```

```sql
SELECT
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage,
    result_rows
FROM system.query_log
WHERE query LIKE '%http_logs%'
  AND type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC
LIMIT 5;
```

Key fields:
- **`read_rows` / `read_bytes`** — actual data scanned; the number you want to shrink via better indexing.
- **`memory_usage`** — peak memory for the query; watch for queries close to your memory limits.
- **`query_duration_ms`** — wall-clock time, useful for before/after comparisons.

### 3.3 PREWHERE — automatic and manual filter pushdown

ClickHouse can apply a filter **before** reading all requested columns, reducing I/O when a query selects many columns but filters on a highly selective one:

```sql
-- Let ClickHouse decide automatically (default behavior)
SELECT path, bytes_sent, user_id
FROM http_logs
WHERE status = 500;

-- Force it explicitly if needed
SELECT path, bytes_sent, user_id
FROM http_logs
PREWHERE status = 500;
```

Compare `read_bytes` in `system.query_log` between a query written with `WHERE` only vs. one that benefits from `PREWHERE` on a very selective column and many other columns selected.

**Exercise 3:** Pick a query that selects 5+ columns but filters on one selective column. Compare `read_bytes` with and without an explicit `PREWHERE`.

---

## 4. Compression Codecs

ClickHouse compresses data per-column, per-part. Choosing the right codec per column can shrink storage and — since less data is read from disk — often speeds up queries too.

| Codec | Best for | Notes |
|---|---|---|
| `LZ4` (default) | General-purpose, fast decompression | Good default, especially for CPU-bound workloads |
| `ZSTD(level)` | Better ratio than LZ4, more CPU cost | Great for cold/archival data; tune `level` (1–22) |
| `Delta` | Monotonically increasing/slowly-changing numerics (timestamps, counters, IDs) | Stores differences between consecutive values |
| `DoubleDelta` | Time series with roughly constant intervals | Second-order delta; very effective on regular timestamps |
| `Gorilla` | Floating-point time-series metrics | XOR-based, effective for slowly changing floats |
| `T64` | Integer columns with a limited range of values | Transposes bits for better downstream compression |

### 4.1 Check current compression ratio

```sql
SELECT
    name AS column,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table = 'http_logs' AND database = 'tuning_lab'
ORDER BY data_uncompressed_bytes DESC;
```

### 4.2 Create a comparison table with tuned codecs

```sql
CREATE TABLE http_logs_tuned
(
    event_time  DateTime CODEC(DoubleDelta, ZSTD(3)),
    domain      LowCardinality(String),
    status      UInt16 CODEC(T64, ZSTD(1)),
    method      LowCardinality(String),
    path        String CODEC(ZSTD(3)),
    bytes_sent  UInt32 CODEC(ZSTD(1)),
    user_id     UInt64 CODEC(Delta, ZSTD(1))
)
ENGINE = MergeTree()
ORDER BY (domain, event_time);

INSERT INTO http_logs_tuned SELECT * FROM http_logs;
```

- `LowCardinality(String)` isn't a codec but a storage optimization for columns with few distinct values (like `domain`, `method`) — dictionary-encodes the values, dramatically shrinking size and speeding up filters/GROUP BY.
- `DoubleDelta` fits `event_time` well since log timestamps trend upward at roughly regular intervals.
- `Delta` fits `user_id`-like columns less perfectly here since IDs are random, but is ideal for genuinely monotonic IDs.

### 4.3 Compare sizes

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed
FROM system.parts
WHERE table IN ('http_logs', 'http_logs_tuned') AND active
GROUP BY table;
```

**Exercise 4:** Try `CODEC(ZSTD(1))` vs `CODEC(ZSTD(9))` on the `path` column of a copy of the table. Compare compressed size and measure `INSERT` time for a fresh load with each — document the throughput/ratio trade-off.

---

## 5. Query Tuning — Putting It Together

### 5.1 A tuning workflow

1. **Identify the slow query** — from `system.query_log` (`ORDER BY query_duration_ms DESC`), or user reports.
2. **Run `EXPLAIN indexes = 1`** — check `Granules: A/B` for the primary key and any skip indexes.
3. **Check `read_rows`/`read_bytes`** in `system.query_log` for the actual execution.
4. **Consider**, in order of typical impact:
   - Reordering/choosing a better `ORDER BY` (only possible at table creation, or via a new table + `INSERT ... SELECT`)
   - Adding a targeted skip index
   - Using `PREWHERE` for highly selective filters over wide row selects
   - Tuning compression codecs to reduce I/O
   - Pre-aggregating with `AggregatingMergeTree`/`SummingMergeTree` + materialized views for repeated aggregate queries
5. **Re-run EXPLAIN and the query**, compare before/after `read_rows`, `query_duration_ms`.

### 5.2 Worked example

```sql
-- Before: filtering on a column with no index support
EXPLAIN indexes = 1
SELECT count() FROM http_logs WHERE method = 'POST' AND bytes_sent > 40000;
```

```sql
-- Add supporting skip indexes
ALTER TABLE http_logs ADD INDEX idx_method_set (method) TYPE set(5) GRANULARITY 4;
ALTER TABLE http_logs MATERIALIZE INDEX idx_method_set;

-- idx_bytes_minmax from Section 2.1 already covers bytes_sent

-- After
EXPLAIN indexes = 1
SELECT count() FROM http_logs WHERE method = 'POST' AND bytes_sent > 40000;
```

Compare the `Granules: A/B` for both skip indexes shown in the "after" plan, and confirm actual `read_rows` dropped via `system.query_log`.

### 5.3 Common tuning wins checklist

- ✅ `ORDER BY` starts with your most common equality-filter column(s), then range-filter columns (e.g., timestamp) last
- ✅ `LowCardinality(String)` on enum-like string columns
- ✅ Skip indexes only where `EXPLAIN` shows a real reduction in granules — not applied blindly to every column
- ✅ `PREWHERE` (or let automatic optimization handle it) when selecting many columns but filtering on a narrow, selective one
- ✅ Compression codecs matched to data shape (`Delta`/`DoubleDelta` for monotonic numerics, `ZSTD` for cold data)
- ✅ Materialized views + `AggregatingMergeTree`/`SummingMergeTree` for dashboard queries run repeatedly on the same aggregation
- ❌ Avoid functions wrapping the `ORDER BY`/indexed column in a `WHERE` clause (e.g., `toDate(event_time) = ...` instead of a native range on `event_time`) — this defeats index pruning

**Exercise 5:** Take one query from earlier in this lab, deliberately wrap the indexed column in a function in the `WHERE` clause (e.g., `WHERE toString(status) = '500'`), and use `EXPLAIN indexes = 1` to observe how pruning is lost. Then fix it and confirm pruning returns.

---

## 6. Wrap-Up Comparison

| Technique | What it optimizes | Diagnostic tool |
|---|---|---|
| Primary (sparse) index | Granule pruning on leading `ORDER BY` columns | `EXPLAIN indexes = 1` → `PrimaryKey` block |
| Skip index (`minmax`/`set`/`bloom_filter`/etc.) | Granule pruning on other columns | `EXPLAIN indexes = 1` → `Skip` block |
| `PREWHERE` | I/O reduction when selecting many columns, filtering on few | `system.query_log.read_bytes` |
| Compression codecs | Storage size and I/O throughput | `system.columns` / `system.parts` |
| Materialized views + aggregate engines | Repeated aggregate query latency | `system.query_log.query_duration_ms` |

## 7. Cleanup

```sql
DROP DATABASE IF EXISTS tuning_lab;
```

---

### Further Reading
- ClickHouse docs: "A simple guide to query optimization," sparse primary indexes, data-skipping indexes
- ClickHouse docs: `EXPLAIN` statement reference, `system.query_log`, `system.trace_log`
- ClickHouse docs: Column compression codecs reference
