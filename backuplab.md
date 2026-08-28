# ClickHouse SQL Fundamentals — Hands-On Lab

**Dataset:** `github_events` on the ClickHouse Playground
**Audience:** Intermediate engineers (comfortable with SQL, new to ClickHouse)
**Duration:** ~90 minutes
**Format:** Live, run-along in browser

---

## Before You Start

**Where to run this:** Open the public ClickHouse Playground (`sql.clickhouse.com` or `play.clickhouse.com`). No login or credentials are needed for the read-only demo account. All queries in this lab target the `github_events` table.

> ⚠️ **Presenter note:** Playground datasets and hostnames occasionally change. Run **Step 1** first, live, to confirm `github_events` exists and to see current row counts and date range. This also doubles as a teaching moment: *always discover the schema before you trust the docs.*

### Agenda

| Time | Block |
|---|---|
| 0:00–0:10 | Setup & Orientation |
| 0:10–0:30 | SQL Fundamentals Warm-up |
| 0:30–0:50 | Aggregations & Time Series |
| 0:50–0:55 | Break |
| 0:55–1:20 | Intermediate Techniques (CTEs, windows, joins, arrays) |
| 1:20–1:30 | Performance & Query-Safety Habits, Wrap-up |

### Learning Objectives

By the end of this lab you will be able to:
- Write ClickHouse SQL for filtering and aggregation
- Use ClickHouse-specific features: arrays, `ARRAY JOIN`, approximate aggregates, window functions
- Read an `EXPLAIN` plan to reason about query performance

---

## Part 0 — Setup & Orientation *(10 min)*

### Why ClickHouse Feels Different

- ClickHouse is **columnar**: a query touching 3 columns out of 60 only reads those 3 columns off disk — unlike row-store databases.
- Tables use the **MergeTree** engine family: data is physically sorted on disk by an `ORDER BY` (sort) key. Filtering on a *prefix* of that key lets ClickHouse skip whole blocks ("granules") instead of scanning everything.
- This table's sort key is **`(event_type, repo_name, created_at)`** — the whole lab leans on that.
- There are no indexes to "add" for filtering, the way there are in Postgres. The sort key *is* the primary index.

### Step 1 — Discover What's Actually in the Playground

Never assume schema — discover it live.

```sql
SHOW DATABASES;

SHOW TABLES;

DESCRIBE TABLE github_events;

-- Cheap: MergeTree tracks row counts in metadata, this does not scan data
SELECT count() AS total_events FROM github_events;

-- Anchor every "recent" example on the data's own freshness, not today()
SELECT min(created_at) AS earliest, max(created_at) AS latest
FROM github_events;
```

**Debrief:** Note the actual row count and date range on screen. You'll reference `max(created_at)` instead of `now()` throughout — the data has a fixed ingestion cutoff; it isn't live-streaming.

---

## Part 1 — SQL Fundamentals Warm-up *(20 min)*

### Step 2 — First Look at the Data

```sql
SELECT event_type, actor_login, repo_name, created_at, action
FROM github_events
LIMIT 10;
```

**Note:** `LIMIT` caps rows *returned* but does **not** cap rows *scanned* — a habit to build now for the heavier queries later (Step 15).

### Step 3 — Counting & Distribution of Event Types

```sql
SELECT event_type, count() AS c
FROM github_events
GROUP BY event_type
ORDER BY c DESC
LIMIT 25;
```

**Note:** This filters nothing, but groups by the **first** sort-key column, so ClickHouse can process it very efficiently — the data is already physically clustered by `event_type`.

### Step 4 — Filtering: Prefix vs. Non-Prefix Filters

```sql
-- (A) Filter on repo_name only — NOT the first sort-key column
SELECT event_type, count() AS c
FROM github_events
WHERE repo_name = 'ClickHouse/ClickHouse'
GROUP BY event_type
ORDER BY c DESC;

-- (B) Filter on event_type AND repo_name — uses the full sort-key prefix
SELECT count() AS c
FROM github_events
WHERE event_type = 'WatchEvent' AND repo_name = 'ClickHouse/ClickHouse';
```

**Note:** Query (B) can use the sort key far more effectively than (A), because `repo_name` is the *second* key column — filtering on it alone still requires scanning within every `event_type` range. This gets proven with `EXPLAIN` in Step 14.

### Step 5 — `IN` Lists and Multi-Value Filters

```sql
SELECT repo_name, count() AS c
FROM github_events
WHERE event_type = 'WatchEvent'
  AND repo_name IN ('ClickHouse/ClickHouse', 'duckdb/duckdb', 'apache/spark')
GROUP BY repo_name
ORDER BY c DESC;
```

