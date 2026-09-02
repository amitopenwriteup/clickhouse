# MergeTree Family & Partitioning — Slide-by-Slide Explained

A plain-English walkthrough of every slide in `MergeTree_Family_and_Partitioning.pptx`, with a simple example added for each concept.

---

## Slide 1 — Title: "MergeTree Engine Family & Partitioning Strategy"

**What it is:** the cover slide. It frames the whole deck in one sentence: what each engine does, when to use it, and how to physically organize your data on disk.

---

## Slide 2 — What We'll Cover

**What it is:** a table of contents, split into two halves:

- **MergeTree family:** the base engine, `ReplacingMergeTree`, `SummingMergeTree`, `AggregatingMergeTree`, and other variants
- **Partitioning:** what it does, how it differs from `ORDER BY`, how to pick a partition key, and common mistakes

**Simple way to think about it:** the first half answers "what happens to rows that share a sort key when ClickHouse merges parts" (this is a per-*engine* decision). The second half answers "how do I physically group data on disk for fast bulk delete/archive" (this is a separate, per-*table* decision, layered on top of whichever engine you picked).

---

## Slide 3 — What Is the MergeTree Family?

**What it says:**
- `MergeTree` is ClickHouse's core storage engine family — nearly every production table uses one of its variants.
- Data is written in small parts, sorted by your `ORDER BY` key, then merged together in the background into larger parts (this is exactly the write-once/merge-later behavior covered earlier).
- **The key idea on this slide:** every variant does the *same* background merging. What differs between variants is only *what happens to rows that share the same sort key* when a merge runs.
- So picking a variant = picking a merge-time behavior, not just picking "a table engine" off a menu.

**The flow shown on the slide:**
```
INSERT  →  small parts on disk
BACKGROUND MERGE  →  parts combined by sort key
MERGED PART  →  engine-specific rules applied
```

**Simple example — same insert, different outcomes depending on the variant:**

Say two rows are inserted with the same `ORDER BY` key (`customer_id = 42`):

```
Row A: customer_id=42, name="Alice",  updated_at=09:00
Row B: customer_id=42, name="Alicia", updated_at=09:05   -- a correction
```

- Plain `MergeTree` → keeps **both** rows after merging (nothing is removed).
- `ReplacingMergeTree` → keeps only **one** row (the newer `updated_at`) after merging.
- `SummingMergeTree` → if there were numeric columns, they'd be **added together** into one row.

Same merge process, three different rules — that's the whole point of this slide.

---

## Slide 4 — MergeTree — the Base Engine (DEFAULT CHOICE)

**What it says:**
- The general-purpose engine: sorts data by `ORDER BY` and stores a sparse index for fast granule skipping (same primary-key-index idea from earlier).
- Rows are **never** merged, summed, or deduplicated — every inserted row is kept exactly as-is, forever.
- Best default for fact tables, logs, and anything you query/filter but don't need row-level de-dup or rollups for.

**Example from the slide:**

```sql
CREATE TABLE orders (
  order_id UInt64,
  order_date Date,
  amount Decimal(12,2)
) ENGINE = MergeTree
ORDER BY (order_date, order_id);
```

**Use it for:** general analytical tables, event/log storage — any table where every single row should be kept, including duplicates if they occur.

**When to reach past it:** the moment you find yourself thinking "actually I only want the latest row per customer" or "I want these numbers pre-summed" — that's your cue to look at the next two slides instead.

---

## Slide 5 — ReplacingMergeTree — De-duplication

**What it says:**
- During a background merge, keeps only the **last row per `ORDER BY` key** — or, if you gave it a version column, the row with the **highest value** in that column.
- **Important gotcha:** de-duplication only happens when parts actually merge, not instantly on insert — so duplicates can briefly exist right after writing, until a background merge catches up.
- For guaranteed-fresh reads *right now* (not waiting for a merge), query with `FINAL`, or compute the latest row yourself with `argMax(column, version)`.

**Example from the slide:**

