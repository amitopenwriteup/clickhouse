Below is a **single Markdown (`.md`) document** you can use directly as beginner-friendly training material.

# ClickHouse MergeTree — Complete Beginner Explanation

## 1. What is MergeTree?

**MergeTree is the most important storage engine in ClickHouse.**

When we create a ClickHouse table, we commonly use:

```sql
CREATE TABLE logs
(
    timestamp DateTime,
    server String,
    status UInt16,
    response_time UInt32,
    message String
)
ENGINE = MergeTree
ORDER BY timestamp;
```

Here:

```text
ENGINE = MergeTree
```

tells ClickHouse:

> "Store and manage the data using the MergeTree storage architecture."

MergeTree is designed for:

* Very large datasets
* Fast INSERT operations
* Fast analytical SELECT queries
* Data skipping
* Background merging
* Efficient compression
* Parallel query processing

---

# 2. Why do we need MergeTree?

Imagine we are collecting web server logs.

Every second, thousands of logs arrive:

```text
10:00:01 web01 200 /login
10:00:01 web02 500 /payment
10:00:02 web01 200 /home
10:00:02 web03 404 /test
...
```

After one day:

```text
100 million rows
```

After one month:

```text
3 billion rows
```

We might ask:

```sql
SELECT count(*)
FROM logs
WHERE status = 500;
```

ClickHouse needs a way to:

1. Store huge amounts of data
2. Insert new data efficiently
3. Organize the data
4. Find relevant data quickly
5. Avoid reading unnecessary data

**MergeTree provides the foundation for doing this.**

---

# 3. The most important idea: Parts

When data is inserted into a MergeTree table, ClickHouse does not normally rewrite the entire table.

Instead, the inserted data becomes a **Part**.

For example:

```text
INSERT 1,000 rows
        ↓
     Part 1
```

Then:

```text
INSERT 2,000 rows
        ↓
     Part 2
```

Then:

```text
INSERT 5,000 rows
        ↓
     Part 3
```

The table now looks conceptually like:

```text
logs
 │
 ├── Part 1 → 1,000 rows
 ├── Part 2 → 2,000 rows
 └── Part 3 → 5,000 rows
```

So:

> **A Part is a physical chunk of data belonging to a MergeTree table.**

---

# 4. Why use Parts?

Suppose we already have:

```text
1 billion rows
```

and another 10,000 rows arrive.

A bad design would be:

```text
Existing 1 billion rows
          +
New 10,000 rows
          ↓
Rewrite everything
```

That would be extremely expensive.

MergeTree instead does:

```text
Existing data
      ↓
  remains as it is

New 10,000 rows
      ↓
   New Part
```

So the INSERT can happen quickly.

Conceptually:

```text
Before:

Part A
Part B
Part C


New INSERT

        ↓


After:

Part A
Part B
Part C
Part D ← new data
```

---

# 5. What happens if we keep creating Parts?

Suppose we have many INSERTs:

```text
Part 1
Part 2
Part 3
Part 4
Part 5
Part 6
Part 7
Part 8
...
Part 1000
```

Having thousands of tiny parts isn't ideal.

Queries would potentially need to deal with many separate pieces.

This is where the **Merge** in MergeTree becomes important.

---

# 6. Background merging

ClickHouse runs background processes that merge parts.

For example:

```text
Part 1 ──┐
         ├──→ Part 10
Part 2 ──┘
```

And:

```text
Part 3 ──┐
         ├──→ Part 11
Part 4 ──┘
```

Then later:

```text
Part 10 ──┐
          ├──→ Part 20
Part 11 ──┘
```

So:

```text
Small Parts
     ↓
Background Merge
     ↓
Larger Parts
```

This happens automatically in the background.

---

# 7. Why is it called MergeTree?

The name becomes easier to understand now.

Data arrives:

```text
       Part 1
       Part 2
       Part 3
       Part 4
```

Parts are merged:

```text
Part 1 + Part 2
       ↓
    Part A

Part 3 + Part 4
       ↓
    Part B
```

Then:

```text
Part A + Part B
       ↓
   Larger Part
```

The structure resembles a tree of progressively merged data.

Hence:

> **MergeTree = data stored as parts that are continuously merged in the background.**

---

# 8. Parts are generally immutable

One important concept is that existing MergeTree parts are generally **immutable**.

If new data arrives, ClickHouse creates another part.

It doesn't normally modify an existing part directly.

For example:

```text
Existing:

Part 1
 └── 1,000 rows
```

New INSERT:

```text
Part 1
 └── 1,000 rows

Part 2
 └── 500 new rows
```

Instead of:

```text
Part 1
 └── modify existing 1,000 rows
```

This design is useful for large-scale analytical workloads.

---

# 9. What is inside a Part?

A Part contains much more than just rows.