**Note:** Keep the first sort-key column (`event_type`) in the filter whenever possible — it's the cheapest lever you have.

---

## Part 2 — Aggregations & Time Series *(20 min)*

### Step 6 — Date/Time Functions

```sql
SELECT toYear(created_at) AS yr, count() AS c
FROM github_events
WHERE event_type = 'WatchEvent'
GROUP BY yr
ORDER BY yr;
```

**Note:** `toYear`, `toStartOfMonth`, `toStartOfWeek`, `toDate` are the everyday time-bucketing toolkit — much like `date_trunc` in Postgres.

### Step 7 — Monthly Rollups

```sql
SELECT toStartOfMonth(created_at) AS month, count() AS prs_opened
FROM github_events
WHERE event_type = 'PullRequestEvent' AND action = 'opened'
GROUP BY month
ORDER BY month;
```

**Note:** A good candidate for a quick bar chart if your SQL client supports it. This is also the basis for an incremental materialized view in production (out of scope for a read-only Playground, but worth mentioning).

### Step 8 — Conditional Aggregation ("Pivot without PIVOT")

```sql
SELECT
    repo_name,
    sum(event_type = 'WatchEvent') AS stars,
    sum(event_type = 'ForkEvent')  AS forks
FROM github_events
WHERE event_type IN ('WatchEvent', 'ForkEvent')
  AND repo_name IN ('ClickHouse/ClickHouse', 'duckdb/duckdb', 'apache/spark')
GROUP BY repo_name
ORDER BY stars DESC;
```

**Note:** `sum(<boolean expression>)` is the classic ClickHouse pivot trick — booleans coerce to `0`/`1`. Cleaner than `CASE WHEN ... THEN 1 ELSE 0 END`, though that also works.

### Step 9 — Approximate vs. Exact Distinct Counts

```sql
SELECT
    event_type,
    uniq(actor_login)      AS approx_distinct_actors,   -- HyperLogLog, fast, ~1-2% error
    uniqExact(actor_login) AS exact_distinct_actors,     -- exact, more memory/CPU
    count()                AS events
FROM github_events
WHERE event_type IN ('WatchEvent', 'PullRequestEvent')
GROUP BY event_type
ORDER BY events DESC;
```

**Note:** At billion-row scale, `uniq()` (probabilistic) is usually the right default. Reach for `uniqExact()` only when you need exact numbers and can afford the cost — this trade-off doesn't exist in most row-store OLTP databases.

---

## ☕ Break *(5 min)*

---

## Part 3 — Intermediate Techniques *(25 min)*

### Step 10 — CTEs & Scalar Subqueries, Anchored on Data Freshness

```sql
-- Scalar CTE: compute "latest full month present in the data" instead of hardcoding a date
WITH (SELECT max(created_at) FROM github_events) AS max_ts
SELECT repo_name, count() AS stars
FROM github_events
WHERE event_type = 'WatchEvent'
  AND created_at >= toStartOfMonth(max_ts) - INTERVAL 1 MONTH
  AND created_at <  toStartOfMonth(max_ts)
GROUP BY repo_name
ORDER BY stars DESC
LIMIT 20;
```

**Note:** `WITH (scalar subquery) AS name` lets you reuse a computed value like a variable throughout the query — handy for "relative to the data's own max date," which is safer than `today()` on a dataset with a fixed ingestion cutoff.

### Step 11 — Window Functions

```sql
SELECT
    repo_name,
    week,
    weekly_stars,
    sum(weekly_stars) OVER (PARTITION BY repo_name ORDER BY week)         AS running_total,
    rank()            OVER (PARTITION BY week ORDER BY weekly_stars DESC) AS rank_in_week
FROM
(
    SELECT repo_name, toStartOfWeek(created_at) AS week, count() AS weekly_stars
    FROM github_events
    WHERE event_type = 'WatchEvent'
      AND repo_name IN ('ClickHouse/ClickHouse', 'duckdb/duckdb')
    GROUP BY repo_name, week
)
ORDER BY repo_name, week;
```

**Note:** Standard SQL:2003 window syntax (`OVER (PARTITION BY ... ORDER BY ...)`). If attendees know Postgres/Snowflake window functions, this is a 1:1 transfer.

### Step 12 — JOINs: Filter Before You Join

