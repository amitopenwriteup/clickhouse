# Lab: ClickHouse for Logs, Metrics & Traces
### Observability Backend Patterns • Comparison with Other Stores • Telemetry Schema Design

**Environment:** ClickHouse Cloud (trial account)

---

## 0. Setup

```sql
CREATE DATABASE IF NOT EXISTS otel_lab;
USE otel_lab;
```

Everything in this lab runs directly in your Cloud trial account — no OpenTelemetry Collector or external agents required. We'll hand-craft representative rows so you can focus on schema design and query patterns rather than pipeline plumbing. (Part 5 shows the real Collector config for when you want to wire up actual instrumented apps later.)

---

## Part 1 — Why ClickHouse for Observability Data?

Telemetry data (logs, metrics, traces) has a distinctive shape that maps well onto a columnar analytical database:

- **Write-once, read-many, append-only** — no updates, just a firehose of new rows
- **Time is always the primary filter** — nearly every query starts with a time range
- **High cardinality, wide schemas** — dozens of attributes (`service.name`, `host`, `trace_id`, custom tags), but any given query touches only a few columns
- **Aggregation-heavy reads** — "error rate by service," "p99 latency," "count of logs matching X" — exactly what columnar engines are built for
- **Massive volume, cost-sensitive retention** — compression ratio directly determines how much history you can afford to keep

This is the same shape ClickHouse's `MergeTree` family was designed around, which is why ClickHouse (directly, or via ClickHouse's own **ClickStack**/HyperDX distribution) has become a common choice as an observability backend, alongside purpose-built options like Elasticsearch, Loki, Tempo, and OpenObserve.

---

## Part 2 — Schema Patterns: Logs

### 2.1 A hand-rolled logs schema (OTel-aligned)

This mirrors the shape ClickHouse's own OpenTelemetry exporter produces, simplified for the lab:

```sql
CREATE TABLE otel_logs
(
    Timestamp        DateTime64(9),
    TraceId          String,
    SpanId           String,
    SeverityText     LowCardinality(String),
    SeverityNumber   UInt8,
    ServiceName      LowCardinality(String),
    Body             String,
    ResourceAttributes Map(LowCardinality(String), String),
    LogAttributes      Map(LowCardinality(String), String)
)
ENGINE = MergeTree()
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SeverityText, Timestamp)
TTL toDateTime(Timestamp) + INTERVAL 14 DAY DELETE;
```

Design choices worth noting:
- **`ORDER BY (ServiceName, SeverityText, Timestamp)`** — most log queries filter by service first ("show me logs for `checkout-service`"), then severity ("errors only"), then a time range. Put your most common equality filters before the time column.
- **`Map(LowCardinality(String), String)`** for attributes — logs have wildly varying custom fields per source; a `Map` column avoids needing a rigid predefined schema for every possible tag while staying queryable.
- **`LowCardinality(String)`** for enum-like fields (`ServiceName`, `SeverityText`) — dictionary-encodes them, shrinking storage and speeding up `GROUP BY`/filters.
- **`TTL ... DELETE`** — logs are the least valuable telemetry signal over time; a short TTL (here 14 days) keeps storage costs bounded automatically.

### 2.2 Load sample data

```sql
INSERT INTO otel_logs
SELECT
    now64(9) - randUniform(0, 3600) AS Timestamp,
    hex(rand64()) AS TraceId,
    hex(rand64()) AS SpanId,
    ['DEBUG','INFO','WARN','ERROR'][1 + rand() % 4] AS SeverityText,
    [5, 9, 13, 17][1 + rand() % 4] AS SeverityNumber,
    ['checkout-service','payments-service','api-gateway','inventory-service'][1 + rand() % 4] AS ServiceName,
    ['request completed','connection timeout','cache miss','invalid input','order placed'][1 + rand() % 5] AS Body,
    map('host.name', concat('host-', toString(rand() % 20)), 'k8s.pod.name', concat('pod-', toString(rand() % 100))) AS ResourceAttributes,
    map('http.status_code', toString([200,200,200,404,500][1 + rand() % 5])) AS LogAttributes
FROM numbers(500000);
```