```sql
CREATE TABLE customers_latest (
  customer_id UInt32,
  name String,
  updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY customer_id;
```

**Walking through it:** insert two rows for the same `customer_id`, and the one with the later `updated_at` wins once a merge happens:

```sql
INSERT INTO customers_latest VALUES (42, 'Alice', '2026-01-01 09:00:00');
INSERT INTO customers_latest VALUES (42, 'Alicia', '2026-01-01 09:05:00');

-- Right after inserting, both rows might still be visible (merge hasn't run yet)
SELECT * FROM customers_latest WHERE customer_id = 42;

-- Force a guaranteed-correct read without waiting for a background merge
SELECT * FROM customers_latest FINAL WHERE customer_id = 42;
-- → returns only the Alicia row
```

**Use it for:** slowly-changing dimensions, CDC (change-data-capture) feeds, or any "keep only the latest state per entity" pattern.

---

## Slide 6 — SummingMergeTree — Numeric Rollups

**What it says:**
- During a merge, rows sharing the same `ORDER BY` key have their **numeric columns automatically summed** into one row.
- Only simple addition is supported — for anything more complex (average, distinct count, etc.), you need `AggregatingMergeTree` instead (next slide).
- Commonly paired with a materialized view, so raw events get rolled up into daily/hourly totals as they're inserted, rather than needing a manual batch job.

**Example from the slide:**

```sql
CREATE TABLE daily_sales (
  order_date Date,
  product_category LowCardinality(String),
  total_sales Decimal(12,2)
) ENGINE = SummingMergeTree()
ORDER BY (order_date, product_category);
```

**Walking through it:**

```sql
INSERT INTO daily_sales VALUES ('2026-01-01', 'Electronics', 100.00);
INSERT INTO daily_sales VALUES ('2026-01-01', 'Electronics', 50.00);

-- After a background merge, these two rows collapse into one:
-- ('2026-01-01', 'Electronics', 150.00)
```

**Use it for:** pre-aggregated dashboards, daily/hourly totals, any "just add the numbers up" rollup.

---

## Slide 7 — AggregatingMergeTree — Flexible Rollups

**What it says:**
- Stores **partial aggregate states** (columns typed as `AggregateFunction(...)`) and merges those states together — supporting `avg`, `uniq`, `quantile`, and more, not just sum.
- Rows are written using **`-State` combinators** (e.g. `avgState`) and read back using **`-Merge` combinators** (e.g. `avgMerge`).
- More powerful than `SummingMergeTree`, but needs more setup — typically driven by a materialized view sitting over the raw data.

**Example from the slide:**

```sql
CREATE TABLE daily_stats (
  order_date Date,
  avg_amount AggregateFunction(avg, Decimal(12,2)),
  unique_customers AggregateFunction(uniq, UInt32)
) ENGINE = AggregatingMergeTree()
ORDER BY order_date;
```

**Simplified end-to-end example (writing and reading):**

```sql
-- Writing: use the -State suffix to produce a mergeable partial state
INSERT INTO daily_stats
SELECT order_date, avgState(amount), uniqState(customer_id)
FROM raw_orders
GROUP BY order_date;

-- Reading: use the -Merge suffix to finish the computation
SELECT order_date,
       avgMerge(avg_amount)      AS avg_order_value,
       uniqMerge(unique_customers) AS distinct_customers
FROM daily_stats
GROUP BY order_date;
```

**Why it's more work than `SummingMergeTree`:** you can't just `SELECT avg_amount` — the column holds an intermediate *state*, not a final number. You always finish the calculation with the matching `-Merge` function.

**Use it for:** dashboards needing average, distinct counts, quantiles, or any non-sum aggregate.

---

## Slide 8 — Other MergeTree Variants

**What it says:** a few specialized variants exist for narrower problems, once the four engines above don't fit:

