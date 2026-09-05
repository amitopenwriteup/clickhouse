# ClickHouse — Trainer Guide
### Companion to `ClickHouse_Understanding.pptx` (13 slides)

**How to use this guide:** Each section below maps to one slide. It gives you a spoken walkthrough, the key points to land, a plain-language example, and a question you can throw at the room before revealing the answer. Suggested pace assumes a ~45–60 minute session; trim the "go deeper" boxes if you're running short.

---

## Slide 1 — Title: "ClickHouse: An Operator's & Data Modeler's Guide"

**Say this:**
"Today we're covering ClickHouse from two angles: running it day-to-day as an operator, and modeling data correctly so it stays fast. We'll split roughly in half — operations first (users, backups, upgrades, quotas), then data modeling (types, schema, keys, partitioning)."

**Key point:** Frame the split up front so the audience knows the deck has two halves — this makes slide 2 (agenda) land better.

**Ask the room:** "Who here is already running ClickHouse in production vs. evaluating it?" — adjust pacing based on the answer.

---

## Slide 2 — Agenda: "What We'll Cover"

**Say this:**
"Eight topics, split into two groups of four. Left column is operations — the stuff that keeps a cluster alive. Right column is data modeling — the stuff that determines whether your queries are fast or slow. We'll go in this order."

**Key point:** Don't over-explain each item here — just orient. Save the depth for its own slide.

**Transition line:** "Let's start with who can do what — user management and roles."

---

## Slide 3 — User Management & Roles

**Say this:**
"ClickHouse uses SQL-driven access control — the same GRANT/REVOKE model you'd expect from Postgres or MySQL, not a static config file you have to redeploy."

**Walk the diagram left to right:**
- **USER** — a login identity. Can authenticate via password, SSL certificate, LDAP, or Kerberos.
- **ROLE** — a named, reusable bundle of privileges. This is the layer you actually design around.
- **PRIVILEGE** — the actual permission (SELECT, INSERT, ALTER, CREATE...), scoped as narrowly as a single column.

**Key point to emphasize:** "Never grant privileges directly to a person. Grant them to a role, then assign the role to the person. When someone changes teams, you swap their role — you don't hunt down twelve individual grants."

**Plain example:**
"Think of it like a hotel keycard system. The card (user) doesn't have permissions baked into the plastic — it's linked to an access profile (role) that says 'housekeeping' or 'management.' Change the profile, and every card linked to it updates instantly."

**Go deeper (if time allows):**
- `access_management = 1` is the setting that moves user/role storage from `users.xml` into SQL-queryable system tables — mention this is usually the first thing to enable in a new cluster.
- ROW POLICY vs. SETTINGS PROFILE: a row policy filters *which rows* a role can see; a settings profile caps *how much* a query can do (memory, time, threads).

**Ask the room:** "If a role is granted SELECT on `sales.*`, and a user has that role plus their own direct GRANT of INSERT on one table — what can they do?" (Answer: both — privileges are additive/union, not overridden.)

---

## Slide 4 — Roles & Permissions in Practice

**Say this:**
"This is what slide 3 looks like as actual SQL. Left side, top to bottom: create a role, grant it a privilege, create a user with a hashed password, attach the role to the user, then revoke one specific privilege from the role."

**Walk the code block:**
```sql
CREATE ROLE analyst;
GRANT SELECT ON sales.* TO analyst;

CREATE USER maria IDENTIFIED WITH sha256_password BY 'strong_password';
GRANT analyst TO maria;

REVOKE INSERT ON sales.orders FROM analyst;
```
"Notice the REVOKE at the bottom — that's mostly there to show the syntax is symmetric. If `analyst` never had INSERT to begin with, this is a no-op, but it demonstrates you can peel back a single privilege without touching the whole role."

**Walk the scope table (right side) from broadest to narrowest:**
Global → Database → Table → Column → Row Policy.

**Key point:** "Notice each row is just a more specific version of `GRANT ... ON ...`. The syntax barely changes — only the scope does. Row-level security is the odd one out — that uses `CREATE ROW POLICY`, a separate statement, not a GRANT scope."

