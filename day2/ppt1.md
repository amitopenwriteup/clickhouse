# ClickHouse Day 2 Training — Slide-by-Slide Explained

A plain-English walkthrough of every slide in `ClickHouse_Day2_Training.pptx`, with a simple example added for each concept.

---

## Slide 1 — Title: "ClickHouse Training, Day 2 — Student Lab Guide"

**What it is:** the cover slide. It tells you this is a hands-on session (theory + labs), and that it assumes you already finished Day 1 — single-node ClickHouse setup and a look at ClickHouse Cloud.

**Why it matters:** nothing technical yet, but it sets expectations — you should already have a running ClickHouse server before this session starts.

---

## Slide 2 — Today's Agenda

**What it is:** a table of contents for the day, split into two labs:

- **Lab 1 — Operations:** user management & roles, backup/restore, version upgrades, quotas & resource management
- **Lab 2 — Data Modeling:** native data types, schema design, choosing primary keys, partitioning strategy

**Simple way to think about it:** Lab 1 is "how do I run and protect a ClickHouse cluster safely," and Lab 2 is "how do I design tables so queries stay fast." Everything after this slide is one of these eight topics.

---

## Slide 3 — Section divider: "LAB 1"

**What it is:** a section break slide, no new content — just marks the start of the "User Management, Backup/Restore, Upgrades & Quotas" block (slides 4–7).

---

## Slide 4 — Lab 1.1: User Management & Roles / Permissions

**What it says:**
- SQL-driven access control (`USER`, `ROLE`, `GRANT`) is preferred over editing XML config files directly.
- A **USER** authenticates (proves who they are). A **ROLE** bundles reusable privileges.
- `GRANT` / `REVOKE` control access at database, table, column, or even row level (via row policies).
- Best practice: don't grant privileges straight to a user — grant them to a role, then grant the role to users.

**The flow shown on the slide:**
```
ROLE  → bundles privileges
GRANT → role → user
USER  → authenticates
```

**Simple example — three people (Alice, Bob, Carol) all need read-only access to `sales_db`:**

```sql
-- 1. Create one role that bundles the privilege
CREATE ROLE analyst_ro;
GRANT SELECT ON sales_db.* TO analyst_ro;

-- 2. Create the users
CREATE USER alice IDENTIFIED WITH sha256_password BY 'pw1';
CREATE USER bob   IDENTIFIED WITH sha256_password BY 'pw2';
CREATE USER carol IDENTIFIED WITH sha256_password BY 'pw3';

-- 3. Grant the role to all three, instead of repeating GRANT SELECT three times
GRANT analyst_ro TO alice, bob, carol;
ALTER USER alice DEFAULT ROLE analyst_ro;
```

**Why this is better than granting privileges directly:** if you later need to add `sales_db.refunds` to everyone's access, you change the role *once* (`GRANT SELECT ON sales_db.refunds TO analyst_ro`) instead of updating three separate users.

---

## Slide 5 — Lab 1.2: Backup & Restore

**What it says:**
- ClickHouse has native `BACKUP` / `RESTORE` SQL commands (since v22.8) that write to local disk, S3, or another disk backend.
- Backups can be **full** or **incremental** (`BASE BACKUP`), and can target a single table, a database, or the whole server.
- Alternatives exist: filesystem snapshots via `FREEZE`, or the community `clickhouse-backup` tool.
- Golden rule: **test your restores** — an untested backup isn't a real backup.

**The flow shown on the slide:**
```
BACKUP  (table/DB → disk or another location)
   ↓ (data loss or migration happens)
RESTORE (reload from the archive)
```

**Simple example:**

```sql
-- Take a backup of one table
BACKUP TABLE sales_db.orders
  TO Disk('backups', 'orders_full.zip');

-- ...disaster happens, or you're migrating to a new server...

-- Restore it
RESTORE TABLE sales_db.orders
  FROM Disk('backups', 'orders_full.zip');
```

