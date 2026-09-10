# ClickHouse Data Types — Complete Guide

ClickHouse natively supports four major categories of data types: **Numeric**, **String**, **Date/Time**, and **Semi-structured**. Choosing the right type in each category directly affects disk space and query speed, since ClickHouse is a **columnar database** — smaller types mean less data to scan per query.

---

## 1. Numeric Types

| Type | Bytes | Range |
|---|---|---|
| `UInt8` | 1 byte | 0 to 255 |
| `UInt16` | 2 bytes | 0 to 65,535 |
| `UInt32` | 4 bytes | 0 to 4,294,967,295 (~4.29 billion) |
| `UInt64` | 8 bytes | 0 to 18,446,744,073,709,551,615 |
| `UInt128` | 16 bytes | 0 to ~3.4 × 10³⁸ |
| `UInt256` | 32 bytes | 0 to ~1.15 × 10⁷⁷ |
| `Int8` | 1 byte | -128 to 127 |
| `Int16` | 2 bytes | -32,768 to 32,767 |
| `Int32` | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `Int64` | 8 bytes | -9.2 × 10¹⁸ to 9.2 × 10¹⁸ |
| `Float32` | 4 bytes | ~±3.4 × 10³⁸ (approximate, ~7 accurate digits) |
| `Float64` | 8 bytes | ~±1.8 × 10³⁰⁸ (approximate, ~15-17 accurate digits) |
| `Decimal(P,S)` | 4/8/16/32 bytes (depends on P) | Exact precision — P is total digits, S is digits after the decimal point (e.g. `Decimal(10,2)` = 8 digits before the point, 2 after) |

**Note:** Decimal's byte size depends on precision (P): P≤9 → 4 bytes, P≤18 → 8 bytes, P≤38 → 16 bytes, P≤76 → 32 bytes. Decimal is best for financial data because it stays exact, unlike Float which can have rounding errors.

**Rule:** Always pick the smallest type that safely covers your data's actual range. If a quantity never exceeds 50, `UInt8` is enough — using `UInt64` or `UInt256` just wastes space for no reason.

### Example — Numeric
```sql
CREATE TABLE orders
(
    order_id  UInt32,   -- incrementing ID, fits easily in 4 bytes
    qty       UInt8     -- quantity never exceeds 50, so 1 byte is enough
)
ENGINE = MergeTree
ORDER BY order_id;
```
Using `UInt256` for both columns instead would cost 32 bytes each per row — 32x more space than needed for `order_id` and 32x more for `qty`.

---

## 2. String Types

| Type | Bytes | Description |
|---|---|---|
| `String` | Variable (actual length + a small length-prefix overhead, ~1-9 bytes) | No fixed limit — takes as much space as the text needs, plus a bit extra to store the length |
| `FixedString(N)` | Exactly N bytes | Always N bytes — shorter text gets zero-padded, longer text gets truncated |
| `LowCardinality(String)` | Small index per row (usually 1-4 bytes) + one shared dictionary | Each unique value is stored once in a dictionary; every row just stores a small integer pointing to that dictionary entry |

**When to use FixedString:** When text always has the same length (currency codes, country codes, hashes).

**When to use LowCardinality:** When a column has very few unique values (like country names — roughly 200 possible values) but millions of rows repeating them. The dictionary is built once, and each row just stores a tiny integer instead of the full text — much more compact.

### Example — String
```sql
CREATE TABLE txns
(
    id            UInt32,
    currency_code FixedString(3),        -- always exactly 3 chars, e.g. "USD"
    status        LowCardinality(String) -- only 4 possible values, repeats a lot
)
ENGINE = MergeTree
ORDER BY id;

INSERT INTO txns
SELECT
    number,
    ['USD','INR','EUR','GBP'][(number % 4) + 1],
    ['pending','shipped','delivered','cancelled'][(number % 4) + 1]
FROM numbers(1000000);
```

- `currency_code` uses `FixedString(3)` → every row takes exactly 3 bytes, no length overhead, since the value is always 3 characters.
- `status` uses `LowCardinality(String)` → ClickHouse builds a small dictionary (`pending=0`, `shipped=1`, `delivered=2`, `cancelled=3`) and each of the 1 million rows stores just a small integer instead of repeating the full word every time.

If plain `String` had been used for both, `currency_code` would pay extra length-overhead bytes per row, and `status` would repeat full text like `"delivered"` a million times instead of storing one small integer per row.

---

## 3. Date/Time Types