Conceptually:

```text
Part
│
├── Column data
│
├── Data marks
│
├── Index information
│
├── Metadata
│
└── Other files required for reading the data
```

Because ClickHouse is a **column-oriented database**, columns are stored separately.

For example:

```text
Part
│
├── timestamp
├── server
├── status
├── response_time
└── message
```

This is important for query performance.

---

# 10. ClickHouse stores data by columns

Suppose our data looks like this:

```text
ID    Server    Status    ResponseTime
1     web01     200       20
2     web02     500       100
3     web01     200       30
4     web03     404       50
```

A row-oriented database conceptually stores:

```text
1, web01, 200, 20
2, web02, 500, 100
3, web01, 200, 30
4, web03, 404, 50
```

ClickHouse stores columns separately:

```text
ID:
1
2
3
4

Server:
web01
web02
web01
web03

Status:
200
500
200
404

ResponseTime:
20
100
30
50
```

---

# 11. Why column storage helps

Suppose we run:

```sql
SELECT avg(ResponseTime)
FROM logs;
```

We only need:

```text
ResponseTime
```

We don't necessarily need:

```text
Server
Status
Message
URL
User
Country
```

So ClickHouse can read primarily the required column.

Conceptually:

```text
100 columns
     ↓
Query needs 1 column
     ↓
Read 1 relevant column
```

This can dramatically reduce disk I/O for analytical queries.

---

# 12. Compression

Column storage also helps compression.

Imagine:

```text
Status

200
200
200
200
200
200
500
200
200
200
```

There are many repeated values.

Compression can work efficiently on this kind of data.

Therefore:

```text
Column storage
      ↓
Similar values grouped together
      ↓
Better compression
      ↓
Less disk space
      ↓
Less data to read from disk
```

---

# 13. What are Granules?

A Part can be divided into smaller logical groups called **granules**.

Think of it like this:

```text
Part
│
├── Granule 1
├── Granule 2
├── Granule 3
├── Granule 4
└── Granule 5
```

A granule represents a group of rows.

The default index granularity in MergeTree is commonly **8,192 rows**, although this can be configured and the actual number of rows in a granule can vary.

The important beginner idea is:

> **A granule is a small group of rows that ClickHouse can consider as a unit when reading or skipping data.**

---

# 14. Why do we need Granules?

Imagine:

```text
1 billion rows
```

We don't want ClickHouse to inspect every row individually just to determine whether it contains useful data.

Instead:

```text
1 billion rows
      ↓
many granules
      ↓
identify relevant granules
      ↓
read only those areas
```

For example:

```text
Part
│
├── Granule 1 → SKIP
├── Granule 2 → SKIP
├── Granule 3 → SKIP
├── Granule 4 → READ
├── Granule 5 → READ
└── Granule 6 → SKIP
```

This is called **data skipping** or **data pruning**.

---

# 15. ORDER BY is extremely important

Consider:

```sql
CREATE TABLE logs
(
    timestamp DateTime,
    server String,
    status UInt16
)
ENGINE = MergeTree
ORDER BY timestamp;
```

The:

```sql
ORDER BY timestamp
```

is very important.

It determines the **sorting key** used to organize rows within parts.

For example:

```text
timestamp

10:00
10:01
10:02
10:03
10:04
10:05
...
```

The data is organized according to the sorting key.

This helps ClickHouse identify ranges of data that may be relevant to a query.

---

# 16. ORDER BY is not just about displaying results

A common beginner mistake is to think:

```sql
ORDER BY timestamp
```

only means:

> "Return the query results sorted by timestamp."

For MergeTree tables, the `ORDER BY` in the table definition is much more important.

Example:

```sql
ENGINE = MergeTree
ORDER BY timestamp
```

defines the table's **sorting key**.

It influences:

* Physical organization of data
* Primary index
* Data skipping
* Query performance

Therefore:

> **Choosing the right ORDER BY is one of the most important ClickHouse design decisions.**

---

# 17. Primary index

MergeTree uses a **sparse primary index**.

Don't think of it like a traditional database index containing an entry for every row.

Instead, ClickHouse stores index information associated with groups of rows / granules.

Conceptually:

```text
Granule 1 → timestamp around 10:00
Granule 2 → timestamp around 10:10
Granule 3 → timestamp around 10:20
Granule 4 → timestamp around 10:30
```

Now suppose we query:

```sql
WHERE timestamp >= '10:20'
```

ClickHouse can determine that earlier ranges aren't relevant:

```text
Granule 1 → SKIP
Granule 2 → SKIP
Granule 3 → READ
Granule 4 → READ
```

This is much better than blindly scanning everything.

---

# 18. Complete query example

Suppose we have:

```sql
CREATE TABLE logs
(
    timestamp DateTime,
    server String,
    status UInt16,
    response_time UInt32
)
ENGINE = MergeTree
ORDER BY timestamp;
```