**Analogy:** think of `BACKUP` like `git commit` for your data, and `RESTORE` like `git checkout` — except you should actually rehearse the checkout before you need it in an emergency.

---

## Slide 6 — Lab 1.3: Version Upgrades

**What it says:**
- Always read the changelog for breaking changes before upgrading, especially for major versions.
- Upgrade staging first, then roll out to production replicas one at a time (a **rolling upgrade**).
- Replicated clusters can briefly run mixed minor versions, which is what makes zero-downtime upgrades possible.
- Always back up before a major version upgrade.

**The 8-step process on the slide, simplified into 4 real phases:**

| Phase | Steps on the slide | What you're actually doing |
|---|---|---|
| 1. Check | `SELECT version();` | Confirm your current version |
| 2. Review | Read changelog for breaking changes | Know what might break before you touch anything |
| 3. Protect | Back up critical DBs | Your rollback safety net |
| 4. Roll out | Install new binaries → restart → confirm version/logs → smoke test → repeat per node | One node at a time, never all at once |

**Simple example — the command sequence on one node:**

```bash
# 1. Check current version
clickhouse-client --query "SELECT version()"

# 2. Back up before upgrading (see Slide 5)
# BACKUP TABLE sales_db.orders TO Disk('backups','pre_upgrade.zip');

# 3. Install the new version
sudo apt-get install clickhouse-server

# 4. Restart and confirm
sudo service clickhouse-server restart
clickhouse-client --query "SELECT version()"
tail -f /var/log/clickhouse-server/clickhouse-server.log

# 5. Smoke test
clickhouse-client --query "SELECT count() FROM sales_db.orders"

# 6. Repeat on the next node in the cluster
```

**Why one node at a time:** if step 4 or 5 reveals a problem, only one replica is affected — the rest of the cluster is still serving traffic on the old (working) version.

---

## Slide 7 — Lab 1.4: Quotas & Resource Management

**What it says:**
- **QUOTAS** limit resource use (queries, errors, time, rows/bytes) per user or role, over a time interval.
- **SETTINGS** like `max_memory_usage`, `max_execution_time`, `max_threads` enforce limits on a *single query*.
- **Settings PROFILES** group settings together — the same idea as roles grouping privileges.
- **Workload/resource scheduling** gives finer CPU & I/O control for multi-tenant setups.

**The key distinction to remember:**

| | Scope | Example |
|---|---|---|
| **Settings** | One query | "This query can use at most 1 GB of RAM" |
| **Settings profile** | A named bundle of settings | "The `limited_profile` bundle = 1 GB memory + 30s timeout" |
| **Quota** | Usage over time | "This user can run at most 1000 queries per day" |

**Simple example — cap what user `bob` can do:**

```sql
-- A reusable bundle of per-query limits
CREATE SETTINGS PROFILE limited_profile
  SETTINGS max_memory_usage = 1000000000,   -- 1 GB per query
           max_execution_time = 30;          -- 30 seconds per query

ALTER USER bob SETTINGS PROFILE limited_profile;

-- A cap on usage over a whole day
CREATE QUOTA daily_quota
  FOR INTERVAL 1 DAY
  MAX QUERIES = 1000, MAX ERRORS = 50, MAX EXECUTION TIME = 3600
  TO bob;
```

**Analogy:** the settings profile is like a per-meal portion limit; the quota is like a daily calorie budget. Both apply to `bob`, but they cap different things.

---

## Slide 8 — Section divider: "LAB 2"

**What it is:** marks the start of the "Data Types, Schema Design, Primary Keys & Partitioning" block (slides 9–12) — the data-modeling half of the day.

---

## Slide 9 — Lab 2.1: Native Data Types

