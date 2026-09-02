# ClickHouse for Beginners: Why It's So Fast

A simple explanation of ClickHouse's core architecture, based on its official [VLDB 2024 paper](https://clickhouse.com/docs/concepts/core-concepts/academic-overview).

---

## 1. What problem is ClickHouse solving?

Imagine you have a table with **billions of rows** (like every click on a website, ever), and you want to ask questions like:

> "What was the average page load time per country, last month?"

A normal database (like MySQL or Postgres) is built to quickly find *one row* — e.g., "get user #4521." That's called **OLTP** (transactional).

ClickHouse is built for the opposite problem: scanning and crunching *millions/billions of rows* at once to compute aggregates (sums, averages, counts). That's called **OLAP** (analytical). This difference in goal is why ClickHouse's internals look so different from a typical database.

---

## 2. The big idea: store data by column, not by row

Most databases store a row at a time, like a filing cabinet where each folder is one customer with all their info together.

ClickHouse stores data **column by column** instead. All the "country" values are stored together, all the "load_time" values are stored together, etc.

**Why this matters:** if your query only needs 2 columns out of 50, ClickHouse only has to read those 2 columns from disk — not all 50. Less data read = faster query.

---

## 3. Data is organized like a "write-once, merge-later" system

ClickHouse's storage engine (called **MergeTree**) works roughly like this:

