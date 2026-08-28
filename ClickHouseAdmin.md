# ClickHouse Cloud Lab
### User Management & Roles/Permissions · Backup & Restore · Version Upgrades · Quotas & Resource Management

**Where to run this:** Your own ClickHouse Cloud service — both the **SQL console** and the **Cloud console** (the service's settings pages).
**Data:** A small SQL-generated dataset in Setup, used to test permissions and quotas hands-on.

> This lab is different in kind from a pure-SQL lab. **User management/RBAC and quotas** are fully hands-on in SQL. **Backup/restore and version upgrades** are managed by ClickHouse Cloud itself, so those sections are guided console walkthroughs rather than SQL you run — that's a deliberate, accurate reflection of how Cloud works, not a shortcut.

---

## Setup — A Small Dataset to Protect *(and throttle)*

```sql
CREATE TABLE sales
(
    sale_id     UInt64,
    country     LowCardinality(String),
    amount      Decimal(10,2),
    sale_date   Date
)
ENGINE = MergeTree
ORDER BY (country, sale_date);

INSERT INTO sales
SELECT
    number + 1 AS sale_id,
    arrayElement(['USA','UK','India','Germany','Brazil'], (number % 5) + 1) AS country,
    CAST(round((rand() % 100000) / 100.0, 2) AS Decimal(10,2)) AS amount,
    toDate('2023-01-01') + toIntervalDay(number % 700) AS sale_date
FROM numbers(50000);
```

```sql
SELECT count() FROM sales;
SELECT * FROM sales LIMIT 5;
```

---

## Part 1 — User Management & Roles/Permissions

ClickHouse Cloud actually has **two separate permission layers** — don't conflate them:

1. **Organization / console access** (who can log into the Cloud console, invite teammates, change billing, create or delete services). This is managed under your organization's **Members** and **Roles** pages in the Cloud console UI — it's not SQL.
2. **Database-level users and roles** (who can connect via SQL and what they can query). This is the standard ClickHouse SQL RBAC system — fully scriptable, and what this section focuses on.

### Step 1 — Look at what already exists

Every ClickHouse Cloud service starts with a `default` user. Inspect the current state:

```sql
SHOW USERS;
SHOW ROLES;
SHOW GRANTS FOR default;
```

### Step 2 — Create a role before creating users

Roles are the reusable unit of permissions — grant to a role, then hand the role to users, rather than granting directly to individuals.

```sql
CREATE ROLE analytics_readonly;

GRANT SELECT ON default.sales TO analytics_readonly;
```

### Step 3 — Create a user and assign the role

```sql
CREATE USER analyst_alice IDENTIFIED WITH sha256_password BY 'Choose-A-Strong-Password-1!';

GRANT analytics_readonly TO analyst_alice;
```

**Note:** Always use `IDENTIFIED WITH sha256_password BY '...'` (or another explicit hash method) rather than leaving a user with no password, even in a lab. ClickHouse Cloud enforces TLS in transit either way, but weak/missing credentials are still a real risk if this service is reachable from outside your IP allow list.

### Step 4 — Verify the grant, then test it

```sql
SHOW GRANTS FOR analyst_alice;
```

Open a **new SQL console tab/connection** and log in as `analyst_alice` (or connect with a client using those credentials) to confirm:

```sql
-- Should succeed
SELECT country, sum(amount) FROM sales GROUP BY country;

-- Should fail — no INSERT privilege granted
INSERT INTO sales VALUES (999999, 'USA', 10.00, today());
```

### Step 5 — Row-level security with a Row Policy

Restrict `analyst_alice` to only see `India` and `Brazil` rows, without changing the `SELECT` grant itself.

```sql
CREATE ROW POLICY india_brazil_only ON default.sales
FOR SELECT USING country IN ('India', 'Brazil')
TO analyst_alice;
```

Re-run the grouped `SELECT` as `analyst_alice` — it now silently excludes the other three countries. This is enforced at the storage layer, not by rewriting the query, so it applies even to `count()`, joins, and subqueries against `sales`.

### Step 6 — Settings profiles: bound what a role is even allowed to try

```sql
CREATE SETTINGS PROFILE limited_resources
SETTINGS max_execution_time = 20, max_memory_usage = 2000000000
TO analytics_readonly;
```

**Note:** This caps every query run under `analytics_readonly` to 20 seconds and ~2 GB of memory, independent of the quota system in Part 4 — a settings profile bounds a *single query's* footprint; a quota bounds *cumulative usage over time*.

### Step 7 — Inspect and clean up when done experimenting

```sql
SHOW CREATE USER analyst_alice;
SELECT * FROM system.role_grants WHERE user_name = 'analyst_alice';
SELECT * FROM system.row_policies;
```

Leave `analyst_alice` and the role in place — Part 4 reuses them.

---

## Part 2 — Backup & Restore

Unlike self-hosted ClickHouse, you do **not** run `BACKUP`/`RESTORE` SQL statements against a ClickHouse Cloud service by default — Cloud takes automatic backups for you, and restoring provisions a **new service** from a backup rather than overwriting the original in place. Treat the steps below as a guided console walkthrough.

### Step 8 — Find your current backup configuration

In the **Cloud console**:
1. Open your service.
2. Go to **Settings** (or the service's **Backups** tab, depending on your console version).
3. Note the **backup schedule** and **retention period** — Cloud takes scheduled backups automatically and retains a rolling window of them.

### Step 9 — Trigger an on-demand backup

1. From the same **Backups** section, look for a **Back up now** (or equivalent) action.
2. Trigger it, then watch the backup appear in the list with a timestamp and status.

**Note:** An on-demand backup doesn't replace the schedule — it's a snapshot you're taking deliberately, e.g. right before a risky schema change.

### Step 10 — Understand what "restore" actually does

1. In the **Backups** list, select an existing backup and choose **Restore**.
2. Confirm that this provisions a **brand-new service** populated from that backup's data, rather than rewinding your current service in place.
3. Note the new service's connection details and (if shown) its one-time password — Cloud typically shows a generated password only once at creation time.

**Note:** Because restore creates a new service, your original service and its `analyst_alice` user/role setup from Part 1 are untouched by this process — a useful safety property, but it also means restoring doesn't "undo" a mistake on the live service by itself; you still need to redirect traffic to the restored service or migrate data back.

### Step 11 — (Optional, advanced) Inspect backups via the API

If you've generated a Cloud API key, you can list backups programmatically instead of using the console:

```bash
curl -u '<API_KEY>:<API_SECRET>' \
  "https://api.clickhouse.cloud/v1/organizations/<ORG_ID>/services/<SERVICE_ID>/backups"
```

This is the same data the console's Backups tab shows — useful for scripting backup audits or triggering restores from automation rather than the UI.

---

## Part 3 — Version Upgrades

ClickHouse Cloud manages ClickHouse version upgrades for you — there's no `apt upgrade` equivalent to run. What you control is *how eagerly* your service receives new versions.

### Step 12 — Check your current version

```sql
SELECT version();
```

### Step 13 — Find your release channel setting

1. In the **Cloud console**, open your service's **Settings** page.
2. Look for a release-channel control, typically labeled something like **Enroll in fast releases**.

**Note:** By default, services are on the **Regular** release channel — updates roll out after they've soaked in production for a while. Enrolling in **Fast releases** gets you new ClickHouse versions and Cloud features sooner, at the cost of being closer to the leading edge. This is a per-service toggle, so you can put a staging service on Fast releases and keep production on Regular.

### Step 14 — Know what to expect during an upgrade

1. Upgrades are applied by ClickHouse Cloud during a maintenance window; replicas are upgraded in a rolling fashion to avoid full downtime on multi-replica services.
2. A single-replica (e.g. Basic-tier) service will briefly be unavailable during its upgrade window — worth knowing before you rely on one for something latency-sensitive.
3. Re-run `SELECT version();` after a scheduled maintenance window to confirm the new version took effect.

---

## Part 4 — Quotas & Resource Management

Two different things fall under "resource management" in Cloud: **SQL-level quotas** (limiting what a *user* can do over time) and **service-level compute sizing** (how much *hardware* the whole service gets). Both matter; only the first is something you script.

### Step 15 — Create a quota and attach it to a role

```sql
CREATE QUOTA analyst_quota
KEYED BY user_name
FOR INTERVAL 1 HOUR MAX QUERIES = 100, MAX ERRORS = 20, MAX RESULT_ROWS = 1000000
TO analytics_readonly;
```

**Note:** `KEYED BY user_name` means the limits apply *per user* under this role, not shared across everyone who holds the role. `FOR INTERVAL 1 HOUR` is a rolling window — you can stack multiple `FOR INTERVAL ...` clauses (e.g. also `FOR INTERVAL 1 DAY MAX QUERIES = 1000`) on the same quota for layered limits.

### Step 16 — Watch the quota get consumed

As `analyst_alice`, run a handful of queries against `sales`, then check usage as an admin:

```sql
SHOW QUOTA;

SELECT *
FROM system.quotas_usage
WHERE quota_name = 'analyst_quota';
```

### Step 17 — Trigger the limit deliberately

Lower the limit temporarily to see the enforcement itself, not just the accounting:

```sql
ALTER QUOTA analyst_quota FOR INTERVAL 1 HOUR MAX QUERIES = 2;
```

Now run 3+ queries as `analyst_alice` in that hour — the query beyond the limit should fail with a quota-exceeded error rather than simply running slowly. Put the limit back afterward:

```sql
ALTER QUOTA analyst_quota FOR INTERVAL 1 HOUR MAX QUERIES = 100, MAX ERRORS = 20, MAX RESULT_ROWS = 1000000;
```

### Step 18 — Service-level resource management (console)

SQL quotas throttle *users*; the service itself has separate compute controls:

1. In the **Cloud console**, open your service's **Settings**.
2. Find the compute/scaling section, which typically shows:
   - **Replica size** (vCPU/memory per replica) — either fixed or a min/max range if **vertical auto-scaling** is enabled.
   - **Number of replicas** if horizontal scaling is available on your tier.
3. Note the current min/max bounds. Auto-scaling lets the service grow toward `max` under load and shrink back toward `min` when idle, rather than running at peak size all the time.

**Note:** This is the lever for "the whole service is too slow/expensive," while quotas (Steps 15–17) and settings profiles (Part 1, Step 6) are the levers for "this specific user/role is using more than its share." Sizing the service larger doesn't stop one runaway query from starving everyone else — that's what quotas and profiles are for.

---

## Wrap-Up

**What you covered:**
- Two permission layers in Cloud (org/console access vs. database RBAC), and hands-on `CREATE USER` / `CREATE ROLE` / `GRANT` / row policies / settings profiles
- How Cloud backups and restores actually work: automatic scheduled backups, on-demand backups, and restore-as-new-service (not in-place rollback)
- How version upgrades are managed by Cloud, and the Regular vs. Fast release channel toggle
- SQL-level `CREATE QUOTA` for per-user throttling, verified by actually tripping the limit, plus where service-level compute sizing lives in the console

**Cleanup (optional):**

```sql
DROP QUOTA IF EXISTS analyst_quota;
DROP ROW POLICY IF EXISTS india_brazil_only ON default.sales;
DROP SETTINGS PROFILE IF EXISTS limited_resources;
DROP USER IF EXISTS analyst_alice;
DROP ROLE IF EXISTS analytics_readonly;
DROP TABLE IF EXISTS sales;
```

**Where to go next:**
- Repeat Part 1 with `GRANTEES` and `WITH GRANT OPTION` to explore delegated permission management (letting a role grant a subset of its own privileges to others).
- Combine a row policy with a quota keyed by a custom `client_key` to rate-limit by API key/application instead of by database user.