**What it says:**
- ClickHouse natively supports numeric (`UInt`/`Int`/`Float`/`Decimal`), string (`String`, `FixedString`), date/time (`Date`, `DateTime64`), and semi-structured (`Array`, `Tuple`, `Map`, `JSON`, `Nested`) types.
- `LowCardinality(T)` dictionary-encodes repeated values (like `country` or `status`) for big speed/storage gains.
- `Nullable(T)` allows NULLs but adds overhead (a separate bitmap tracking which rows are null) — avoid it on high-cardinality columns.
- Rule of thumb: **choose the smallest type that safely fits your data** — since ClickHouse is columnar, smaller types directly reduce scan cost (see the "columnar storage" explanation from earlier — same idea).

**Table from the slide:**

| Category | Examples |
|---|---|
| Numeric | `UInt32`, `Int16`, `Float64`, `Decimal(10,2)` |
| String | `String`, `FixedString`, `LowCardinality(String)` |
| Date / Time | `Date`, `DateTime`, `DateTime64` |
| Semi-structured | `Array`, `Tuple`, `Map`, `JSON`, `Nested` |

**Simple example:**

```sql
CREATE TABLE events (
    event_time   DateTime,
    country      LowCardinality(String),  -- only ~200 possible values, encode as dictionary
    status_code  UInt16,                  -- fits 0-65535, no need for UInt64
    tags         Array(String),
    user_id      UInt32
) ENGINE = MergeTree
ORDER BY event_time;
```

**Why `LowCardinality(String)` matters here:** `country` only ever holds a couple hundred distinct values across billions of rows. Storing it as plain `String` repeats the same text over and over; `LowCardinality` stores each unique value once and references it by a small integer — much less disk, much faster filtering.

---

## Slide 10 — Lab 2.2: Schema Design Best Practices

**What it says:**
- Design tables around your query patterns — ClickHouse favors **wide, denormalized** tables over lots of joins (unlike OLTP databases, which favor normalization).
- Pick the right table engine for the job.
- Avoid excessive `Nullable` columns and high-cardinality key columns; prefer `LowCardinality` + `Enum`.
- **Materialized views** pre-compute aggregates at insert time, speeding up read-heavy dashboards.

**Engine table from the slide:**

| Engine | Use it for |
|---|---|
| `MergeTree` | General-purpose analytical tables |
| `ReplacingMergeTree` | Keeping only the latest row per key (de-duplication) |
| `Summing` / `AggregatingMergeTree` | Pre-aggregated rollups, e.g. daily sales totals |

**Simple example — why "wide and denormalized" beats joins here:**

```sql
-- OLTP instinct: normalize into separate tables and JOIN at query time
-- orders(order_id, customer_id, product_id, amount)
-- customers(customer_id, name, country)
-- products(product_id, name, category)

-- ClickHouse-friendly: one wide table, joined once at write time, not at every query
CREATE TABLE orders_wide (
    order_id     UInt64,
    customer_name String,
    customer_country LowCardinality(String),
    product_name String,
    product_category LowCardinality(String),
    amount       Decimal(10,2),
    order_time   DateTime
) ENGINE = MergeTree
ORDER BY order_time;
```

**Why:** joining billions of rows at query time is expensive. Doing the join once, when the data is written (or via a materialized view), means every future `SELECT` just scans one flat table.

---

## Slide 11 — Lab 2.3: Choosing Primary Keys

**What it says:**
- The `PRIMARY KEY` (usually the same as `ORDER BY`) is a **sparse index used to skip granules** — it is *not* a uniqueness constraint like a primary key in MySQL/Postgres.
- Order columns from **lowest to highest cardinality**, putting the most common filter/range columns first (e.g. date, then `customer_id`).
- Avoid putting a high-cardinality unique ID (like a UUID) as the *first* key column — it blocks granule skipping.

**The comparison shown on the slide:**

| | Key | What happens |
|---|---|---|
| **Poorly chosen** | `ORDER BY event_id` | High-cardinality UUID first — no effective pruning; almost every granule could contain the filtered value |
| **Well-chosen** | `ORDER BY (event_type, event_time, user_id)` | Low-cardinality, frequently-filtered columns lead — efficient pruning |

