# ClickHouse Cloud Lab
### Native Data Types · Schema Design Best Practices · Choosing Primary Keys · Partitioning Strategy

**Where to run this:** Your own ClickHouse Cloud service → **SQL console**.
**Approach:** You'll build a deliberately naive table first, measure it, then rebuild it well and measure the difference — the numbers make the "why" concrete instead of theoretical.

> ⚠️ If your service shows "This service is idle. Wake your service...", click it and wait a few seconds before running queries.

---

## Part 1 — Native Data Types

Rather than read a type list, build one table that uses a representative type from each family, insert a single row, and inspect it.

### Step 1 — A type "playground" table

```sql
CREATE TABLE type_playground
(
    id             UInt32,
    small_flag     UInt8,
    signed_int     Int32,
    big_counter    UInt64,
    price          Decimal(10,2),
    ratio          Float64,
    code           FixedString(3),
    name           String,
    category       LowCardinality(String),
    is_active      Bool,
    status         Enum8('pending' = 1, 'active' = 2, 'closed' = 3),
    tags           Array(String),
    coordinates    Tuple(Float64, Float64),
    metadata       Map(String, String),
    day            Date,
    day_precise    Date32,
    created_at     DateTime,
    created_at_ms  DateTime64(3),
    external_id    UUID,
    client_ip      IPv4,
    client_ip_v6   IPv6,
    maybe_missing  Nullable(String)
)
ENGINE = MergeTree
ORDER BY id;

INSERT INTO type_playground VALUES
(
    1, 1, -42, 9999999999,
    19.99, 0.5, 'ABC', 'Widget',
    'electronics', true, 'active',
    ['sale','new'], (37.7749, -122.4194),
    {'source':'web','campaign':'spring'},
    '2024-01-15', '2024-01-15',
    '2024-01-15 10:30:00', '2024-01-15 10:30:00.123',
    generateUUIDv4(), '192.168.1.1', '::1',
    NULL
);

SELECT * FROM type_playground FORMAT Vertical;
```

### Step 2 — What each type is actually for

| Type | Use it for | Note |
|---|---|---|
| `UInt8`/`UInt16`/`UInt32`/`UInt64` | Non-negative integers, sized to the actual range | A flag or small enum-like value fits in `UInt8` (1 byte) — don't default to `UInt64` (8 bytes) out of habit |
| `Int8`…`Int64` | Signed integers | Same sizing logic as above |
| `Decimal(P,S)` | Money, exact fixed-point math | Never use `Float64` for currency — it's binary floating point and will round |
| `Float32`/`Float64` | Measurements, ratios, scientific data | Fine where exactness isn't required |
| `String` | Free-text, unbounded, or genuinely high-cardinality values | The default "just works" choice — but see `LowCardinality` below |
| `FixedString(N)` | Fixed-length codes (currency codes, hashes) | Pads short values with zero bytes and truncates long ones — only use when every value really is length `N` |
| `LowCardinality(String)` | Repeated string values from a small-ish set (status, country, category, event type) | Dictionary-encodes the column; often a large win on both size and speed |
| `Enum8`/`Enum16` | A **known, fixed** small set of string labels | Similar benefit to `LowCardinality`, but the set is baked into the schema at `CREATE TABLE` time — adding a new value needs `ALTER TABLE` |
| `Bool` | True/false flags | Stored as `UInt8` under the hood; use it for readability |
| `Date`/`Date32` | Calendar dates | `Date` covers 1970–2149 in 2 bytes; `Date32` extends the range at 4 bytes |
| `DateTime`/`DateTime64(P)` | Timestamps | `DateTime` is second precision (4 bytes); `DateTime64(3)` gives millisecond precision when you need it |
| `UUID` | Globally unique identifiers | Native 16-byte storage, not a text `String` |
| `IPv4`/`IPv6` | IP addresses | Native storage + IP-aware functions, much smaller than storing as `String` |
| `Array(T)` | A variable-length list of same-typed values per row | `tags`, `scores`, etc. |
| `Tuple(T1, T2, ...)` | A small, fixed-shape group of values | Coordinates, (min, max) pairs |
| `Map(K, V)` | Key-value pairs per row | Loosely-structured attributes; a real dedicated column is usually still better if you know the keys ahead of time |
| `Nullable(T)` | A column that can be genuinely absent | Use sparingly — see Part 2 |

**Note on the JSON type:** ClickHouse has a native `JSON` type for schemaless/semi-structured data. It's evolving quickly, so if you're reaching for it, check the current docs for your version rather than assuming behavior from memory — this lab doesn't use it for that reason.

---

## Part 2 — Schema Design Best Practices

### Step 3 — Build a deliberately naive table