**Closing line for this slide:** "Remember: privileges compound. A user's effective access is the union of everything granted to them directly, plus everything granted through every role they hold."

---

## Slide 5 — Backup & Restore

**Say this:**
"Three ways to back up a ClickHouse cluster, in order of how most teams should prefer them."

**Walk each card:**
1. **Native BACKUP / RESTORE** — Built into ClickHouse since version 22.8. SQL commands, can write to local disk, S3, or GCS. Supports incremental backups on top of a full base backup. This is the modern default.
2. **clickhouse-backup** — A popular open-source community tool. Wraps the native commands with scheduling, retention policies, and cloud upload — good if you want backup automation without writing your own cron/orchestration layer.
3. **Freeze + rsync/clone** — The old-school, low-level method. `ALTER TABLE ... FREEZE` creates hard-linked snapshots on disk that you then copy manually. Still useful for pure filesystem-level snapshots, but it's manual and easy to get wrong.

**Plain example:**
"Native BACKUP/RESTORE is like using your phone's built-in cloud backup. clickhouse-backup is like a third-party app that adds scheduling on top. FREEZE is like manually copying files to a USB drive — it works, but nothing schedules it or tells you if it failed."

**Key point:** "Restore is just the mirror image — `RESTORE TABLE ... FROM ...`. But the most-skipped step on real teams is testing the restore, not just running the backup. A backup you've never restored from is a hypothesis, not a plan."

---

## Slide 6 — Backup & Restore: Best Practices

**Say this:**
"This is the checklist version of the last slide — six things to actually put into your runbook."

**Walk the table, pausing on the two most commonly missed:**
- **"Test restores regularly, not just backups"** — repeat this, people nod along but rarely schedule it.
- **"Back up ZooKeeper / Keeper metadata alongside data"** — call this out specifically: replicated table metadata lives in ZooKeeper/Keeper, not on the data disk. If you only back up the data directory, you can restore data but lose the replication coordination state.

**Quick pass through the rest:**
- Incremental-on-full reduces both backup time and storage cost.
- Off-cluster storage protects against a whole-node or whole-disk failure — backing up to the same disk you're protecting defeats the purpose.
- Version-matching backup and restore clusters avoids surprises when a newer table feature doesn't exist on an older restore target.
- A retention scheme like 7 daily / 4 weekly / 12 monthly keeps costs predictable.

**Ask the room:** "Show of hands — who has actually run a full restore drill in the last 6 months?" (This usually gets an honest, slightly uncomfortable answer — good moment for discussion.)

---

## Slide 7 — Version Upgrades

**Say this:**
"Upgrades are where a lot of self-inflicted outages happen. The rule here is simple: never upgrade a whole cluster at once."

**Walk the 5-step rolling path on the left:**
1. Read the release notes — specifically for breaking changes and deprecated settings.
2. Upgrade **one replica** first.
3. Run real, representative queries against it — compare latency and query plans against the old version.
4. Only then roll to the rest, one node at a time.
5. Upgrade the coordination layer (ZooKeeper/Keeper) separately if it needs it — don't bundle it with a data-node upgrade.

**Walk the guardrails panel on the right:**
- Prefer small, incremental version hops over jumping multiple major versions at once.
- Keep at least one replica on the old version until you've validated the new one — this is your instant rollback path.
- Check client/driver protocol compatibility before assuming everything downstream still works.
- Snapshot config and take a backup *immediately before* upgrading, not last week.
- Watch `system.errors` and the replication queue actively during and after each node's upgrade.

**Plain example:**
"This is exactly why phone OS updates roll out gradually to a percentage of devices first — you want to catch a bad build on 1% of your fleet, not 100%."

**Key point:** "The replica you deliberately leave un-upgraded isn't laziness — it's your safety net."

---

## Slide 8 — Quotas & Resource Management

**Say this:**
"Two different knobs here, and people often conflate them. Quotas control what a *user* can consume over *time*. Settings profiles control what a *single query* can do *right now*."

**Walk the Quotas panel (left):**
- `queries` — a hard cap on how many queries a user can run in the interval.
- `errors` — caps on failed queries (useful for catching a broken client hammering the cluster).
- `result_rows` / `read_rows` — caps how much data can be returned or scanned.
- `execution_time` — a cumulative time budget across all their queries in the window.