### 2.3 Query patterns

```sql
-- Error rate by service, last hour
SELECT ServiceName, countIf(SeverityText = 'ERROR') AS errors, count() AS total,
       round(errors / total * 100, 2) AS error_pct
FROM otel_logs
WHERE Timestamp >= now() - INTERVAL 1 HOUR
GROUP BY ServiceName
ORDER BY error_pct DESC;

-- Free-text-ish search within a Map attribute
SELECT Timestamp, ServiceName, Body, LogAttributes['http.status_code'] AS status
FROM otel_logs
WHERE LogAttributes['http.status_code'] = '500'
ORDER BY Timestamp DESC
LIMIT 20;
```

**Exercise 1:** Add a `bloom_filter` skip index on `TraceId` (`ALTER TABLE otel_logs ADD INDEX idx_trace TraceId TYPE bloom_filter GRANULARITY 4`) and confirm with `EXPLAIN indexes = 1` that a lookup like `WHERE TraceId = '<some hex value from your data>'` benefits from it.

---

## Part 3 — Schema Patterns: Traces

### 3.1 A spans table

```sql
CREATE TABLE otel_traces
(
    Timestamp       DateTime64(9),
    TraceId         String,
    SpanId          String,
    ParentSpanId    String,
    SpanName        LowCardinality(String),
    ServiceName     LowCardinality(String),
    Duration        UInt64,   -- nanoseconds
    StatusCode      LowCardinality(String),
    SpanAttributes  Map(LowCardinality(String), String)
)
ENGINE = MergeTree()
PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SpanName, Timestamp)
TTL toDateTime(Timestamp) + INTERVAL 30 DAY DELETE;
```

- Traces get a **longer TTL than logs** (here 30 vs. 14 days) since they're lower-volume-per-event and higher-value for post-incident root-cause analysis.
- `Duration` as `UInt64` nanoseconds (the OTel-native unit) lets you compute percentiles directly without unit conversion in every query.
- `ORDER BY` again leads with the columns you filter by most (service, operation name) before time.

### 3.2 Load sample spans (simulate a simple request trace)

```sql
INSERT INTO otel_traces
WITH trace_ids AS (
    SELECT hex(rand64()) AS TraceId FROM numbers(50000)
)
SELECT
    now64(9) - randUniform(0, 3600) AS Timestamp,
    TraceId,
    hex(rand64()) AS SpanId,
    '' AS ParentSpanId,
    ['GET /checkout','POST /payment','GET /inventory','DB query'][1 + rand() % 4] AS SpanName,
    ['api-gateway','payments-service','inventory-service','checkout-service'][1 + rand() % 4] AS ServiceName,
    (rand() % 500000000) AS Duration,
    ['OK','OK','OK','ERROR'][1 + rand() % 4] AS StatusCode,
    map('http.method', 'GET') AS SpanAttributes
FROM trace_ids;
```

### 3.3 Query patterns

```sql
-- p50 / p95 / p99 latency by span name (nanoseconds -> ms)
SELECT
    SpanName,
    round(quantile(0.50)(Duration) / 1e6, 2) AS p50_ms,
    round(quantile(0.95)(Duration) / 1e6, 2) AS p95_ms,
    round(quantile(0.99)(Duration) / 1e6, 2) AS p99_ms,
    count() AS span_count
FROM otel_traces
WHERE Timestamp >= now() - INTERVAL 1 HOUR
GROUP BY SpanName
ORDER BY p99_ms DESC;

-- Reconstruct all spans for one trace (trace waterfall)
SELECT SpanName, ServiceName, Duration, StatusCode
FROM otel_traces
WHERE TraceId = (SELECT TraceId FROM otel_traces LIMIT 1)
ORDER BY Timestamp;
```

