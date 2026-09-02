# ClickHouse Day 2 Lab Guide — Explained Section by Section

This walks through *why* each section of the lab guide exists and what each step is actually accomplishing — not just repeating the commands, but explaining the reasoning behind the order and the gotchas called out.

---

## Intro block

**What it establishes:** this is a hands-on continuation of Day 1 (single node + ClickHouse Cloud). The format promise — "theory, then hands-on, each step gets a 1–2 line explanation" — matters because it tells you every command in the guide is meant to be typed and observed, not just read.

**Two labs, two different concerns:**
- **Lab 1 = operations** — keeping a running cluster safe, upgradeable, and fair to share
- **Lab 2 = data modeling** — designing tables so queries stay fast as data grows

---

## LAB 1: User Management, Backup/Restore, Version Upgrades, Quotas

### 1.1 — User Management & Roles/Permissions

**The theory in one sentence:** don't grant privileges to people directly — grant privileges to a *role*, then hand the role to people, so permission changes happen in one place instead of N places.

**What the 9 hands-on steps are really doing, grouped:**

| Steps | Purpose |
|---|---|
| 1 | Connect as admin — you need elevated rights just to create the users/roles that follow |
| 2–3 | Build the reusable unit first: create `analyst_ro`, then attach `SELECT` to it — the role exists *before* any user does |
| 4–5 | Create two people with two different job functions (`alice` = read-only, `bob` = read/write) and attach the matching role to each — this is the "least privilege" principle in action: bob's role is separate from alice's, not a superset |
| 6 | Set a *default* role — without this, `alice` would have to manually run `SET ROLE analyst_ro` every session before her grants apply |
| 7 | Prove it worked — reconnect *as* alice and check `SHOW GRANTS`, rather than trusting the admin's view of what should be true |
| 8 | An optional deeper layer: a row policy filters *which rows* alice can see even within a table she's allowed to `SELECT` from — privilege and visibility are two separate controls |
| 9 | Cleanup — revoke, then drop. This order matters: you're removing the *grant* before removing the *user*, undoing things in the reverse order you built them |

**The one-sentence takeaway:** every step here builds toward proving, not just assuming, that access control does what you intended — step 7 (verify as alice) is the step most people skip and shouldn't.

---

### 1.2 — Backup & Restore

**The theory in one sentence:** ClickHouse has native `BACKUP`/`RESTORE` SQL commands that write to a named "disk" — but that disk has to be explicitly registered and allowlisted in config before any backup command will work.

**Why the hands-on has 12 steps instead of just "run BACKUP":** most of this section (steps 1–6) is actually infrastructure setup, and only steps 7 onward touch actual backup/restore SQL. That split is the real lesson:

| Steps | What's happening | Why it's not optional |
|---|---|---|
| 1 | Create the physical folder, owned by the `clickhouse` OS user | The server process needs write permission on this exact path, or every backup attempt fails silently at the OS level |
| 2–3 | Add a config file defining a disk named `backups`, **and** allowlist that disk name | This is the step people miss: even with the disk correctly defined, ClickHouse refuses `BACKUP ... TO Disk('backups', ...)` unless `<allowed_disk>` also names it explicitly — defining a disk and *permitting* its use are two separate config blocks |
| 4 | Sanity-check the XML *before* restarting | Malformed config can stop the server from starting at all — catching that before a restart avoids a full outage |
| 5 | Query `system.disks` to confirm ClickHouse actually picked up the config | Closes the loop — don't assume a restart worked, check `SELECT uptime()` and the disk list |
| 6 | A throwaway test backup | Cheap way to confirm the whole chain (folder → permissions → config → allowlist → restart) is wired correctly, before doing anything that matters |
| 7–8 | Real backups — one table, then a whole database | Shows both scopes are available: targeted or comprehensive |
| 9 | Deliberately destroy data (`TRUNCATE`) | You can't actually prove a restore works unless there's something real to recover from |
| 10–11 | Restore, then verify row counts | Restoring isn't "done" until you've confirmed the data is *actually* back — matching row counts is the simplest proof |
| 12 | Incremental backup, layered on the full one | Only stores what changed since the base backup — the practical reason incremental backups exist is speed/space, once you already have a full baseline |

**The callout worth remembering:** on ClickHouse Cloud, none of steps 1–5 apply — backups are platform-managed. This whole disk-registration dance is specific to self-managed/on-prem clusters, and production setups usually point the disk at S3 instead of local disk, so a backup survives losing the node itself.

---

### 1.3 — Version Upgrades

**The theory in one sentence:** upgrade staging first, then production one node at a time — never all nodes simultaneously — because replicated clusters can tolerate a brief mix of versions, which is exactly what makes a *rolling* upgrade possible.

