# Lab: End-to-End OpenTelemetry → ClickHouse Pipeline
### OTel Exporter Configuration • Schema Mapping • Full Pipeline Demo • Course Recap, Q&A & Assessment

**Environment:** ClickHouse Cloud (trial account) + local Docker for the OpenTelemetry Collector

This is the capstone lab for the course. It ties together ingestion (Kafka/streaming lab), schema design (observability lab), indexing/compression (tuning lab), and system-table monitoring (Grafana lab) into one working pipeline, then closes with a recap and self-assessment.

---

## 0. What runs where

| Component | Where it runs |
|---|---|
| ClickHouse tables, schema, queries | Your ClickHouse Cloud trial service |
| OpenTelemetry Collector (Contrib distro, with the `clickhouseexporter`) | Local Docker container |
| Sample instrumented app generating real traces/logs | Local Docker container (or the Collector's built-in demo generator) |

No node-level access to ClickHouse is required — the Collector talks to your Cloud service over its public HTTPS/native endpoint, exactly like any other client.

---

## Part 1 — The ClickHouse OTel Exporter

### 1.1 What it is

The `clickhouseexporter` ships in the **OpenTelemetry Collector Contrib** distribution (not core) — it's the component that takes OTLP data the Collector has received and writes it into ClickHouse via batched `INSERT`s.

Two operating modes:
- **`create_schema: true` (default)** — the exporter auto-creates the database/tables on first startup. Fine for a quick demo (what we'll use in this lab).
- **`create_schema: false`** — recommended for production. You own the DDL (as in the observability-schema lab), and the exporter only ever issues `INSERT`s. This avoids multiple exporter replicas racing to create tables, and lets you control `ORDER BY`, TTL, partitioning, and codecs precisely. Column **names and types must match** what the exporter expects for inserts to work.

### 1.2 Minimal Collector config

Create `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 5000

exporters:
  clickhouse:
    endpoint: "tcp://<YOUR-SERVICE>.clickhouse.cloud:9440?secure=true"
    username: default
    password: "${env:CLICKHOUSE_PASSWORD}"
    database: capstone_lab
    create_schema: true
    logs_table_name: otel_logs
    traces_table_name: otel_traces
    metrics_table_name: otel_metrics
    ttl: 336h            # 14 days
    connection_params:
      compression: lz4

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [clickhouse]
```

Notes tying back to earlier labs:
- The `batch` processor is the Collector-side equivalent of the **bulk insert best practices** you learned earlier — ClickHouse recommends batches of at least ~5,000 rows (or no more than one insert request per second), and `send_batch_size: 5000` implements exactly that on the Collector side before it ever reaches ClickHouse.
- `endpoint` uses the **native protocol** (port 9440, TLS) — one of the three driver types from the ingestion lab.
- `ttl: 336h` mirrors the TTL tiering strategy from the schema-design lab.

### 1.3 Get your ClickHouse Cloud connection details

From the Cloud console: select your service → **Connect** → **Native** tab → copy the hostname. Use your trial service's default user password, or create a dedicated writer user:

```sql
CREATE DATABASE IF NOT EXISTS capstone_lab;

CREATE USER otel_writer IDENTIFIED WITH sha256_password BY 'a-strong-password';
GRANT SELECT, INSERT, CREATE TABLE ON capstone_lab.* TO otel_writer;
```

(If using `otel_writer`, update `username`/`password` in the config accordingly, and note it needs `CREATE TABLE` only because `create_schema: true` — a `create_schema: false` production setup would only need `INSERT`.)

---

## Part 2 — Run the Pipeline

### 2.1 Docker Compose for the Collector + a demo trace generator

```yaml
# docker-compose.yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml
    environment:
      CLICKHOUSE_PASSWORD: "a-strong-password"
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP

  demo-generator:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/loadgen.yaml"]
    volumes:
      - ./loadgen.yaml:/etc/otel/loadgen.yaml
    depends_on:
      - otel-collector
```

A simple synthetic load generator config (`loadgen.yaml`) using the Collector's built-in `otlp` exporter pointed back at the main Collector, with the `k8sobjects`/`hostmetrics` or a simple `otlp`-emitting receiver, is one option — but the more common approach for a first pipeline test is to instrument a small real app. If you don't have one handy, ClickHouse and OpenTelemetry both publish minimal "demo app" images you can substitute here.

### 2.2 Start it

```bash
docker compose up -d
docker compose logs -f otel-collector
```

Watch for log lines confirming successful ClickHouse connection and table creation, not connection errors.

### 2.3 Send a manual test span (no app required)

If you want to verify the pipeline without a full demo app, send a span directly via `curl` to the OTLP HTTP endpoint using a minimal JSON payload, or use a tool like `telemetrygen`:

```bash
docker run --rm --network host \
  otel/opentelemetry-collector-contrib:latest \
  telemetrygen traces --otlp-insecure --otlp-endpoint localhost:4317 --traces 50
```

This generates 50 synthetic spans and pushes them through your running Collector into ClickHouse.

---

## Part 3 — Verify Schema Mapping in ClickHouse

### 3.1 Confirm the exporter auto-created tables

```sql
SHOW TABLES FROM capstone_lab;
```

You should see `otel_logs`, `otel_traces`, `otel_metrics_gauge`/`otel_metrics_sum` (metrics get split into multiple tables by point type).

### 3.2 Inspect the auto-generated schema

```sql
SHOW CREATE TABLE capstone_lab.otel_traces;
```

Compare this against the hand-rolled `otel_traces` schema from the previous observability lab. You should recognize the same shape: `Timestamp`, `TraceId`, `SpanId`, `ServiceName`, `Duration`, `SpanAttributes` (as a `Map`), `ORDER BY` leading with service-oriented columns, and a `TTL` clause matching what you set in the exporter config.

### 3.3 Query the real ingested spans

```sql
SELECT ServiceName, SpanName, count() AS span_count, avg(Duration) / 1e6 AS avg_ms
FROM capstone_lab.otel_traces
GROUP BY ServiceName, SpanName
ORDER BY span_count DESC;
```

**Exercise 1:** Run `EXPLAIN indexes = 1` against a filtered query on `TraceId` (e.g., a single trace lookup). Since the exporter's default schema already includes a `TraceId` skip index, confirm it's being used — this is the auto-created equivalent of the index you built by hand in Part 2 of the observability-schema lab.

---

## Part 4 — Full Pipeline Health Check (tying back to the Grafana/alerting lab)

### 4.1 Confirm ingestion via system tables

```sql
SELECT event, value
FROM system.events
WHERE event IN ('InsertedRows', 'InsertQuery')
ORDER BY event;
```

```sql
SELECT toStartOfMinute(event_time) AS minute, count() AS insert_queries, sum(written_rows) AS rows_written
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%INSERT INTO%otel_%'
  AND event_time >= now() - INTERVAL 30 MINUTE
GROUP BY minute
ORDER BY minute;
```

### 4.2 Reconnect this to Grafana (optional, if you built the dashboard in the earlier lab)

Add a panel querying `capstone_lab.otel_traces` alongside your existing `system.query_log`/`system.metrics` panels — you now have both **infrastructure health** (ClickHouse's own performance) and **application telemetry** (the traces flowing through the pipeline you just built) in one dashboard, which is the end state a real observability platform is working toward.

**Exercise 2:** Set up one alert rule (from the alerting lab pattern) that fires if `rows_written` for the `otel_traces` table drops to zero for 5+ minutes — a simple "pipeline stopped ingesting" tripwire.

---

## Part 5 — Course Recap

A quick map of everything this course covered, and where each piece slots into the pipeline you just built:

| Lab | Concept | Where it showed up today |
|---|---|---|
| 1 | MergeTree family, ReplacingMergeTree, Summing/AggregatingMergeTree, Distributed | `otel_traces`/`otel_logs` are `MergeTree`; a metrics rollup would use `AggregatingMergeTree` |
| 2 | Batch vs. streaming ingestion, Kafka engine, drivers, bulk insert best practices | The Collector's `batch` processor and native-protocol exporter connection |
| 3 | Primary/skip indexes, EXPLAIN, compression codecs, query tuning | The auto-created `TraceId` skip index; `ORDER BY` shape in `SHOW CREATE TABLE` |
| 4 | Sharding, replication, cluster topology, Keeper, failover | Underlies how your Cloud service stays available while the pipeline writes to it |
| 5 | system.query_log, system.metrics, Grafana, alerting | Part 4 of this lab — pipeline health monitoring |
| 6 | ClickHouse as an observability backend, schema patterns, comparison with other stores | The `otel_logs`/`otel_traces`/`otel_metrics` schema design principles |
| 7 (this lab) | OTel exporter, schema mapping, end-to-end pipeline | Everything above, wired together and actually running |

---

## Part 6 — Q&A: Common Questions

**Q: Why does the metrics table get split into `otel_metrics_gauge`, `otel_metrics_sum`, etc.?**
A: OTel metrics have distinct point types (gauge, sum, histogram, summary) with different fields; the exporter models each as its own ClickHouse table rather than forcing them into one lossy shared schema.

**Q: Should I run the Collector as an Agent (sidecar per service) or a Gateway (central fleet)?**
A: Agent-per-host/service is common for resource attribution and local buffering; a Gateway layer in front of ClickHouse centralizes batching, TLS termination, and exporter config so individual services don't each need ClickHouse credentials. Larger deployments often run both: Agents → Gateway → ClickHouse.

**Q: Why `create_schema: false` in production if `true` "just works"?**
A: Auto-created schemas use generic defaults for `ORDER BY`, TTL, and codecs. You already know from the tuning lab that `ORDER BY` choice is close to irreversible without a table rebuild — get it right upfront by owning the DDL.

**Q: How does this compare to just using ClickStack instead of hand-wiring the Collector?**
A: ClickStack packages an opinionated Collector config, pre-tuned schema, and a UI (HyperDX) on top of exactly this pipeline. This lab showed you the mechanics underneath so you can diverge from ClickStack's defaults when your workload needs something custom (e.g., a different retention tier, a custom skip index, a non-standard attribute schema).

---

## Part 7 — Self-Assessment

Answer these without looking back at the labs, then check yourself against the reference answers.

1. **Why does the OTel Collector's `batch` processor matter for ClickHouse specifically, beyond generic network efficiency?**
   <details><summary>Reference answer</summary>Every insert creates a new MergeTree part; batching many spans/logs per insert (≥5,000 rows recommended) avoids the "too many parts" problem and reduces background merge pressure, directly applying the bulk-insert best practices from Lab 2.</details>

2. **You're deciding between `create_schema: true` and `false` for a production deployment with 20 Collector replicas. Which do you choose, and what's the specific failure mode `true` risks at that scale?**
   <details><summary>Reference answer</summary><code>create_schema: false</code>. With 20 replicas all set to auto-create, they can race to create the same database/tables on startup, and you lose control over schema evolution across upgrades. Own the DDL centrally instead.</details>

3. **A dashboard query filtering `WHERE ServiceName = 'checkout-service' AND SpanName = 'POST /payment'` is slow. What's the first thing you check, and why?**
   <details><summary>Reference answer</summary><code>EXPLAIN indexes = 1</code>, checking the <code>Granules: A/B</code> ratio for the primary key. If <code>ServiceName</code> and <code>SpanName</code> are the leading <code>ORDER BY</code> columns (as designed in Lab 6), this should already prune heavily; if not, that's the root cause, not a missing skip index.</details>

4. **Why does the metrics table get the longest TTL of the three signals, and traces get a longer TTL than logs?**
   <details><summary>Reference answer</summary>Value-density and storage cost per signal: metrics are small, regular, and highly compressible (cheap to retain long), and remain useful for long-term trend/capacity analysis. Traces are moderate volume and high value for incident root-cause. Logs are the highest volume and lowest long-term value per byte, so they get trimmed fastest.</details>

5. **In a ClickHouse Cloud trial account, which parts of the sharding/replication/failover lab could you actually execute directly against Cloud, and which required a local Docker cluster — and why?**
   <details><summary>Reference answer</summary>Topology/replica inspection (<code>system.clusters</code>, <code>system.replicas</code>) and cluster design reasoning ran fine on Cloud. Actually killing a Keeper node or a replica to observe failover required local Docker, because Cloud's SharedMergeTree architecture is fully managed — there's no node-level access in a trial account.</details>

6. **Name one observability store better suited than ClickHouse for pure full-text log search, and one better suited for metrics-only workloads at very large scale, and explain why in one sentence each.**
   <details><summary>Reference answer</summary>Elasticsearch/OpenSearch for full-text search — its inverted-index model is purpose-built for that, where ClickHouse's columnar/skip-index model is comparatively weaker. Prometheus/Mimir/Thanos for metrics-only — purpose-built time-series storage and PromQL ecosystem outperform a general-purpose columnar store for that single signal.</details>

---

## 8. Cleanup

```sql
DROP DATABASE IF EXISTS capstone_lab;
DROP USER IF EXISTS otel_writer;
```

```bash
docker compose down -v
```

---

### Further Reading
- ClickHouse docs: "Integrating OpenTelemetry for data collection"
- OpenTelemetry Collector Contrib: `clickhouseexporter` README and config reference
- ClickHouse / ClickStack: reference architecture for a fully packaged OTel + ClickHouse + HyperDX stack

---

**Course complete.** You've now built and reasoned about ClickHouse across storage engines, ingestion, indexing/tuning, cluster architecture, observability monitoring, and a full telemetry pipeline — end to end, on a free trial account.