**Exercise 2:** Compute error rate per `ServiceName` (`StatusCode = 'ERROR'`) and cross-reference it against the log error rate query from Part 2.3 — in a real deployment, correlating these by `TraceId`/`ServiceName`/time window is exactly how you'd pivot from "logs show an error spike" to "here's the slow/failing span that caused it."

---

## Part 4 — Schema Patterns: Metrics

Metrics differ from logs/traces: they're regularly-sampled numeric time series, not discrete events, so the schema and storage strategy differ meaningfully.

### 4.1 A gauge/sum metrics table

```sql
CREATE TABLE otel_metrics
(
    Timestamp    DateTime64(9),
    MetricName   LowCardinality(String),
    ServiceName  LowCardinality(String),
    Value        Float64,
    Attributes   Map(LowCardinality(String), String)
)
ENGINE = MergeTree()
PARTITION BY toDate(Timestamp)
ORDER BY (MetricName, ServiceName, Timestamp)
TTL toDateTime(Timestamp) + INTERVAL 90 DAY DELETE;
```

- Metrics typically get the **longest retention** of the three signals (here 90 days) — they're cheap to store (small, regular, highly compressible numeric values) and valuable for long-term capacity planning/trend analysis.
- `Value CODEC(Gorilla, ZSTD)` (see the compression lab) is a natural fit here since metric values are often slowly-varying floats sampled at regular intervals — add it as an exercise.

### 4.2 Load sample metrics

```sql
INSERT INTO otel_metrics
SELECT
    now64(9) - number * 10 AS Timestamp,  -- one point every 10s
    ['cpu.usage','memory.usage','request.count','request.duration'][1 + rand() % 4] AS MetricName,
    ['checkout-service','payments-service','api-gateway','inventory-service'][1 + rand() % 4] AS ServiceName,
    randUniform(0, 100) AS Value,
    map('region', 'us-east-1') AS Attributes
FROM numbers(200000);
```

### 4.3 Query patterns — this is where SummingMergeTree/AggregatingMergeTree from earlier labs come back

```sql
-- Downsample to 1-minute averages (a dashboard-friendly aggregate)
SELECT toStartOfMinute(Timestamp) AS minute, ServiceName, avg(Value) AS avg_value
FROM otel_metrics
WHERE MetricName = 'cpu.usage'
  AND Timestamp >= now() - INTERVAL 1 HOUR
GROUP BY minute, ServiceName
ORDER BY minute;
```

**Exercise 3:** Build an `AggregatingMergeTree` rollup table (as in the earlier indexing/engines lab) fed by a Materialized View that pre-computes `avg`, `min`, `max` per `MetricName`/`ServiceName`/minute — this is exactly the pattern production ClickHouse observability stacks use to keep dashboard queries fast as raw metric volume grows.

---

## Part 5 — The Real Pipeline: OpenTelemetry Collector → ClickHouse (reference, not required to run)

For when you're ready to connect actual instrumented applications instead of synthetic data:

```yaml
exporters:
  clickhouse:
    endpoint: tcp://<your-service>.clickhouse.cloud:9440?secure=true
    username: otel_writer
    password: ${CH_PASSWORD}
    database: otel_lab
    logs_table_name: otel_logs
    metrics_table_name: otel_metrics
    traces_table_name: otel_traces
    ttl: 336h   # 14 days, matches the logs TTL above

service:
  pipelines:
    logs:
      receivers: [otlp]
      exporters: [clickhouse]
    metrics:
      receivers: [otlp]
      exporters: [clickhouse]
    traces:
      receivers: [otlp]
      exporters: [clickhouse]
```

> ClickHouse's own recommendation: **disable the Collector's auto-schema-creation** and define tables manually (as you did in Parts 2–4) so you control `ORDER BY`, TTLs, and codecs rather than accepting generic defaults.

**Exercise 4 (optional, no Cloud trial limitation — just needs Docker):** If you have a local app instrumented with an OTel SDK, run an OTel Collector with the config above pointed at your Cloud trial service, and confirm real spans land in `otel_traces`.