| Engine | What it does | Use it for |
|---|---|---|
| `CollapsingMergeTree` | Cancels row pairs using a `+1`/`-1` sign column during merge | High-frequency updates modeled as insert + cancel |
| `VersionedCollapsingMergeTree` | Like Collapsing, but tolerant of out-of-order inserts via a version column | Same, but with concurrent/out-of-order writers |
| `GraphiteMergeTree` | Thins out old metric data using rollup rules, Graphite-style | Time-series metrics with age-based downsampling |

**Simple example — the Collapsing pattern:**

```sql
-- sign = +1 means "this row is active", sign = -1 means "cancel the matching row"
CREATE TABLE page_views (
  page_id UInt32,
  views UInt32,
  sign Int8
) ENGINE = CollapsingMergeTree(sign)
ORDER BY page_id;

INSERT INTO page_views VALUES (1, 100, 1);   -- original count
INSERT INTO page_views VALUES (1, 100, -1);  -- cancel it
INSERT INTO page_views VALUES (1, 150, 1);   -- corrected count

-- After merging, the +1/-1 pair for the original 100 cancels out,
-- leaving only the corrected row (1, 150, 1)
```

**Takeaway from the slide:** these solve real but specific problems — reach for them only once `MergeTree`, `ReplacingMergeTree`, `Summing`, or `Aggregating` genuinely don't fit your case.

---

## Slide 9 — Which Engine Should You Use?

**What it is:** a decision table tying everything together:

| If you need to… | Use | Because |
|---|---|---|
| Store every row, query/filter freely | `MergeTree` | No merge-time row manipulation — simplest, fastest default |
| Keep only the latest version per key | `ReplacingMergeTree` | Duplicates collapse automatically during merges |
| Maintain running numeric totals | `SummingMergeTree` | Numeric columns sum for matching keys, no MV logic needed |
| Roll up avg / uniq / quantile etc. | `AggregatingMergeTree` | Stores merge-able partial states for any aggregate function |
| Model frequent updates as cancel/replace | `(Versioned)CollapsingMergeTree` | Sign-column semantics purpose-built for this pattern |

**Rule of thumb from the slide (worth memorizing):** start with plain `MergeTree`. Only reach for a variant once you can name the *exact* merge-time behavior you need — "I want duplicates gone" → Replacing; "I want these numbers pre-summed" → Summing; "I want averages/distinct counts pre-computed" → Aggregating.

---

## Slide 10 — Partitioning — What It Actually Does

**What it says:**
- `PARTITION BY` splits a table's data into separate **physical** parts on disk — commonly by month or day.
- It exists for **data lifecycle management**: fast bulk drop/archive of old data, and pruning whole partitions before a query even starts.
- It is a **physical/storage-level split**, not a query index — that job belongs to `ORDER BY`.

**The side-by-side comparison from the slide:**

| `PARTITION BY` | `ORDER BY` |
|---|---|
| Lifecycle management | Query filtering |
| Drop/archive whole partitions instantly | Sparse index for skipping granules within a partition during a scan |
| Prunes entire partitions before a scan starts | Prunes *within* a partition, during the scan |

**Simple mental model:** `PARTITION BY` decides which *folders* your data lives in (and lets you delete a whole folder instantly). `ORDER BY` decides how data is *sorted inside each folder*, so a query can jump straight to the relevant rows once it's already looking inside one.

---

## Slide 11 — Choosing a Partition Key

**What it says:**
- Rule of thumb: partition by **month** for most time-series workloads; by **day** only for very high ingest volumes.
- A good partition key is **coarse and low-cardinality** — the goal is a manageable number of large partitions, not many small ones.
- Avoid partitioning by an exact timestamp or any high-cardinality column — it creates excessive tiny parts and expensive merges.

**The good vs. bad comparison from the slide:**

| Good: coarse & low-cardinality | Bad: fine-grained & high-cardinality |
|---|---|
| `PARTITION BY toYYYYMM(log_time)` | `PARTITION BY log_time` or `PARTITION BY user_id` |
| A handful of large partitions per year — fast pruning, cheap background merges | Thousands of tiny partitions — merge overhead outweighs any pruning gain |

