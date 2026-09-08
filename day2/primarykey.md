# How ClickHouse Primary Key Works — `logs` Table Example

## Table Definition

```sql
CREATE TABLE logs
(
    event_time   DateTime,
    service      String,
    level        String,
    status_code  UInt16,
    latency_ms   UInt32,
    message      String
)
ENGINE = MergeTree()
PARTITION BY toDate(event_time)
ORDER BY (service, event_time);
```

No `PRIMARY KEY` is specified, so it defaults to the `ORDER BY` clause:

```sql
PRIMARY KEY (service, event_time)
```

---

## 1. What the Primary Key Actually Does

Unlike MySQL/PostgreSQL, ClickHouse's primary key:

| Behavior | ClickHouse | Traditional RDBMS |
|---|---|---|
| Enforces uniqueness | ❌ No | ✅ Yes |
| Builds a B-tree per row | ❌ No | ✅ Yes |
| Purpose | Sparse index + physical sort order | Uniqueness + fast lookup |
| Index granularity | 1 entry per 8,192 rows (default) | 1 entry per row |

So in this table, you can have duplicate `(service, event_time)` pairs — perfectly fine for logs.

---

## 2. Physical Storage Layout

Because of `PARTITION BY toDate(event_time)` and `ORDER BY (service, event_time)`:

1. Data is split into **daily partitions** first (one partition per calendar day).
2. **Within each partition**, rows are physically sorted by `service`, then by `event_time` within each service.
3. ClickHouse groups every 8,192 sorted rows into a **granule** and stores one sparse index entry (the first row's key values) per granule.

**Example physical order inside a single day's partition:**

```
service        event_time
-----------    -------------------
auth-api       2026-09-08 00:01:02
auth-api       2026-09-08 00:04:15
auth-api       2026-09-08 03:22:40
checkout-api   2026-09-08 00:00:11
checkout-api   2026-09-08 00:02:47
checkout-api   2026-09-08 05:10:03
payments-api   2026-09-08 00:00:59
...
```

Notice: rows are grouped by `service` first, and only sorted by time *within* each service block.

---

## 3. How the Sparse Index Speeds Up Queries

The index stores marks like:

| Mark # | service | event_time (first row of granule) |
|---|---|---|
| 0 | auth-api | 00:01:02 |
| 1 | auth-api | 03:40:00 |
| 2 | checkout-api | 00:00:11 |
| 3 | checkout-api | 04:55:00 |

When a query comes in, ClickHouse binary-searches this small in-memory index to find which granules could possibly contain matching rows, then reads only those from disk.

---

## 4. Query Performance Patterns

### ✅ Fast — uses the primary key efficiently

```sql
-- filters on the FIRST key column
SELECT * FROM logs WHERE service = 'checkout-api';

-- filters on service, then narrows by event_time (ideal case)
SELECT * FROM logs
WHERE service = 'checkout-api'
  AND event_time >= now() - INTERVAL 1 HOUR;
```
ClickHouse jumps straight to the `checkout-api` block, then binary-searches within it by time.

### ⚠️ Partial benefit

```sql
-- date is in the partition key, so whole days are skipped,
-- but within a day it still scans across all services
SELECT * FROM logs
WHERE event_time >= '2026-09-08 00:00:00'
  AND event_time <  '2026-09-09 00:00:00';
```

### ❌ Slow — can't use the index

```sql
-- event_time is the 2nd key column; without 'service' filter,
-- ClickHouse can't binary-search it directly
SELECT * FROM logs WHERE event_time >= now() - INTERVAL 1 HOUR;

-- status_code / level are not in the key at all — full granule scan
-- (partition pruning by date still helps, but that's it)
SELECT * FROM logs WHERE status_code >= 500;
SELECT * FROM logs WHERE level = 'ERROR';
```

---

## 5. Is `(service, event_time)` the Right Key Here?

Depends on your dominant query pattern:

| Your typical query | Best key choice |
|---|---|
| "Logs for service X in time range Y" | `(service, event_time)` ← current setup ✅ |
| "Recent errors across ALL services" | `(event_time)` or `(toStartOfHour(event_time), service)` |
| "Errors for service X, recent first" | `(service, level, event_time)` |

If you frequently filter on `status_code` or `level` regardless of `service`, add **secondary skip indexes** instead of reworking the primary key:

```sql
ALTER TABLE logs ADD INDEX idx_status status_code TYPE minmax GRANULARITY 4;
ALTER TABLE logs ADD INDEX idx_level  level       TYPE set(10) GRANULARITY 4;
```

---

## 6. Summary

- **Primary key = sort order + sparse index**, not a uniqueness constraint.
- Column **order matters**: only a left-to-right *prefix* of the key can be efficiently used by `WHERE` filters.
- `PARTITION BY toDate(event_time)` prunes whole days; the primary key then prunes granules *within* a day.
- For this `logs` table, queries scoped by `service` first are cheap; queries scoped only by `event_time` or other columns are not — pick the key order based on how you actually query the data.