```sql
WITH top_repos AS
(
    SELECT repo_name
    FROM github_events
    WHERE event_type = 'WatchEvent'
    GROUP BY repo_name
    ORDER BY count() DESC
    LIMIT 20
)
SELECT s.repo_name, s.stars, f.forks
FROM
(
    SELECT repo_name, count() AS stars
    FROM github_events
    WHERE event_type = 'WatchEvent' AND repo_name IN (SELECT repo_name FROM top_repos)
    GROUP BY repo_name
) AS s
ANY LEFT JOIN
(
    SELECT repo_name, count() AS forks
    FROM github_events
    WHERE event_type = 'ForkEvent' AND repo_name IN (SELECT repo_name FROM top_repos)
    GROUP BY repo_name
) AS f
ON s.repo_name = f.repo_name
ORDER BY s.stars DESC;
```

**Notes:**
- Both sides are pre-filtered and pre-aggregated **before** the join, not after — this keeps the join's build/probe tables small.
- We use `ANY LEFT JOIN` because the relationship is 1 row per `repo_name` on each side. Plain `LEFT JOIN` would work too, but `ANY` signals intent and protects against accidental row duplication ("fan-out") if that assumption is ever violated.

### Step 13 — Arrays & `ARRAY JOIN`

```sql
SELECT label, count() AS c
FROM github_events
ARRAY JOIN labels AS label
WHERE event_type = 'IssuesEvent' AND action = 'opened' AND notEmpty(labels)
GROUP BY label
ORDER BY c DESC
LIMIT 20;
```

**Note:** `labels` is stored as `Array(LowCardinality(String))` — one issue can have many labels. `ARRAY JOIN` "unnests" the array into one row per label, ClickHouse's native answer to Postgres's `unnest()`/`LATERAL`. `LowCardinality(String)` is why filtering/grouping on `repo_name`, `actor_login`, and `labels` stays cheap even at billions of rows.

---

## Part 4 — Performance & Query-Safety Habits *(10 min)*

### Step 14 — Prove the Sort Key Matters: `EXPLAIN`

```sql
-- Filters on repo_name only (2nd key column) — expect more granules read
EXPLAIN indexes = 1
SELECT count() FROM github_events WHERE repo_name = 'ClickHouse/ClickHouse';

-- Filters on event_type + repo_name (full prefix) — expect a much smaller scan
EXPLAIN indexes = 1
SELECT count() FROM github_events WHERE event_type = 'WatchEvent' AND repo_name = 'ClickHouse/ClickHouse';
```

**Note:** Look at the `Granules` line in each plan and compare. This is the concrete payoff of the sort-key concept from Part 0 and Step 4 — it isn't abstract, it's a number you can watch shrink.

### Step 15 — Being a Good Citizen on a Shared Playground

```sql
SELECT event_type, count() AS c
FROM github_events
WHERE event_type = 'PushEvent'
GROUP BY event_type
LIMIT 100
SETTINGS
    max_execution_time = 30,
    max_rows_to_read = 1000000000,
    max_bytes_to_read = 100000000000,
    timeout_before_checking_execution_speed = 0;
```

**Note:** The Playground is a shared, free resource used by many people at once. Make bounding a habit, not an afterthought:
- Always add `LIMIT`
- Bound scans with `max_rows_to_read` / `max_bytes_to_read` (`LIMIT` alone doesn't stop a full scan)
- Set `max_execution_time`
- Never run unbounded `SELECT *` on a multi-billion-row table

---

## Wrap-Up

### Recap

- Columnar storage + MergeTree sort keys → why filter order matters
- `GROUP BY` / `ORDER BY`, conditional aggregation, `uniq()` vs `uniqExact()`
- Scalar CTEs, window functions, filter-before-join with `ANY JOIN`, `ARRAY JOIN`
- Reading `EXPLAIN indexes=1`, and query-safety habits on shared systems

### Resources

- Dataset docs: `https://clickhouse.com/docs/getting-started/example-datasets/github-events`
- ClickHouse SQL reference: `https://clickhouse.com/docs`
- Try re-running Step 1's discovery queries against your **own** ClickHouse Cloud service afterward, using the same workflow: discover → plan → execute.

**Q&A** for the remaining time.

---

## Appendix — Provenance & Caveats

- **Source:** Content authored from ClickHouse's official documentation and public GitHub issues describing the `github_events` schema and its `ORDER BY (event_type, repo_name, created_at)` sort key.
- **Not yet executed live:** These queries have not been run against live Playground data as part of preparing this lab. **Action for the presenter:** do a full dry run against the actual Playground before presenting — dataset snapshots and hostnames can change over time (this is also why Step 1 exists as a live discovery step).
- **Freshness:** Per ClickHouse docs, the canonical example dataset covers GitHub events from 2011 through Dec 6, 2020 (~3.1B rows). The live Playground copy may differ — confirm the actual `max(created_at)` in Step 1.
- **Confidence:** Medium. Schema and SQL syntax are verified against ClickHouse docs. Exact current Playground row counts, date range, and table availability are unverified until the presenter's dry run.