And we have:

```text
1 billion rows
```

Now run:

```sql
SELECT avg(response_time)
FROM logs
WHERE timestamp >= '2026-09-02 10:00:00'
  AND timestamp <  '2026-09-02 11:00:00';
```

ClickHouse can conceptually do:

```text
                 SQL Query
                     │
                     ↓
              Check sorting key
                     │
                     ↓
              Primary index
                     │
                     ↓
          Identify relevant ranges
                     │
             ┌───────┴───────┐
             ↓               ↓
          Skip data       Read data
                             │
                             ↓
                    Read response_time
                             │
                             ↓
                    Process in batches
                             │
                             ↓
                          AVG()
                             │
                             ↓
                           Result
```

The key point is:

> **ClickHouse tries to avoid reading data that cannot contribute to the answer.**

---

# 19. MergeTree and INSERT performance

Suppose logs arrive continuously:

```text
Application
    │
    ├── Log
    ├── Log
    ├── Log
    └── Log
         ↓
     ClickHouse
         ↓
       Part
```

New data can become new parts.

This avoids having to constantly reorganize the entire table for every INSERT.

So MergeTree is very suitable for workloads such as:

* Application logs
* Server logs
* Security events
* Metrics
* Clickstream data
* IoT data
* Financial events
* Observability data

---

# 20. Merge happens in the background

An important point:

**INSERT and Merge are different activities.**

When data arrives:

```text
INSERT
  ↓
New Part
```

Later:

```text
Background merge
  ↓
Combine Parts
```

The INSERT doesn't normally wait for all possible future merges to finish.

This allows ClickHouse to accept data while background processes maintain the storage layout.

---

# 21. Example with millions of log records

Imagine an observability platform receiving:

```text
1 million logs per minute
```

After several minutes:

```text
Part 1
Part 2
Part 3
Part 4
Part 5
...
```

ClickHouse keeps accepting data.

Meanwhile background merging may do:

```text
Part 1 + Part 2
        ↓
     Part 10

Part 3 + Part 4
        ↓
     Part 11

Part 10 + Part 11
        ↓
     Part 20
```

This happens continuously.

The application doesn't have to manually combine these parts.

---

# 22. What happens during a Merge?

Conceptually:

```text
Part 1
  1
  3
  5

Part 2
  2
  4
  6
```

If the sorting key is:

```text
ORDER BY id
```

the merged part becomes:

```text
Part 3

1
2
3
4
5
6
```

The resulting part is properly organized according to the sorting key.

The old parts can then eventually be removed once the new merged part is safely committed.

---

# 23. Does MergeTree update existing rows?

This is an important distinction.

MergeTree is primarily designed around **immutable data parts**.

If you need UPDATE/DELETE-like behavior, ClickHouse provides mechanisms such as:

* `ALTER TABLE ... UPDATE`
* `ALTER TABLE ... DELETE`
* `ReplacingMergeTree`
* `CollapsingMergeTree`
* `VersionedCollapsingMergeTree`
* Lightweight deletes

These have different behavior and performance characteristics.

For a beginner, remember:

> **MergeTree is optimized primarily for append-oriented analytical workloads rather than frequent row-by-row updates.**

---

# 24. What is ReplacingMergeTree?

MergeTree has several specialized variants.

One commonly used variant is:

```text
ReplacingMergeTree
```

For example:

```sql
ENGINE = ReplacingMergeTree(version)
```

It can be useful when data arrives multiple times and you want newer versions to eventually replace older versions.

Conceptually:

```text
User 101 → version 1
User 101 → version 2
User 101 → version 3
```

During merging, older versions can be removed according to the engine's rules.

But this is a more advanced topic.

Start with:

```text
MergeTree
```

before learning:

```text
ReplacingMergeTree
CollapsingMergeTree
AggregatingMergeTree
SummingMergeTree
```

---

# 25. MergeTree vs traditional database

Consider PostgreSQL/MySQL and ClickHouse.

### Traditional OLTP database

Typical workload:

```text
INSERT one row
UPDATE one row
DELETE one row
SELECT one customer
```

### ClickHouse MergeTree

Typical workload:

```text
INSERT millions of events
        ↓
Store efficiently
        ↓
Analyze billions of events
        ↓
GROUP BY
COUNT
SUM
AVG
```

So MergeTree is designed around a different workload.

---

# 26. The complete MergeTree architecture

You can remember it like this:

```text
                     MergeTree
                         │
             ┌───────────┴───────────┐
             │                       │
          INSERT                  SELECT
             │                       │
             ↓                       ↓
        Create Part            Use sorting key
             │                       │
             ↓                       ↓
       Store columns          Primary index
             │                       │
             ↓                       ↓
        Data granules          Skip granules
             │                       │
             ↓                       ↓
      Background merge        Read needed columns
             │                       │
             ↓                       ↓
      Larger Parts             Vectorized processing
                                     │
                                     ↓
                                   Result
```