**Walk the Settings Profiles panel (right):**
- `max_memory_usage` — the memory ceiling for one query.
- `max_threads` — parallelism cap per query.
- `max_execution_time` — a per-query timeout.
- `max_concurrent_queries_for_user` — how many queries that user can have running at once.

**Plain example:**
"A quota is like a monthly data cap on a phone plan — it's about total usage over time. A settings profile is like a per-call time limit — it doesn't care about your monthly total, just this one call, right now."

**Key point:** "Newer ClickHouse versions add workload scheduling — you can carve CPU and I/O into named 'workloads,' so a heavy overnight batch job can't starve an interactive dashboard query competing for the same cluster."

---

## Slide 9 — Native Data Types

**Say this:**
"This is the foundation for everything in the second half of the deck. Get types right, and schema design, primary keys, and partitioning all get easier."

**Walk the table by category:**
- **Numeric** — Int8 through Int256, unsigned variants, floats, and Decimal for exact fixed-precision math (money, for example).
- **String** — `String` for variable length; `FixedString(N)` only when every value truly is the same length (country codes, hashes).
- **Date/Time** — `Date`/`Date32` for day-level precision, `DateTime`/`DateTime64` when you need time-of-day, with `DateTime64` adding sub-second precision.
- **Complex** — `Array`, `Tuple`, `Map`, and `Nested` let you model structured or repeated data without a join.
- **Special** — `LowCardinality`, `Nullable`, `UUID`, `Enum8/16`, IP address types. Each one is a deliberate optimization, not a default.

**Key point — the rule of thumb to repeat:** "The narrower and more specific the type, the better the compression and the faster the query. Don't reach for `Int64` or `String` out of habit — reach for the smallest type that actually fits your data's range."

**Ask the room:** "If you're storing a country code that's always exactly 2 characters, `String` or `FixedString(2)`?" (Answer: FixedString(2) — fixed length, more efficient.)

---

## Slide 10 — Schema Design Best Practices

**Say this:**
"Six practices, and they all point the same direction: design for how ClickHouse actually reads and compresses data, not for how you'd normalize a transactional database."

**Walk each card:**
1. **Avoid Nullable where possible** — Nullable adds a hidden bitmap tracking which rows are null and disables some optimizations. If the domain allows it, use a sentinel default (like `0` or `''`) instead.
2. **Use LowCardinality for repeated strings** — status, country, category fields with a small set of distinct values. This can dramatically cut storage and speed up grouping/filtering.
3. **Denormalize deliberately** — ClickHouse is built for wide, flat tables. Joins across large tables are comparatively expensive — pull related data into the same row where it makes sense.
4. **Model repeated structures with Array/Nested** — instead of a separate "order line items" table joined back to orders, put the line items inline as an Array.
5. **Order columns by access pattern, not convenience** — this is a direct preview of the primary key discussion coming up next.
6. **Separate hot and cold data** — use TTL rules and storage tiers so old, rarely-queried data moves to cheaper storage automatically.

**Plain example:**
"This is the opposite instinct from a normalized SQL database. In Postgres you'd split things into five joined tables to avoid repetition. In ClickHouse, you often want one wide table — repetition is cheap when it's this well compressed, and joins are the expensive part."

**Key point:** "None of these are absolute rules — they're trade-offs. Nullable exists for a reason; just don't reach for it by default."

---

## Slide 11 — Choosing Primary Keys

**Say this:**
"This is probably the single most important slide in the deck for query performance. In a MergeTree table, the primary key is *also* the sorting key — and that's a very different concept from a primary key in a traditional database."

**Key point — say this clearly and slowly:**
"It is NOT a uniqueness constraint. ClickHouse will happily let you insert duplicate primary key values. What it actually does is define the physical order rows are stored in, and build a sparse index on top of that order so ClickHouse can skip huge chunks of data during a scan."

**Walk the ordering diagram:**
`toDate(timestamp) → customer_id → event_type → session_id`
"Order matters, and the rule is: lowest cardinality and most frequently filtered column first, highest cardinality and rarely-filtered-alone column last. Here, date has huge, predictable value overlap and is almost always in a WHERE clause — so it goes first. `session_id` is nearly unique per row and rarely queried alone — so it goes last."