**The 10 steps map to 4 real phases:**

| Phase | Steps | Purpose |
|---|---|---|
| Baseline | 1–2 | Know your current version and read the changelog *before* touching anything — this is where you catch breaking changes in advance instead of during an incident |
| Safety net | 3 | Back up first (linking back to 1.2) — the rollback plan has to exist before you need it |
| Install | 4–6 | Add the repo (first time only), pull the new packages via `dnf`, restart via `systemctl` | This section is explicitly Rocky Linux/RHEL-flavored — `dnf`/`yum` and `systemctl` replace Debian's `apt-get`/`service` |
| Verify | 7–9 | Confirm the service is active *and* enabled (survives reboot), confirm the version actually changed, check logs, then run a real query as a smoke test |
| Scale out | 10 | Repeat the whole install→verify cycle one node at a time on a cluster | This is the actual rolling-upgrade behavior — each node is upgraded and confirmed healthy before moving to the next |

**The Rocky-Linux-specific gotcha at the end:** SELinux enforcing mode is far more likely to interfere with an RPM-based install than it would on Debian/Ubuntu — the guide tells you to check `sestatus` and the audit log specifically *after* the upgrade, because a newly-installed binary can trip a policy an older one didn't.

---

### 1.4 — Quotas & Resource Management

**The theory in one sentence:** two different controls exist for two different problems — **settings/profiles** cap what a *single query* can do, while **quotas** cap *total usage over a time window*. Confusing the two is the most common mistake here.

**Walking through the 7 steps:**