- Every time you `INSERT` data, it gets written as a new little chunk called a **part** (parts are never edited after creation — they're immutable).
- In the background, ClickHouse continuously **merges** small parts into bigger ones, similar to how you might periodically consolidate small piles of paper into fewer, bigger folders.
- This keeps writes fast (just append new parts) while keeping reads fast too (fewer, bigger, sorted files to scan).

This pattern is known as an **LSM tree** design, used by many modern databases — ClickHouse adapts it specifically for analytics.

---

## 4. Worked example: how MergeTree actually behaves

Say you have a table sorted by `EventTime`, and you run three separate `INSERT` statements throughout the day.

**Step 1 — Inserts create separate parts**

```
INSERT 1 (09:00) -> Part_1  [EventTime 1-20]   50 rows
INSERT 2 (09:05) -> Part_2  [EventTime 21-40]  50 rows
INSERT 3 (09:07) -> Part_3  [EventTime 41-45]  15 rows
```

Each part is written to disk fully sorted by the primary key, and is never modified again. Right now, a `SELECT` query has to open and scan **3 separate parts**.

**Step 2 — A background merge kicks in**

ClickHouse's background merge process notices there are multiple small parts and merges two of them:

```
Part_1 + Part_2  ->  Part_1_2  [EventTime 1-40]  100 rows
```

This happens automatically, without blocking new inserts — you could be inserting `Part_4` at the very same moment.

**Step 3 — Merging continues over time**

```
Part_1_2 + Part_3  ->  Part_1_2_3  [EventTime 1-45]  115 rows
```

Eventually all the small parts consolidate into fewer, larger, sorted parts. A query now only has to scan **1 part** instead of 3+ — and because the merged part is still sorted by `EventTime`, ClickHouse can use its primary key index to jump straight to the range it needs.

**Why this design is clever:**

| Without merges | With background merges |
|---|---|
| Every `INSERT` = 1 more file to scan later | Small files get consolidated automatically |
| Reads get slower as inserts pile up | Reads stay fast because part count stays low |
| Writes are still fast (immutable, append-only) | Writes are still fast — merging never blocks inserts |

This is also how `DELETE`/`UPDATE`, TTL-based data expiry, and pre-aggregation (via special MergeTree variants like `SummingMergeTree` or `AggregatingMergeTree`) get applied — lazily, during merges, rather than instantly rewriting data on every write.

---

## 5. It skips data it doesn't need to read (the real magic)

This is arguably ClickHouse's most important trick. Instead of scanning every row, it uses three techniques to skip irrelevant data:

| Technique | Simple analogy |
|---|---|
| **Primary key index** | Like a book's index — data is sorted by a key (e.g., time), so ClickHouse can jump straight to the relevant section instead of reading page by page. |
| **Skipping indices** | Sticky notes on each chunk saying "this chunk only has values between X and Y" — so ClickHouse skips chunks that can't match your filter. |
| **Projections** | Pre-sorted duplicate copies of the table, optimized for different query patterns — like having the same phone book sorted alphabetically *and* by area code. |

Combined, these mean a query often only touches a tiny fraction of the total data on disk.

---

## 6. It processes data in batches, using all your CPU power

Rather than processing one row at a time (slow), ClickHouse:

- Processes rows in **batches/chunks** ("vectorized execution") so the CPU works more efficiently.
- Uses **SIMD** instructions — a CPU feature that can do the same math on multiple numbers simultaneously.
- Splits work across **all your CPU cores**, and if data is spread across multiple machines, across **all those machines too**.

Think of it like a factory: instead of one worker processing one item at a time, you have many workers on an assembly line, each handling many items in parallel.

---

## 7. It compresses data heavily

Since columns store similar values together (e.g., all timestamps, all country codes), they compress *very* well — much better than mixed row-based data. Less data on disk also means less data to read, which again means faster queries.

---

## 8. It's built for constant, high-speed data ingestion

ClickHouse is designed for use cases like logs, metrics, and events — data that keeps arriving nonstop. Its background merging process handles cleanup (aggregating, deleting old data, moving old data to cheaper storage) **without ever blocking new inserts**.

---

## 9. Architecture overview

ClickHouse is organized into four layers, plus shared services underneath:

```
+----------------------------------------------------+
|                   Access layer                      |
|   Native protocol / HTTP / MySQL & Postgres wire     |
+----------------------------+-------------------------+
                             |
+----------------------------v-------------------------+
|              Query processing layer                  |
|   Parse -> optimize -> execute (vectorized,           |
|   optionally compiled to native code)                 |
+----------------------------+-------------------------+
                             |
+----------------------------v-------------------------+
|                   Storage layer                       |
|  +--------------+  +--------------+  +--------------+ |
|  |  MergeTree*  |  | Special-     |  |  Virtual     | |
|  |  (primary,   |  | purpose      |  |  engines     | |
|  |  LSM-based)  |  | (dictionar-  |  |  (Kafka,     | |
|  |              |  | ies, Distri- |  |  S3, PG...)  | |
|  |              |  | buted)       |  |              | |
|  +--------------+  +--------------+  +--------------+ |
+----------------------------+-------------------------+
                             |
+----------------------------v-------------------------+
|                Integration layer                      |
|   Table functions / database engines / 90+ formats     |
+--------------------------------------------------------+

     Shared: threading, caching, access control,
              backups, monitoring
```

**What each layer does, in plain terms:**

- **Access layer** — how clients connect: the native binary protocol, HTTP, or by pretending to be MySQL/Postgres so existing tools/drivers just work.
- **Query processing layer** — takes your SQL, figures out the fastest way to execute it, and runs it using vectorized (batch-at-a-time) execution — occasionally even compiling parts of the query to native machine code for extra speed.
- **Storage layer** — where data actually lives. **MergeTree** (see the worked example above) is the default and most important engine. Special-purpose engines handle things like in-memory tables or the `Distributed` engine (which fans a query out across a cluster). Virtual engines let a "table" actually be a live pointer to Kafka, S3, or another database.
- **Integration layer** — lets ClickHouse read/write external systems directly (Kafka, Postgres, MySQL, S3, data lakes, 90+ file formats like CSV/JSON/Parquet) so you often don't need to copy data in before querying it.
- **Shared services** — threading, caching, access control, backups, and monitoring are used by every layer above, rather than being duplicated per-layer.

It all ships as a **single, statically-linked C++ binary** — no external dependencies to install — and can be deployed on-premise as a cluster, as ClickHouse Cloud, as a standalone CLI tool, or even embedded inside another application (via `chDB`).

---

## TL;DR — The core recipe

| Feature | What it buys you |
|---|---|
| Columnar storage | Read only the columns you need |
| MergeTree (LSM-based) | Fast inserts + background optimization (see worked example above) |
| Primary key / skipping indices / projections | Skip irrelevant data entirely |
| Vectorized + parallel execution | Use all CPU power, on all cores/machines |
| Heavy compression | Less disk I/O |
| Layered architecture | Clean separation: access -> query processing -> storage -> integration |
| Native integrations | Query external data without copying it first |

Put together, this is why ClickHouse can scan billions of rows and return an answer in under a second — it's not one trick, it's several complementary design choices all pointed at the same goal: **read as little as possible, as fast as possible.**