**Plain example:**
"Think of a library sorted by genre, then author, then title. If you're looking for 'mystery novels,' that sort order lets you skip every other genre instantly. If the library were sorted by book title first, 'find me all mystery novels' would mean scanning the whole building."

**Walk the guidelines:**
- Lead with whatever's most common in your WHERE clauses.
- Keep it short — 4–5 columns is typical; each additional column adds index overhead.
- ORDER BY and PARTITION BY solve different problems — don't assume they're interchangeable (this sets up slide 12 perfectly).
- Changing the sorting key later means rewriting the whole table — this is a design decision you want to get right up front, not patch later.

**Ask the room:** "If 90% of your queries filter by `customer_id` but only 10% filter by date, does date still go first?" (Answer: no — lead with what you actually filter on most; the example order isn't universal.)

---

## Slide 12 — Partitioning Strategy

**Say this:**
"This slide exists mostly to correct a common misconception: people assume partitioning is what makes ClickHouse fast. It's not — that's the primary key's job. Partitioning is about data *lifecycle*, not query speed."

**Walk the left panel — what it's actually for:**
- Dropping old data instantly with `ALTER TABLE ... DROP PARTITION` — near-instant, versus a slow DELETE.
- Moving whole partitions between storage tiers — e.g., last month's data moves from SSD to cheaper object storage.
- Scoping backups to a time range.
- The common pattern: `PARTITION BY toYYYYMM(event_date)` — monthly buckets.

**Walk the right panel — the pitfalls, and spend the most time here:**
- **Partitioning too finely** (by day, or worse, by customer) creates thousands of tiny physical parts — this slows down background merges and can genuinely hurt performance rather than help it.
- **High-cardinality partition keys** are the same mistake in different clothing — same problem, different cause.
- **Relying on partitioning for query speed** instead of the ORDER BY key — restate the core message.
- **Forgetting that each INSERT typically creates one part per partition it touches** — if a single batch insert spans 12 months of data, you just created 12 small parts instead of one.

**Plain example:**
"Partitioning is like organizing filing cabinets by year, so you can wheel out and shred all of 2019's folders at once. It's not what makes it fast to find one specific document inside 2023's folder — that's the internal sorting inside the drawer, which is your primary key."

**Key point to land before moving on:** "If someone asks 'how do I make ClickHouse faster,' the answer is almost always primary key design, not partitioning."

---

## Slide 13 — Key Takeaways / Summary

**Say this:**
"Let's close the loop — one line per topic, matching the agenda from the start."

**Walk both columns, slower than you'd think necessary — this is the slide people screenshot:**

*Operations:*
- Manage access with SQL-driven roles — group privileges, don't grant ad hoc.
- Automate backups with tested, off-cluster restores.
- Upgrade one replica at a time; never skip validation.
- Use quotas and settings profiles to keep shared clusters stable.

*Data Modeling:*
- Choose the narrowest native type that fits each column.
- Design schemas wide and denormalized, not heavily joined.
- Order the primary key from low to high cardinality.
- Partition for lifecycle management, not for query speed.

**Closing line:** "If you remember nothing else from today: the primary key controls speed, partitioning controls lifecycle, and roles should always sit between your users and your privileges — never grant directly."

**Open the floor:** "Questions on anything — operations side or data modeling side?"

---

## Appendix: Quick-Reference Cheat Sheet

| Topic | One-line rule |
|---|---|
| Roles | Grant to roles, not users |
| Backup | Test the restore, not just the backup |
| Upgrades | One replica at a time, always |
| Quotas | Quotas = over time; settings profiles = per query |
| Data types | Narrowest type that fits |
| Schema | Wide and denormalized beats joined |
| Primary key | Low → high cardinality, leads with WHERE-clause columns |
| Partitioning | Lifecycle, not speed |

**Suggested total run time:** ~45 min core content + 10–15 min Q&A, using the "Go deeper" boxes on slides 3, 6, and 11 as flex time if the session runs short.
