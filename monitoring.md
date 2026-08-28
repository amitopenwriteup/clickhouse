# Lab: Observability — System Tables, Grafana & Alerting
### system.query_log / system.metrics • Integrating with Grafana • Alerting on Cluster Health

**Environment:** ClickHouse Cloud (trial account) + a local Grafana instance (free, via Docker)

---

## 0. What a Cloud trial account can and can't show you

| What you asked for | Can you do it in a Cloud trial account? |
|---|---|
| Query `system.query_log`, `system.metrics`, `system.events`, etc. | ✅ Yes — fully available, read-only system tables |
| Connect Grafana to your ClickHouse Cloud service | ✅ Yes — Cloud exposes a public HTTPS/native endpoint made exactly for this |
| Build dashboards on query performance / cluster health | ✅ Yes |
| Configure Grafana Alerting rules against ClickHouse metrics | ✅ Yes |
| Edit server-side `config.xml` metric exporters, or install Prometheus directly on a ClickHouse node | ❌ No — no node/OS access in a managed trial service |

Everything in this lab runs against the system tables ClickHouse Cloud already exposes — no server config access needed.

---

## Part 1 — Setup

```sql
CREATE DATABASE IF NOT EXISTS obs_lab;
USE obs_lab;

CREATE TABLE demo_events
(
    event_time DateTime,
    user_id    UInt64,
    event_type String,
    value      Float64
)
ENGINE = MergeTree()
ORDER BY (event_type, event_time);

INSERT INTO demo_events
SELECT
    now() - randUniform(0, 3600),
    rand() % 10000,
    ['click','view','purchase','error'][1 + rand() % 4],
    randUniform(0, 500)
FROM numbers(200000);
```

Run a handful of varied queries against it so `system.query_log` has something interesting to show:

```sql
SELECT count() FROM demo_events;
SELECT event_type, count() FROM demo_events GROUP BY event_type;
SELECT avg(value) FROM demo_events WHERE event_type = 'purchase';
SELECT * FROM demo_events ORDER BY value DESC LIMIT 5;
```

---

## Part 2 — system.query_log