---

# 27. One complete example

Let's create a log table:

```sql
CREATE TABLE logs
(
    timestamp DateTime,
    server String,
    status UInt16,
    response_time UInt32,
    message String
)
ENGINE = MergeTree
ORDER BY timestamp;
```

Now insert:

```sql
INSERT INTO logs VALUES
('2026-09-02 10:00:01', 'web01', 200, 20, 'OK'),
('2026-09-02 10:00:02', 'web02', 500, 100, 'Error'),
('2026-09-02 10:00:03', 'web01', 200, 30, 'OK');
```

ClickHouse creates a part:

```text
logs
 │
 └── Part 1
       │
       ├── timestamp
       ├── server
       ├── status
       ├── response_time
       └── message
```

Another INSERT:

```sql
INSERT INTO logs VALUES
('2026-09-02 10:01:01', 'web03', 404, 50, 'Not Found'),
('2026-09-02 10:01:02', 'web01', 500, 200, 'Error');
```

Now:

```text
logs
 │
 ├── Part 1
 └── Part 2
```

Later:

```text
Part 1 + Part 2
       ↓
     Merge
       ↓
    Part 3
```

The table eventually has fewer, larger parts.

---

# 28. Why MergeTree is fast

There isn't one magic feature.

Several design decisions work together:

```text
                    MergeTree
                        │
       ┌────────────────┼────────────────┐
       ↓                ↓                ↓
 Column storage    Sorting key       Parts
       │                │                │
       ↓                ↓                ↓
Read fewer        Find relevant      Fast inserts
columns           data
       │                │                │
       ↓                ↓                ↓
Compression       Primary index     Background
                                     merging
       │                │
       └────────┬───────┘
                ↓
         Less data read
                ↓
        Faster analytics
```

---

# 29. The most important concept: "Read less"

When learning ClickHouse, remember this principle:

> **The fastest data to process is data you don't have to read.**

MergeTree helps ClickHouse achieve this through:

```text
ORDER BY
   ↓
Sorting
   ↓
Primary index
   ↓
Granules
   ↓
Data skipping
   ↓
Read only relevant data
```

And then:

```text
Column storage
   ↓
Read only required columns
```

And:

```text
Compression
   ↓
Read less physical data
```

And:

```text
Vectorized processing
   ↓
Process data efficiently
```

---

# 30. Beginner mental model

Imagine a huge warehouse.

### Parts = boxes

```text
┌─────────┐
│ Part 1  │
└─────────┘

┌─────────┐
│ Part 2  │
└─────────┘
```

### Granules = sections inside boxes

```text
Part
│
├── Section 1
├── Section 2
├── Section 3
└── Section 4
```

### ORDER BY = how the warehouse organizes things

```text
timestamp
    ↓
old → new
```

### Primary index = warehouse map

It tells ClickHouse approximately where the required data is.

### Data skipping = don't open irrelevant boxes

```text
Box 1 → Not needed → SKIP
Box 2 → Not needed → SKIP
Box 3 → Needed → READ
```

### Merge = combine small boxes

```text
Small box + Small box
          ↓
      Large box
```

This is a good mental model for understanding MergeTree.

---

# 31. Final summary

If someone asks:

> **What is MergeTree?**

Answer:

> **MergeTree is ClickHouse's primary family of storage engines. It stores incoming data as immutable parts, organizes the data according to a sorting key, divides data into granules, maintains sparse index information for data skipping, and continuously merges smaller parts into larger ones in the background. This design allows ClickHouse to handle very large analytical datasets efficiently.**

The simplified flow is:

```text
              INSERT
                 │
                 ↓
            Create Part
                 │
                 ↓
          Column-oriented data
                 │
                 ↓
             Granules
                 │
                 ↓
        Background Merge
                 │
                 ↓
          Larger Parts


SELECT
  │
  ↓
Use ORDER BY / sorting key
  │
  ↓
Primary index
  │
  ↓
Skip irrelevant granules
  │
  ↓
Read required columns
  │
  ↓
Vectorized processing
  │
  ↓
Result
```

## The 5 things to remember

1. **MergeTree = storage engine**
2. **INSERT creates Parts**
3. **Parts are merged in the background**
4. **ORDER BY controls how data is organized**
5. **Indexes + granules help ClickHouse skip unnecessary data**

Once these five concepts are clear, the deeper ClickHouse architecture becomes much easier to understand.

This is suitable as a **single `.md` training note**. The next topic I’d recommend learning is **`ORDER BY` + Primary Key + Granules**, because that is where MergeTree's query-performance advantage becomes much clearer.