```sql
CREATE TABLE events_naive
(
    event_id    UInt64,
    user_id     UInt64,
    event_type  String,
    country     String,
    device      String,
    is_premium  Nullable(UInt8),
    revenue     Nullable(Float64),
    event_time  DateTime
)
ENGINE = MergeTree
ORDER BY event_id;

INSERT INTO events_naive
SELECT
    number + 1 AS event_id,
    (number % 500000) + 1 AS user_id,
    arrayElement(['click','view','purchase','signup','logout'], (number % 5) + 1) AS event_type,
    arrayElement(['USA','UK','India','Germany','Brazil','Japan'], (number % 6) + 1) AS country,
    arrayElement(['ios','android','web'], (number % 3) + 1) AS device,
    if(rand() % 4 = 0, NULL, rand() % 2) AS is_premium,
    if(rand() % 3 = 0, NULL, round((rand() % 10000) / 100.0, 2)) AS revenue,
    toDateTime('2024-01-01 00:00:00') + toIntervalSecond(number) AS event_time
FROM numbers(3000000);
```

### Step 4 — Measure it

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    sum(rows)                                        AS total_rows
FROM system.parts
WHERE table = 'events_naive' AND active
GROUP BY table;

SELECT
    name,
    type,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed
FROM system.columns
WHERE table = 'events_naive' AND database = currentDatabase()
GROUP BY name, type
ORDER BY sum(data_compressed_bytes) DESC;
```

Note the size, and which columns are the biggest contributors — you'll compare against this after Step 5.

### Step 5 — Rebuild it well

```sql
CREATE TABLE events
(
    event_id    UInt64,
    user_id     UInt64,
    event_type  LowCardinality(String),
    country     LowCardinality(String),
    device      LowCardinality(String),
    is_premium  UInt8 DEFAULT 0,
    revenue     Float64 DEFAULT 0,
    event_time  DateTime CODEC(DoubleDelta, ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, country, event_time);

INSERT INTO events (event_id, user_id, event_type, country, device, is_premium, revenue, event_time)
SELECT event_id, user_id, event_type, country, device, coalesce(is_premium, 0), coalesce(revenue, 0), event_time
FROM events_naive;
```

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    sum(rows)                                      AS total_rows
FROM system.parts
WHERE table = 'events' AND active
GROUP BY table;
```

Compare this against Step 4's `events_naive` number for the same row count.

### What changed, and why

- **`LowCardinality(String)` on `event_type`, `country`, `device`** — these have only 5–6 distinct values repeated 3 million times. Dictionary encoding turns "the same short string over and over" into "a small integer over and over," which compresses far better and speeds up equality filters and `GROUP BY`.
- **Sentinel defaults instead of `Nullable`** — `is_premium Nullable(UInt8)` isn't just "a UInt8 that can be null": ClickHouse stores a second hidden column (a null-bitmap) alongside it, so every read and write pays for two columns instead of one. If "unknown" doesn't need to be distinguishable from "0"/"false" for your use case, a plain column with a `DEFAULT` is both smaller and faster. (If you genuinely need three-state logic — true/false/unknown — `Nullable` is the right tool; the point is not to reach for it automatically.)
- **`CODEC(DoubleDelta, ZSTD(1))` on `event_time`** — timestamps inserted in increasing order compress very well once you store the *differences between consecutive values* (`DoubleDelta`) rather than the raw values, then general-purpose compress what's left (`ZSTD`). This is a targeted codec for a specific data shape, not a default you'd put on every column.
- **`PARTITION BY toYYYYMM(event_time)`** and the **`ORDER BY`** choice are covered in Parts 3–4 below — they're doing real work here too, not just cosmetic.

**Note on column order in `CREATE TABLE`:** the order columns are *listed* in the table definition has essentially no effect on compression — each column is stored and compressed independently. What matters enormously is the order of columns in `ORDER BY`, which is the subject of Part 3.

---

## Part 3 — Choosing Primary Keys

In MergeTree, `ORDER BY` does two jobs at once: it defines the **physical sort order** of data on disk, and it defines the **primary index** — a sparse index (one entry roughly every 8,192 rows by default) that lets ClickHouse skip whole blocks instead of scanning them. `PRIMARY KEY`, if you specify one, must be a **prefix** of `ORDER BY`; if you omit it, it defaults to the full `ORDER BY`.

### Step 6 — See the effect of column order directly

`events` is ordered by `(event_type, country, event_time)`. Compare a filter on the *first* key column against a filter on a column that isn't in the key at all:

```sql
-- Filters on event_type — the FIRST ORDER BY column
EXPLAIN indexes = 1
SELECT count() FROM events WHERE event_type = 'purchase';

-- Filters on user_id — NOT part of the ORDER BY
EXPLAIN indexes = 1
SELECT count() FROM events WHERE user_id = 12345;
```

Compare the `Granules` line in each plan. The `user_id` filter has to scan far more granules, because nothing about the physical layout groups rows by `user_id`.

### Step 7 — The ordering principle

Put columns in `ORDER BY` roughly in this order:
1. **Columns you filter on with equality most often**, lowest cardinality first (a handful of distinct values, like `event_type`).
2. **Columns you filter on with equality less often, or with slightly higher cardinality** (`country`).
3. **A column you range-filter on, usually a timestamp**, last (`event_time`) — this keeps each `(event_type, country)` group internally sorted by time, which is exactly what most dashboards query for.

The intuition: a sparse index can only use a key column to skip data if every column *before* it in the key is already pinned down by the query's filters. A filter on `event_time` alone gets little benefit here, because the index is sorted by `event_type`/`country` first — time order is only guaranteed *within* an `(event_type, country)` group, not globally.

### Step 8 — A narrower `PRIMARY KEY` than `ORDER BY`

Sometimes you want extra columns in the sort order (for compression, or so a later column is grouped for fast range scans) without growing the in-memory primary index that has to be kept for every part.

```sql
CREATE TABLE events_v2
(
    event_id    UInt64,
    user_id     UInt64,
    event_type  LowCardinality(String),
    country     LowCardinality(String),
    device      LowCardinality(String),
    is_premium  UInt8 DEFAULT 0,
    revenue     Float64 DEFAULT 0,
    event_time  DateTime CODEC(DoubleDelta, ZSTD(1))
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
PRIMARY KEY (event_type, country)
ORDER BY (event_type, country, event_time, user_id);
```

**Note:** Data is still physically sorted by the full `ORDER BY` (including `user_id` last), so you still get whatever compression benefit that ordering provides — but the sparse index itself only tracks `(event_type, country)`, keeping it smaller. This is a fine-tuning move; start with `PRIMARY KEY` = `ORDER BY` (i.e., don't specify `PRIMARY KEY` at all) until you have a concrete reason to split them.

```sql
DROP TABLE events_v2;
```

---

## Part 4 — Partitioning Strategy

`PARTITION BY` is easy to mistake for "another index." It isn't, primarily. Its real job is **data lifecycle management** — letting you cheaply drop, move, or back up a whole time range (or other coarse business dimension) at once. It can *also* help a query skip whole partitions, but that's a secondary benefit, not the main reason to use it.

### Step 9 — Inspect the partitions you already created

`events` is partitioned by `toYYYYMM(event_time)`:

```sql
SELECT
    partition,
    count()                                    AS num_parts,
    sum(rows)                                  AS total_rows,
    formatReadableSize(sum(bytes_on_disk))     AS size
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY partition;
```

With three million rows inserted across `2024-01-01` and onward at one-second increments, you should see this land in a small, manageable number of monthly partitions.

### Step 10 — See the anti-pattern: partitioning by a high-cardinality column

```sql
CREATE TABLE events_bad_partition
(
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    event_time DateTime
)
ENGINE = MergeTree
PARTITION BY user_id
ORDER BY (user_id, event_time);

INSERT INTO events_bad_partition
SELECT number + 1, (number % 20000) + 1, 'click', now()
FROM numbers(50000);

SELECT count() AS num_partitions
FROM system.parts
WHERE table = 'events_bad_partition' AND active;
```

**Note:** This creates up to 20,000 separate partitions from just 50,000 rows — each one a tiny, separately-managed chunk of data. In production, this leads to excessive background merge work, "too many parts" errors under write load, and slower queries overall, because ClickHouse now has thousands of small parts to open and merge instead of a handful of large ones. As a rule of thumb: aim for a partition scheme that produces tens to a few hundred partitions for a table, each holding a substantial chunk of data — not one partition per entity.

```sql
DROP TABLE events_bad_partition;
```

### Step 11 — Use partitions for lifecycle management

Dropping an entire partition is close to instant — it deletes files, it doesn't rewrite data — which is the actual payoff of partitioning by time.

```sql
-- See partition IDs in their concrete form (e.g. '202401' for January 2024)
SELECT DISTINCT partition FROM system.parts WHERE table = 'events' AND active ORDER BY partition;

-- Drop the oldest month outright
ALTER TABLE events DROP PARTITION '202401';
```

Or automate the same idea with a TTL instead of a manual `DROP PARTITION`:

```sql
ALTER TABLE events MODIFY TTL event_time + INTERVAL 12 MONTH DELETE;
```

**Note:** `DROP PARTITION` is immediate and manual — good for "get rid of this range right now." `TTL` is declarative and ongoing — ClickHouse will expire rows older than the threshold automatically as a background process. Use TTL for a standing retention policy; use `DROP PARTITION` for one-off cleanup or reprocessing a specific range.

---

## Wrap-Up

**What you covered:**
- A tour of ClickHouse's native types and what each is actually for, including the storage cost of `Nullable`
- Concrete before/after compression numbers from switching to `LowCardinality`, sentinel defaults, and a targeted `CODEC`
- Why `ORDER BY` column order determines which filters are cheap, verified with `EXPLAIN indexes = 1`, plus when a narrower `PRIMARY KEY` makes sense
- Why partitioning is primarily a lifecycle tool, not a query-speed tool — including a hands-on look at the "too many partitions" failure mode

**Cleanup (optional):**

```sql
DROP TABLE IF EXISTS type_playground;
DROP TABLE IF EXISTS events_naive;
DROP TABLE IF EXISTS events;
```

**Where to go next:**
- Try `ALTER TABLE events MODIFY COLUMN revenue Float64 CODEC(Gorilla, ZSTD(1))` and compare compressed size against the plain `ZSTD`-only version — `Gorilla` is tuned for slowly-changing floating point series.
- Recreate `events` with a different `ORDER BY` (e.g. `(country, event_type, event_time)`) and re-run Step 6's `EXPLAIN` comparisons to see how the "cheap filter" set shifts with the key order.