---

## Part 6 — Comparison with Other Observability Stores

| Store | Data model | Query language | Strengths | Trade-offs |
|---|---|---|---|---|
| **ClickHouse** | Columnar, wide-event/relational | SQL | Extremely fast aggregations, high compression, one engine for logs+metrics+traces, cheap long retention | You design the schema yourself (or adopt ClickStack); no built-in agent — needs OTel Collector or similar |
| **Elasticsearch / OpenSearch** | Inverted index, document-oriented | Query DSL / Lucene | Best-in-class full-text search, mature ecosystem | Higher storage overhead per byte ingested, less efficient for pure numeric aggregation at scale |
| **Loki** | Log-line + label index (no full-text index by default) | LogQL | Very cheap storage (indexes only labels, not content), pairs natively with Grafana/Prometheus | Slower on ad-hoc full-text search across unindexed content; log-focused only |
| **Prometheus / Mimir / Thanos** | Purpose-built time-series (metrics only) | PromQL | Extremely efficient for metrics specifically, huge ecosystem (alerting, exporters) | Not designed for logs or traces at all; long-term storage needs Mimir/Thanos/Cortex layered on |
| **Tempo** | Trace-specific object storage | TraceQL | Purpose-built trace storage, cheap, integrates with Grafana | Traces only; typically paired with Loki+Prometheus for the other two signals |
| **OpenObserve** | Columnar (built partly on similar principles to ClickHouse) | SQL-like | Ships as a single self-contained binary, built-in ingestion/UI, no separate query layer needed | Newer/smaller ecosystem; in independent benchmarks results vary by workload — always benchmark your own data rather than trusting vendor numbers |

### 6.1 Where ClickHouse fits

The recurring theme: **purpose-built stores (Loki, Tempo, Prometheus) each optimize hard for one signal**, while **ClickHouse (and similarly-shaped systems like OpenObserve) optimize for unifying all three signals in one queryable store with SQL**, at the cost of you owning more schema/pipeline design decisions. ClickHouse's own **ClickStack** packages an opinionated schema, OTel Collector config, and UI (HyperDX) specifically to remove that design burden for teams who want the unified-SQL-store benefits without building it from scratch.

**Exercise 5 (discussion, no SQL required):** For a team already running Prometheus + Grafana for metrics and considering whether to add ClickHouse for logs and traces (keeping Prometheus as-is) versus migrating everything to ClickHouse/ClickStack, write 3–4 sentences on what you'd want to know before deciding (query patterns, team SQL familiarity, existing alerting investment, retention/cost requirements).

---

## Part 7 — Wrap-Up: Schema Design Principles for Telemetry

| Principle | Logs | Traces | Metrics |
|---|---|---|---|
| `ORDER BY` leads with | Service, severity | Service, span name | Metric name, service |
| Typical TTL | Short (7–14 days) | Medium (30 days) | Long (90+ days) |
| High-cardinality fields | `Map` columns for attributes | `Map` columns for span attributes | `Map` for labels, but consider dedicated columns for hot label dimensions |
| Compression focus | `LowCardinality` on enums, `ZSTD` on body text | `LowCardinality` on names, plain numeric codecs on `Duration` | `Gorilla`/`Delta` codecs — biggest compression wins of the three |
| Pre-aggregation | Rare (logs are usually queried raw) | Occasional (span duration rollups) | Common — `AggregatingMergeTree` + Materialized Views for dashboards |

## 8. Cleanup

```sql
DROP DATABASE IF EXISTS otel_lab;
```

---

### Further Reading
- ClickHouse docs: "Observability" use-case guide, OpenTelemetry integration, schema design guidance
- ClickHouse / ClickStack (formerly HyperDX): open-source observability stack built on ClickHouse + OTel
- OpenTelemetry docs: Collector exporters, semantic conventions for logs/traces/metrics