| Step | What it does | Which control it demonstrates |
|---|---|---|
| 1 | Create `bob` (if he doesn't already exist from 1.1) | Setup — a profile/quota needs a user to attach to |
| 2 | Create `limited_profile` with `max_memory_usage` and `max_execution_time`, both marked `READONLY` | A **settings profile** — note `READONLY` here means bob himself can't override these settings, not that his queries are read-only |
| 3 | Attach the profile to bob | Now every one of bob's queries is capped, automatically, with no per-query effort |
| 4 | Create `daily_quota`: max 1000 queries, 50 errors, 1 hour total compute, per rolling 24h | A **quota** — this caps bob's *aggregate* behavior over a day, separate from any single query's limits |
| 5 | Connect as bob, run `SELECT sleep(35)` | This is a live test: 35 seconds exceeds the 30-second `max_execution_time` from the profile, so it should be killed — proving the profile actually enforces, not just declares |
| 6 | `SHOW QUOTA` | Lets bob (or you, as him) see exactly how much of his daily budget is used |
| 7 | Reconnect as admin, inspect `system.query_log` sorted by duration | Zooms out from "is bob capped" to "what's expensive across the whole cluster" — this is the operational habit the whole quota system exists to support |

**The one distinction to hold onto:** step 2's limits apply *per query*; step 4's limits apply *per day*. A single fast query can still count toward the daily quota even though it never comes close to the per-query memory/time caps.

---

## LAB 2: Data Types, Schema Design, Primary Keys, Partitioning

### 2.1 — Native Data Types

**The theory in one sentence:** because ClickHouse is columnar, the *size* of a column's type directly affects both storage and scan speed — so picking the smallest type that safely fits your data isn't a micro-optimization, it's a core performance lever.

**What the 5 hands-on steps demonstrate, in order:**

1. **Build one table using five type categories at once** (`UInt32`, `LowCardinality(String)`, `Decimal`, `Array`, `Nullable`) — this isn't just a syntax tour, it's meant to be a single reference schema you can compare against later.
2. **Insert one row** — just enough data to inspect, not a performance test yet.
3. **Query `system.columns` for compressed vs. uncompressed byte size** — this is the step that turns "LowCardinality helps" from a claim into a number you can actually see per column.
4. **Compare `String` vs. `LowCardinality(String)` on a larger dataset** — the earlier steps only prove the feature exists; this step is where the *speed* difference on a low-distinct-value column (like `country`) actually shows up.
5. **Query `WHERE is_active IS NULL`** — confirms `Nullable` works correctly, but the explanation is really a warning: nullability isn't free, it's a hidden second column tracking which rows are null.

**Why this section leads with a mixed schema instead of one type at a time:** in a real table you're never choosing just one type — you're making five or six of these decisions at once, and step 3's byte-size query is the tool you'd actually use afterward to check whether your choices paid off.

---

### 2.2 — Schema Design Best Practices

**The theory in one sentence:** bring OLTP instincts here and you'll normalize into many small joined tables — the opposite of what performs well; ClickHouse rewards wide, denormalized tables and engines chosen for their *merge-time behavior*, not just their name.

**The 4 hands-on steps build on each other:**

1. **`orders_flat`** — a single wide table standing in for what would be three normalized tables (orders/customers/products) in an OLTP design. The join happens once, when you decide the schema — not on every query.
2. **`customers_latest`, using `ReplacingMergeTree`** — introduces the idea that engine choice is part of schema design, not separate from it. This table auto-resolves duplicate/updated customer rows during background merges, so you don't need application-level "upsert" logic.
3. **`daily_sales_mv`, a materialized view over `orders_flat`** — this is where pre-aggregation enters: instead of summing `orders_flat` on every dashboard load, the sum happens once, incrementally, as rows are inserted.
4. **Run the same logical query against the raw table and the materialized view** — this is the payoff step. It's meant to make the speed difference visible and concrete, not just asserted in the theory bullet above it.

**The thread connecting all four steps:** every one of them is really the same decision — do the expensive work *once, up front* (flattening the joins, deduplicating, aggregating), so every future query is reading something already close to the answer.

---

### 2.3 — Choosing Primary Keys

**The theory in one sentence:** the primary key in MergeTree is a sparse index for *skipping data*, not a uniqueness constraint — so its usefulness depends entirely on column order, with low-cardinality, frequently-filtered columns needing to come first.

**Why the hands-on builds a deliberately bad table first:**

| Step | Purpose |
|---|---|
| 1 | `events_bad`, `ORDER BY event_id` (a UUID) | A high-cardinality, effectively-unique column leads the key — since almost every value is unique, sorting by it gives the query engine almost nothing to skip |
| 2 | `events_good`, `ORDER BY (event_type, event_time, user_id)` | Same data, key reordered so the low-cardinality, commonly-filtered column (`event_type`) leads |
| 3 | Load identical data into both | Keeps the comparison fair — this isn't a comparison of *data*, it's a comparison of *key order* on the same data |
| 4 | `EXPLAIN indexes = 1` on the same filtered query, against both tables | This is the step that turns "primary key order matters" from a claim into something you can literally see — the explain output shows how many granules got skipped |
| 5 | Check `read_rows` and `query_duration_ms` in `system.query_log` | The final proof point — translates "granules skipped" into an actual, measurable time difference |

**Why this section exists right after 2.2:** schema design (2.2) decides *what* columns exist and which engine to use; primary key choice (2.3) decides *how those columns are physically ordered on disk* — two separate decisions that both determine query speed.

---

### 2.4 — Partitioning Strategy

**The theory in one sentence:** `PARTITION BY` and `ORDER BY` solve two different problems — partitioning is about *physically* grouping data for fast bulk delete/archive, while `ORDER BY` is about *sorting within* whatever partition a query lands in.

**Walking through the 6 hands-on steps:**

1. **Create `logs_partitioned`, `PARTITION BY toYYYYMM(log_time)`** — monthly partitioning, deliberately coarse rather than fine-grained.
2. **Insert rows spanning two different months** — the only way to actually get more than one partition to inspect in the next step.
3. **Query `system.parts`** — makes the abstract idea of "physical partitions" concrete: you can literally see separate partition rows with their own row counts.
4. **`EXPLAIN indexes = 1` with a recent-time filter** — proves partition pruning is actually happening: the query should only touch the relevant month's partition, not scan everything.
5. **`ALTER TABLE ... DROP PARTITION 202405`** — the entire point of partitioning shows up here: deleting a month of data is a near-instant metadata operation, not a slow row-by-row `DELETE`.
6. **`MODIFY TTL log_time + INTERVAL 90 DAY`** — automates step 5 going forward, so nobody has to remember to manually drop old partitions on a schedule.

**Why this is the last topic in Lab 2:** it assumes everything before it — the table already has sensible types (2.1), a sensible engine and shape (2.2), and a sensible `ORDER BY` (2.3). Partitioning is the final, physical-layout decision layered on top of all three.

---

## How the two labs connect

| | Lab 1 (Operations) | Lab 2 (Data modeling) |
|---|---|---|
| Question it answers | Who can do what, and how do we keep the cluster safe/running? | How do we structure tables so queries stay fast? |
| Where mistakes show up | Security incidents, failed upgrades, one noisy user starving everyone else | Slow queries, runaway storage growth, expensive scans |
| Common thread | Explicit, reusable structures (roles, profiles, quotas) instead of one-off fixes per user | Explicit, reusable structures (engines, keys, partitions) instead of one-off fixes per query |

**The guide's closing note** — Day 3 covers query optimization, materialized views in more depth, and monitoring — is really Lab 2's natural continuation: today you learn to *design* fast tables; Day 3 is about *diagnosing* and *watching* them once they're live.
