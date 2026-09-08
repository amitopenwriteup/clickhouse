# How ClickHouse Reassembles Rows from Columnar Storage

## The Core Question

ClickHouse stores data **column-by-column** on disk, but returns query results as **rows**. How does it bridge the two without losing the performance benefits of columnar storage?

The short answer: **rows are never actually stored or fetched individually** — they're reconstructed only at the very last step, using fast, vectorized (batch) operations on entire arrays of values. Nothing is done one row at a time.

---

## 1. Storage Layout — Columns, Not Rows

For a table like:

```sql
CREATE TABLE logs (
  event_time DateTime,
  service String,
  level String,
  status_code UInt16,
  latency_ms UInt32,
  message String
) ENGINE = MergeTree ORDER BY event_time;
```

Each column is stored in its **own file**, physically separate from the others:

```
event_time.bin
service.bin
level.bin
status_code.bin
latency_ms.bin
message.bin
```

Every column file stores its values in the **same row order**. There is no explicit "row" structure on disk — the *position* (index) of a value is what implicitly ties it to the same logical row across all column files.

```
Position:        0        1        2        3
service.bin:     payment  payment  orders   payment
latency_ms.bin:  52       152      89       252
```

Position 3 in `service.bin` and position 3 in `latency_ms.bin` belong to the same row — purely because they share the same index. No joins, no pointers.

---

## 2. Granules and Marks — The Sparse Index

Column files aren't read as one giant stream. Data is split into **granules** (default: 8,192 rows each). A separate **marks file** (`.mrk`) records the byte offset where each granule begins in the compressed column file.

This enables two things:
- **Skip entire granules** that can't match the query's `WHERE` clause (using the primary key's sparse index).
- **Jump directly** to the relevant byte offset instead of scanning from the start of the file.

---

## 3. Query Execution — Vectorized, Not Row-by-Row

When you run:

```sql
SELECT * FROM logs WHERE service = 'payment';
```

Here's the actual sequence:

### Step 1 — Read the filter column's granule in full
ClickHouse reads the **entire granule** (e.g., all 8,192 values) of `service` in one sequential, contiguous read — not selective positions.

### Step 2 — Evaluate the predicate across the whole array at once
It compares `service == 'payment'` across all 8,192 values in a single vectorized (SIMD-friendly) pass, producing a **filter mask** — the list of matching positions:

```
matching positions: [0, 1, 3, 4, 7, ...]
```

This is a bulk operation, not a per-row branch/check loop.

### Step 3 — Read every other selected column's granule in full, then apply the same mask
For `latency_ms`, `event_time`, `message`, etc., ClickHouse:
1. Reads the **entire granule** sequentially (cheap I/O).
2. Applies the **same positional mask** from Step 2 to extract only the matching values — using a fast bulk filter-copy operation (`IColumn::filter()`), not individual lookups.

Because every column shares row order, the *same* mask works identically across all of them.

---

## 4. Blocks — The In-Memory Unit

Filtered column arrays are grouped into a **`Block`** — the fundamental unit ClickHouse operates on internally:

```
Block {
  IColumn(event_time)   →  [values...]
  IColumn(service)      →  [values...]
  IColumn(latency_ms)   →  [values...]
  ...
}
```

All downstream operations — aggregation, sorting, further filtering — happen on this columnar `Block`. ClickHouse never constructs row objects like `{event_time: ..., service: ...}` internally during processing.

---

## 5. Row Reconstruction — Only at Output Time

The **only** point where "rows" are actually assembled is the final **output formatter** (e.g., `Pretty`, `JSON`, `CSV`, or the binary Native protocol used by clients/UIs).

The formatter walks position `i` from `0` to the block's row count, and for each `i`, reads:

```
column[0][i], column[1][i], column[2][i], ...
```

across all selected columns — emitting that as one output row. This happens on the **already-filtered, already-shrunk** result set, not the full original granule.

---

## Full Execution Chain (Summary)

```
Disk: separate column files, aligned by row position, chunked into granules
   ↓
Sparse index / marks prune irrelevant granules
   ↓
Read + decompress entire needed granules per column (sequential I/O)
   ↓
Load into IColumn arrays → grouped into a Block
   ↓
Vectorized predicate evaluation → filter mask
   ↓
Apply mask uniformly across all columns (bulk filter-copy)
   ↓
(Aggregation / sorting / etc. — still columnar)
   ↓
Output formatter walks position i across columns → emits row i
```

---

## Why This Is Fast

| Aspect | Row-based DB | ClickHouse (columnar) |
|---|---|---|
| I/O per query | Reads full rows (all columns) even if only 2 are needed | Reads only the columns actually selected |
| Filtering | Per-row branching/checks | Vectorized bulk comparison over arrays |
| Row assembly | Rows already exist on disk | Reconstructed only at output, on the already-filtered subset |
| Access pattern | Can involve random seeks | Always sequential reads within granules |

The key insight: **columnar storage optimizes what gets read and how filtering is computed; row output is just a cheap final transpose step over a much smaller, already-reduced dataset.** This is why ClickHouse can scan hundreds of thousands of rows in milliseconds — the expensive work (I/O, filtering) happens in bulk on columns, and row reconstruction only touches the small number of rows that actually made it through.