**Simple example, connecting back to the "MergeTree primary key index" concept from earlier:**

```sql
-- Poorly chosen: every query filtering on event_type has to scan almost everything
CREATE TABLE events_bad (
    event_id UUID,
    event_type LowCardinality(String),
    event_time DateTime
) ENGINE = MergeTree
ORDER BY event_id;

-- Well-chosen: a query filtering "WHERE event_type = 'click'" can skip most of the table
CREATE TABLE events_good (
    event_id UUID,
    event_type LowCardinality(String),
    event_time DateTime
) ENGINE = MergeTree
ORDER BY (event_type, event_time);
```

**Analogy:** this is the same idea as a phone book — sorted by last name (low-ish cardinality, commonly searched), it lets you skip straight to "Smith." Sorted by phone number (effectively unique, rarely searched by), the sorting buys you nothing for a name lookup.

---

## Slide 12 — Lab 2.4: Partitioning Strategy

**What it says:**
- `PARTITION BY` splits table data into separate physical parts (commonly by month or day) — this enables fast pruning *and* easy bulk drop/archive of old data.
- **Partitioning is not the same as `ORDER BY`:** use partitioning for lifecycle management (TTL, dropping old data); use `ORDER BY` for filtering *within* a partition.
- Too many small partitions (e.g. partitioning by exact timestamp, or by a high-cardinality column) hurts performance — a common mistake.
- Rule of thumb: partition by **month** for most time-series workloads, by **day** only for very high-ingest use cases.

**Simple example:**

```sql
CREATE TABLE logs_partitioned (
    log_time DateTime,
    level    LowCardinality(String),
    message  String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(log_time)   -- one partition per month
ORDER BY (level, log_time);        -- sorting/index WITHIN each partition

-- See what partitions actually exist
SELECT partition, name, rows
FROM system.parts
WHERE table = 'logs_partitioned';

-- Instantly drop an entire month's worth of data
ALTER TABLE logs_partitioned DROP PARTITION 202405;

-- Or let old data expire automatically after 90 days
ALTER TABLE logs_partitioned MODIFY TTL log_time + INTERVAL 90 DAY;
```

**Why "partition by month, not by exact timestamp" matters:** if you partitioned by the exact second, you'd end up with millions of tiny partitions — the opposite of the "few large sorted parts" goal from the MergeTree explanation earlier. Partitioning and `ORDER BY` solve two different problems: partitioning answers "which chunk of data can I delete/skip entirely," `ORDER BY` answers "where inside this chunk is the row I want."

---

## Slide 13 — Recap & What's Next

**What it is:** a one-line summary of each of the 8 topics covered, split by lab:

**Lab 1 recap:**
- Roles bundle privileges → grant to users.
- `BACKUP` / `RESTORE` protects your data.
- Upgrade in staging first, then roll out.
- Quotas & settings profiles cap resource use.

**Lab 2 recap:**
- Pick the smallest fitting data type.
- Design wide tables around query patterns.
- Order keys by cardinality, filters first.
- Partition for lifecycle, not for filtering.

**What's next:** Day 3 will cover query optimization, a deep-dive on materialized views, and monitoring.

---

## One-page cheat sheet

| Slide | Topic | One-line takeaway |
|---|---|---|
| 4 | Users & roles | Grant privileges to roles, not directly to users |
| 5 | Backup & restore | Test your restores, not just your backups |
| 6 | Version upgrades | Staging first, then roll out node by node |
| 7 | Quotas & settings | Settings cap one query; quotas cap usage over time |
| 9 | Data types | Use the smallest type that fits; `LowCardinality` for repeated strings |
| 10 | Schema design | Wide, denormalized tables beat joins at query time |
| 11 | Primary keys | Sort low-cardinality, frequently-filtered columns first — never lead with a UUID |
| 12 | Partitioning | Partition for lifecycle (drop/TTL), not for query filtering |