**Simple example showing why "bad" is bad:**

```sql
-- Good: ~12 partitions per year
CREATE TABLE logs_good (
  log_time DateTime,
  message String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(log_time)
ORDER BY log_time;

-- Bad: one partition PER SECOND of data — millions of tiny partitions
CREATE TABLE logs_bad (
  log_time DateTime,
  message String
) ENGINE = MergeTree
PARTITION BY log_time
ORDER BY log_time;
```

**Why this breaks things:** each partition manages its own set of parts and merges independently. Millions of partitions means millions of tiny part sets that can never usefully merge with each other (remember: merging only ever happens *within* a partition) — so you end up with the exact "too many small parts" problem the MergeTree merge process was designed to prevent in the first place.

---

## Slide 12 — Partitioning in Practice

**What it is:** four practical command groups, shown side by side on the slide.

**1. Define & inspect:**

```sql
CREATE TABLE logs_partitioned (
  log_time DateTime,
  level LowCardinality(String),
  message String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(log_time)
ORDER BY (level, log_time);

-- See what partitions actually exist
SELECT partition, name, rows
FROM system.parts WHERE table = 'logs_partitioned';
```

**2. Prune with a filter — verify it's actually working:**

```sql
EXPLAIN indexes = 1
SELECT count() FROM logs_partitioned
WHERE log_time >= now() - INTERVAL 1 DAY;
```
`EXPLAIN indexes = 1` shows you which partitions/granules ClickHouse actually decided to skip — a good way to confirm your partition key is doing its job before trusting it in production.

**3. Drop old data instantly:**

```sql
ALTER TABLE logs_partitioned DROP PARTITION 202405;
```
This deletes an entire month's data in one metadata operation — no slow row-by-row `DELETE` needed, because it's just removing whole physical parts from disk.

**4. Automate with TTL:**

```sql
ALTER TABLE logs_partitioned
  MODIFY TTL log_time + INTERVAL 90 DAY;
```
Old partitions now expire automatically after 90 days — you don't have to remember to run the `DROP PARTITION` command yourself.

---

## Slide 13 — Recap

**MergeTree family, one line each:**
- `MergeTree` — keep every row (the default).
- `ReplacingMergeTree` — de-dup on merge.
- `SummingMergeTree` — sum numeric columns.
- `AggregatingMergeTree` — any aggregate, via states.
- `Collapsing` / `VersionedCollapsingMergeTree` — cancel/replace via a sign column.

**Partitioning, one line each:**
- A physical split for lifecycle management, not a query index.
- Partition coarse and low-cardinality (month, not exact timestamp).
- `ORDER BY` still does the query filtering *within* each partition.

**Closing rule of thumb from the slide:** start simple — `MergeTree` + monthly partitioning covers most tables. Add a variant only when the merge behavior you need is explicit and you can name exactly what it should do.

---

## One-page cheat sheet

| Slide | Topic | One-line takeaway |
|---|---|---|
| 3 | MergeTree family concept | Same background merge for all variants — only the merge *behavior* differs |
| 4 | MergeTree (base) | Keeps every row, no merge-time changes — the default |
| 5 | ReplacingMergeTree | Keeps only the newest row per key — use `FINAL` for guaranteed-fresh reads |
| 6 | SummingMergeTree | Auto-sums numeric columns for matching keys |
| 7 | AggregatingMergeTree | Stores partial states (`-State`), finish with `-Merge` — any aggregate function |
| 8 | Other variants | Collapsing/Versioned/Graphite — narrow, specific problems only |
| 9 | Choosing an engine | Start with `MergeTree`; pick a variant only when you can name the exact merge behavior needed |
| 10 | What partitioning does | Physical, lifecycle-focused split — not a query index |
| 11 | Choosing a partition key | Coarse and low-cardinality (month) — never exact timestamp or high-cardinality columns |
| 12 | Partitioning in practice | `DROP PARTITION` = instant bulk delete; `TTL` = automatic expiry |