`system.query_log` records every query executed (by default, after it finishes — there's a short async flush delay). It's the single most useful table for both performance tuning and health monitoring.

### 2.1 Basic shape

```sql
SELECT
    event_time,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage,
    type,
    user,
    query
FROM system.query_log
WHERE event_time >= now() - INTERVAL 15 MINUTE
ORDER BY event_time DESC
LIMIT 20;
```

Key `type` values: `QueryStart`, `QueryFinish`, `ExceptionBeforeStart`, `ExceptionWhileProcessing`. For most analysis, filter to `type = 'QueryFinish'` (successful) or explicitly include exception types when hunting for failures.

### 2.2 Slowest queries in the last hour

```sql
SELECT
    query_duration_ms,
    read_rows,
    formatReadableSize(read_bytes) AS read_size,
    user,
    substring(query, 1, 100) AS query_preview
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY query_duration_ms DESC
LIMIT 10;
```

### 2.3 Failed queries

```sql
SELECT event_time, user, exception, substring(query, 1, 150) AS query_preview
FROM system.query_log
WHERE type IN ('ExceptionBeforeStart', 'ExceptionWhileProcessing')
  AND event_time >= now() - INTERVAL 1 DAY
ORDER BY event_time DESC
LIMIT 20;
```

### 2.4 Query volume and load over time (the basis of a Grafana panel)

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    count() AS queries,
    avg(query_duration_ms) AS avg_duration_ms,
    sum(read_rows) AS total_rows_read
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

**Exercise 1:** Deliberately run a query that will fail (e.g., `SELECT * FROM does_not_exist`). Find it in `system.query_log` via the exception-filtered query above.

---

## Part 3 — system.metrics, system.events, system.asynchronous_metrics

These three tables cover different time horizons of server health:

| Table | What it shows | Update pattern |
|---|---|---|
| `system.metrics` | Current, instantaneous gauge values (active connections, running queries, memory) | Live snapshot |
| `system.events` | Cumulative counters since server start (total queries, total merges, total errors) | Monotonically increasing |
| `system.asynchronous_metrics` | Periodically sampled gauges (disk usage, replication queue size, uptime) | Refreshed every ~60s |

### 3.1 Current live metrics

```sql
SELECT metric, value, description
FROM system.metrics
WHERE metric IN ('Query', 'TCPConnection', 'MemoryTracking', 'BackgroundMergesAndMutationsPoolTask')
ORDER BY metric;
```

### 3.2 Cumulative event counters

```sql
SELECT event, value, description
FROM system.events
WHERE event IN ('Query', 'SelectQuery', 'InsertQuery', 'FailedQuery', 'FailedSelectQuery')
ORDER BY event;
```

> Because `system.events` counters only go up, they're most useful as **rate** metrics in Grafana (`increase()`/derivative over time), not as raw values.

### 3.3 Asynchronous / periodic metrics

```sql
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric ILIKE '%replicat%' OR metric ILIKE '%disk%' OR metric = 'Uptime'
ORDER BY metric;
```

### 3.4 Cluster/replica health snapshot

```sql
SELECT database, table, is_leader, total_replicas, active_replicas, absolute_delay, queue_size
FROM system.replicas;

SELECT cluster, shard_num, replica_num, host_name, is_local
FROM system.clusters;
```

**Exercise 2:** Run the queries in 3.1–3.4 twice, five minutes apart. Identify which values changed (metrics/events) and which stayed static (topology), and explain why.

---

## Part 4 — Connecting Grafana to ClickHouse Cloud

### 4.1 Get your Cloud connection details

In the ClickHouse Cloud console: select your trial service → **Connect** → choose **HTTPS** (Grafana's official plugin uses HTTP(S), port 8443) — note the host, port, username, and password/generate one.

### 4.2 Run Grafana locally (free, via Docker)

```bash
docker run -d --name grafana -p 3000:3000 \
  -e "GF_INSTALL_PLUGINS=grafana-clickhouse-datasource" \
  grafana/grafana:latest
```

Open `http://localhost:3000` (default login `admin` / `admin`).

### 4.3 (Recommended) Create a read-only Grafana user in ClickHouse first

Never point dashboards at a full-access account — Grafana does not validate that queries are safe, and a careless panel query could run `ALTER`/`DELETE`.

```sql
CREATE USER grafana_reader IDENTIFIED WITH sha256_password BY 'a-strong-password'
SETTINGS readonly = 1;

GRANT SELECT ON obs_lab.* TO grafana_reader;
GRANT SELECT ON system.query_log TO grafana_reader;
GRANT SELECT ON system.metrics TO grafana_reader;
GRANT SELECT ON system.events TO grafana_reader;
GRANT SELECT ON system.asynchronous_metrics TO grafana_reader;
GRANT SELECT ON system.replicas TO grafana_reader;
GRANT SELECT ON system.clusters TO grafana_reader;
```

### 4.4 Add the data source in Grafana

1. **Connections → Add new connection → search "ClickHouse"** → install/select the official Grafana Labs plugin.
2. **Add new data source**, fill in:
   - **Server host address**: your Cloud service hostname (from step 4.1)
   - **Server port**: `8443` (HTTPS) or `9440` (native TLS)
   - **Protocol**: HTTP or Native — HTTP is simplest for a first pass
   - **Secure connection**: enabled (Cloud requires TLS)
   - **Username / Password**: `grafana_reader` / the password you set
3. Click **Save & Test** — you should see a success confirmation.

### 4.5 Build your first panel

New dashboard → Add panel → select the ClickHouse data source → raw SQL editor:

```sql
SELECT
    toStartOfMinute(event_time) AS time,
    count() AS queries
FROM system.query_log
WHERE type = 'QueryFinish'
  AND $__timeFilter(event_time)
GROUP BY time
ORDER BY time
```

`$__timeFilter(...)` is a Grafana macro that automatically substitutes the dashboard's selected time range.

### 4.6 A small starter dashboard

Add three more panels using the queries from Part 2/3, adapted with `$__timeFilter`:
- **Avg query duration over time** (from §2.4, using `$__timeFilter(event_time)`)
- **Failed queries count** (from §2.3, as a time series with `count()` grouped by minute)
- **Active replicas / replication delay** (from §3.4, as a table or stat panel — this one won't use `$__timeFilter` since it's a live snapshot, not historical)

**Exercise 3:** Save the dashboard. Then deliberately generate a slow query (e.g., a full scan over `demo_events` with a non-indexed filter) and confirm it shows up in your "Avg query duration" panel within a minute or two.

---

## Part 5 — Alerting on Cluster Health

Grafana Alerting evaluates a panel's query on a schedule and fires when a threshold condition is met.

### 5.1 Alert: failed query rate spike

Query (as an alert rule condition, using a rolling window):

```sql
SELECT count() AS failed_count
FROM system.query_log
WHERE type IN ('ExceptionBeforeStart', 'ExceptionWhileProcessing')
  AND event_time >= now() - INTERVAL 5 MINUTE
```

In Grafana: **Alerting → Alert rules → New alert rule**
- Query: the SQL above against your ClickHouse data source
- Condition: `WHEN last() OF query IS ABOVE 5` (tune threshold to your traffic)
- Evaluation interval: every `1m`, for `5m` before firing (avoids flapping on a single blip)
- Labels/annotations: `severity: warning`, a clear summary like "Failed query rate elevated"

### 5.2 Alert: replication delay

```sql
SELECT max(absolute_delay) AS max_delay
FROM system.replicas
```

- Condition: `WHEN last() OF query IS ABOVE 300` (seconds) — tune to your tolerance
- This is a **shared-storage caveat on Cloud**: since Cloud's SharedMergeTree doesn't rely on classic part-copying the way self-managed `ReplicatedMergeTree` does, `absolute_delay` may behave differently (often near-zero) compared to a self-managed cluster. Test this query against your actual trial service and adjust the threshold based on observed baseline values rather than assuming a self-managed cluster's typical numbers apply.

### 5.3 Alert: memory pressure

```sql
SELECT value AS memory_bytes
FROM system.metrics
WHERE metric = 'MemoryTracking'
```

- Condition: threshold set relative to your Cloud service's provisioned memory (check the Cloud console for your trial tier's RAM allocation, then alert at ~80% of that).

### 5.4 Alert: no active replicas (hard outage signal)

```sql
SELECT count() AS unhealthy_tables
FROM system.replicas
WHERE active_replicas < total_replicas
```

- Condition: `WHEN last() OF query IS ABOVE 0`
- This is the closest thing to a "cluster health" tripwire — the moment any replicated table has fewer active replicas than expected.

### 5.5 Configure a notification channel

**Alerting → Contact points → Add contact point** — choose a channel available to you for testing (e.g., a webhook to a service like webhook.site, or email if your Grafana instance has SMTP configured). Attach it to the alert rules above via a **Notification policy**.

**Exercise 4:** Trigger the "failed query rate" alert intentionally by running 6+ deliberately broken queries (`SELECT * FROM nonexistent_table_xyz`) within a 5-minute window. Confirm the alert transitions from `Normal` → `Pending` → `Firing` in the Grafana Alerting UI, and that your contact point received the notification.

---

## Part 6 — Wrap-Up Comparison

| System table | Best for | Grafana panel type |
|---|---|---|
| `system.query_log` | Query performance, failures, throughput trends | Time series |
| `system.metrics` | Live instantaneous state (connections, running queries) | Stat / gauge |
| `system.events` | Cumulative counters → convert to rates | Time series (rate) |
| `system.asynchronous_metrics` | Periodic health gauges (disk, uptime, replication) | Stat / gauge / table |
| `system.replicas` / `system.clusters` | Topology and replication health | Table / stat |

## 7. Cleanup

```sql
DROP USER IF EXISTS grafana_reader;
DROP DATABASE IF EXISTS obs_lab;
```

```bash
docker stop grafana && docker rm grafana
```

---

### Further Reading
- ClickHouse docs: "ClickHouse data source plugin for Grafana," system tables reference (`system.query_log`, `system.metrics`, `system.events`, `system.asynchronous_metrics`)
- Grafana docs: Alerting — alert rules, contact points, notification policies