| Type | Bytes | Range |
|---|---|---|
| `Date` | 2 bytes | 1970-01-01 to 2149-06-06 (day precision only, no time) |
| `Date32` | 4 bytes | 1900-01-01 to 2299-12-31 (wider range when needed) |
| `DateTime` | 4 bytes | 1970-01-01 00:00:00 to 2106-02-07 (second precision) |
| `DateTime64(P)` | 8 bytes | Same range as DateTime, but P (precision) adds sub-second detail — milliseconds, microseconds, or nanoseconds |

**Note:** `DateTime64(3)` means millisecond precision, `DateTime64(6)` means microsecond precision. Higher P = finer precision (max P = 9, which is nanoseconds).

**Rule:** If you only need a date (no time), use `Date` (2 bytes) — using `DateTime` (4 bytes) or `DateTime64` (8 bytes) wastes space unnecessarily.

---

## 4. Semi-structured Types

These don't have a fixed byte size because their size depends on what's stored inside them.

| Type | Storage Behavior |
|---|---|
| `Array(T)` | A list of values in one column; internally stores an "offsets" array (8 bytes per row, marking where each list ends) plus the actual values in a column of type T |
| `Tuple(T1, T2, ...)` | A fixed set of differently-typed values grouped together; each element takes its own type's bytes (e.g. `Tuple(UInt32, String)` = 4 bytes + the string's variable size) |
| `Map(K, V)` | Key-value pairs; internally stored like `Array(Tuple(K, V))`, so it needs offsets plus storage for both keys and values |
| `JSON` | Schema-less nested data; storage is flexible — ClickHouse tries to "flatten" it into normal columns where possible, otherwise stores it more like a raw string |
| `Nested` | Groups several related columns together like an object; internally each field becomes its own Array column, sharing the same offsets |

**Rule:** Semi-structured types are powerful for flexibility, but if your data is actually flat and structured (like fixed key-value pairs), splitting it into separate plain columns is usually faster and more compact than using these types.

### Example — Semi-structured
```sql
CREATE TABLE customer_orders
(
    customer_id UInt32,
    order_date  Date,
    product_ids Array(UInt32)   -- a customer can buy multiple products in one order
)
ENGINE = MergeTree
ORDER BY customer_id;

INSERT INTO customer_orders VALUES
(101, '2026-09-01', [2001, 2002, 2005]),
(102, '2026-09-02', [2010]),
(103, '2026-09-03', [2001, 2003, 2004, 2007, 2009]);
```

- `product_ids` is an `Array(UInt32)` — each row can hold a different number of product IDs.
- Internally, ClickHouse stores an **offsets array** (8 bytes per row, marking where each customer's list ends) plus the actual `UInt32` values one after another.
- Customer 101 (3 products): offset entry (8 bytes) + 3 × 4 bytes (UInt32 values) = 20 bytes for that row's array data.
- Customer 103 (5 products): 8 bytes (offset) + 5 × 4 bytes = 28 bytes.

This is much simpler than creating a separate table just to link customers to products, especially when the list is small and read together most of the time.

---

## Special Type Behaviors

### LowCardinality(T) — Dictionary Encoding
Used when a column has **very few unique values** repeated across **many rows** (e.g. `country`, `status`). ClickHouse builds a dictionary mapping each unique value to a small integer ID, and stores only that integer per row instead of the full text — saving both storage and improving query speed (comparisons happen on integers, not strings).

### Nullable(T) — Overhead of Allowing NULLs
`Nullable(T)` allows NULL values in a column, but adds a hidden cost: ClickHouse maintains a **separate bitmap** per row tracking whether that row is NULL or not. This adds extra bytes-per-row overhead.

**Avoid Nullable on high-cardinality columns** — the bitmap overhead adds up quickly at scale. Where possible, use a default/empty value instead of NULL.

### Smallest Type Rule
Since ClickHouse is columnar, queries scan entire columns rather than individual rows. Choosing the smallest type that safely fits your data's range directly reduces the amount of data scanned per query — leading to both storage savings and faster performance.

---

## Quick Reference Summary

| Category | Examples |
|---|---|
| Numeric | `UInt32` (4B, 0 to ~4.29B), `Int16` (2B, -32,768 to 32,767), `Float64` (8B), `Decimal(10,2)` (8B, exact) |
| String | `String` (variable + overhead), `FixedString(N)` (exactly N bytes), `LowCardinality(String)` (small index + shared dictionary) |
| Date/Time | `Date` (2B, up to 2149), `DateTime` (4B, up to 2106), `DateTime64` (8B, sub-second precision) |
| Semi-structured | `Array` (8B offset + values), `Tuple` (sum of element sizes), `Map` (array of tuples), `JSON` (flexible), `Nested` (multiple parallel arrays) |

**Core Principle:** ClickHouse is a columnar database — the smaller the type, the less disk space it uses and the faster scans run. For every column, ask: "What's the actual range or length of this data?" and pick the smallest type that safely fits it.
