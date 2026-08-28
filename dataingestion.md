# Lab: ClickHouse Data Ingestion
### Batch vs Streaming • Kafka Table Engine • Native/JDBC/ODBC Drivers • Bulk Insert Best Practices

**Environment:** ClickHouse Cloud (SQL console, `clickhouse-client`, and a local shell for driver examples)

---

## 0. Setup

```sql
CREATE DATABASE IF NOT EXISTS ingestion_lab;
USE ingestion_lab;
```

You'll also want, for the driver sections:
- A recent Python 3 environment (`pip install clickhouse-connect`)
- (Optional) Java 11+ and a JDBC-capable client (e.g., DBeaver) for the JDBC section
- Your ClickHouse Cloud connection details: host, port, username, password (from the Cloud console **Connect** panel)

---

## 1. Batch vs. Streaming Ingestion — Concepts

| | Batch | Streaming |
|---|---|---|
| **Pattern** | Large, periodic loads (files, exports, scheduled jobs) | Continuous, near-real-time flow of events |
| **Typical sources** | S3/GCS files, CSV/Parquet exports, nightly ETL | Kafka, Kinesis, application event streams, CDC |
| **ClickHouse mechanism** | `INSERT ... SELECT`, table functions (`s3`, `url`, `file`), bulk `INSERT` | Kafka table engine, ClickPipes, async inserts |
| **Latency** | Minutes to hours | Seconds |
| **Tuning goal** | Maximize throughput per job | Maximize throughput per second while keeping part count low |

The core tension in both cases is the same: **ClickHouse wants few, large parts, not many small ones.** Batch ingestion naturally produces large parts. Streaming ingestion, by nature, tends to produce many small ones — which is why special mechanisms (Kafka engine + materialized views, async inserts) exist to reshape a stream into batch-like writes.

### 1.1 Exercise: simulate a batch load with a table function

```sql
CREATE TABLE nyc_taxi_sample
(
    trip_id      UInt64,
    pickup_time  DateTime,
    fare_amount  Float32,
    passenger_count UInt8
)
ENGINE = MergeTree()
ORDER BY pickup_time;

-- Batch-load directly from a public S3 bucket (table function), no local download needed
INSERT INTO nyc_taxi_sample
SELECT
    rowNumberInAllBlocks() AS trip_id,
    pickup_time,
    fare_amount,
    passenger_count
FROM s3(
    'https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_small.csv',
    'CSVWithNames'
)
LIMIT 100000;
```

This is a classic **batch ingestion** pattern: one large `INSERT ... SELECT`, pulling directly from object storage, producing a handful of large parts rather than thousands of tiny ones.

```sql
SELECT count(), min(pickup_time), max(pickup_time) FROM nyc_taxi_sample;
SELECT table, count() AS num_parts, sum(rows) AS total_rows
FROM system.parts
WHERE table = 'nyc_taxi_sample' AND active = 1
GROUP BY table;
```

**Exercise 1:** Repeat the same load but split it into 10 separate `INSERT` statements of ~10,000 rows each (using `LIMIT`/`OFFSET`, or by filtering on a range). Compare `system.parts` part counts before and after `OPTIMIZE TABLE nyc_taxi_sample FINAL`.

---

## 2. Kafka Table Engine — Streaming Ingestion

> **ClickHouse Cloud note:** The Kafka table engine works in ClickHouse Cloud, but ClickHouse recommends **ClickPipes** for production streaming ingestion on Cloud — it natively supports private networking, scales ingestion independently of query compute, and gives you built-in monitoring, so you don't have to manage consumer processes yourself. This lab teaches the Kafka table engine directly because the underlying concepts (consumer offsets, materialized views as the "glue," dead-letter handling) are the same whether you use the raw engine or ClickPipes under the hood.

### 2.1 The three-part streaming pattern

Reading from a `Kafka`-engine table **consumes** messages and advances the consumer group offset — it's effectively destructive. So the standard architecture is always three pieces:

1. A `Kafka` engine table (the "pipe" into the topic)
2. A `MergeTree` table (durable storage)
3. A Materialized View that continuously moves rows from (1) into (2)

### 2.2 Create the Kafka engine table

```sql
CREATE TABLE events_queue
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  String,
    page        String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'your-broker-host:9092',
    kafka_topic_list  = 'web-events',
    kafka_group_name  = 'clickhouse-consumer-group1',
    kafka_format      = 'JSONEachRow',
    kafka_num_consumers = 3;
```

- `kafka_num_consumers` — number of parallel consumers ClickHouse spins up; scale with topic partition count.
- `kafka_format` — any input format ClickHouse supports (`JSONEachRow`, `CSV`, `Avro`, `Protobuf`, etc.)
- This table **stores no data**. `SELECT * FROM events_queue` would consume and permanently advance the offset — avoid querying it directly outside of debugging.

### 2.3 Create the durable storage table

```sql
CREATE TABLE events_store
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  String,
    page        String
)
ENGINE = MergeTree()
ORDER BY (event_time, user_id);
```

### 2.4 Wire them together with a Materialized View

```sql
CREATE MATERIALIZED VIEW events_mv
TO events_store
AS
SELECT event_time, user_id, event_type, page
FROM events_queue;
```

As soon as this view exists, ClickHouse begins continuously consuming from `web-events` and writing into `events_store` in the background.

### 2.5 Monitor consumer health

```sql
SELECT database, table, consumer_id, assignments.topic, assignments.partition_id,
       assignments.current_offset, num_messages_read, last_poll_time
FROM system.kafka_consumers
WHERE table = 'events_queue';
```

Watch `num_messages_read` growing and check for `exceptions.text` if consumption stalls.

### 2.6 Handling malformed messages (dead-letter pattern)

```sql
CREATE TABLE events_dlq
(
    raw_payload String,
    error       String,
    failed_at   DateTime DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY failed_at;
```

Route parse failures here via `kafka_format = 'JSONAsString'` on a secondary consumer table plus a materialized view that validates/parses defensively — this keeps a single bad message from stalling the whole pipeline.

**Exercise 2:** If you don't have a live Kafka broker handy, describe (in comments) how you'd test this pipeline locally with `docker run bitnami/kafka` and a producer script, and what you'd check in `system.kafka_consumers` to confirm the pipeline is healthy versus stuck.

---

## 3. Native, JDBC, and ODBC Drivers

ClickHouse exposes multiple protocols; picking the right one matters for both performance and tooling compatibility.

| Interface | Port (default) | Best for |
|---|---|---|
| **Native TCP protocol** | 9440 (TLS) / 9000 | Fastest — used by `clickhouse-client`, `clickhouse-connect`, native Go/Python/Java drivers |
| **HTTP(S) interface** | 8443 (TLS) / 8123 | Universal — `curl`, most language clients, easy to load-balance/proxy |
| **JDBC** | via HTTP(S) | Java applications, BI tools (Tableau, Metabase), JVM ecosystems |
| **ODBC** | via HTTP(S) | Legacy BI tools, Excel, Power BI, applications expecting a standard ODBC data source |

### 3.1 Native driver — Python example

```bash
pip install clickhouse-connect
```

```python
import clickhouse_connect

client = clickhouse_connect.get_client(
    host='your-service.clickhouse.cloud',
    port=8443,
    username='default',
    password='***',
    secure=True,
)

result = client.query('SELECT count() FROM ingestion_lab.nyc_taxi_sample')
print(result.result_rows)
```

### 3.2 JDBC — connection string and quick test

Download the ClickHouse JDBC driver JAR (or add via Maven: `com.clickhouse:clickhouse-jdbc`), then:

```
jdbc:ch://your-service.clickhouse.cloud:8443/ingestion_lab?ssl=true&user=default&password=***
```

```java
Connection conn = DriverManager.getConnection(
    "jdbc:ch://your-service.clickhouse.cloud:8443/ingestion_lab?ssl=true",
    "default", "***"
);
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT count() FROM nyc_taxi_sample");
while (rs.next()) {
    System.out.println(rs.getLong(1));
}
```

Use JDBC when connecting BI tools (Tableau, Metabase, DBeaver) or JVM-based ETL frameworks (Spark, Flink connectors often use JDBC or a native connector alongside it).

### 3.3 ODBC — when you need it

ODBC is mostly relevant for:
- Excel / Power BI "Get Data → ODBC" workflows
- Legacy Windows applications with no native ClickHouse support

Setup pattern:
1. Install the ClickHouse ODBC driver for your OS.
2. Create a DSN (Data Source Name) pointing at your Cloud service's HTTPS endpoint, port 8443, with TLS enabled.
3. Test the DSN with `isql` (Linux/Mac) or the Windows ODBC Data Source Administrator before connecting your application.

**Exercise 3:** Connect to your ClickHouse Cloud service with `clickhouse-connect` (native/HTTP) **and** with a JDBC client (e.g. DBeaver), running the same `SELECT count()` query from both. Compare round-trip latency reported by each tool.

---

## 4. Bulk Insert Best Practices

### 4.1 The golden rule: batch size

Aim for **1,000–100,000 rows per INSERT**, and roughly **one insert query per second** as an upper bound on frequency — background merges need time to consolidate parts before the next wave arrives.

```sql
-- Bad: one row per statement — creates a new part every time
INSERT INTO nyc_taxi_sample VALUES (1, now(), 12.50, 1);
INSERT INTO nyc_taxi_sample VALUES (2, now(), 9.00, 2);

-- Good: one statement, many rows
INSERT INTO nyc_taxi_sample VALUES
(1, now(), 12.50, 1),
(2, now(), 9.00, 2),
(3, now(), 25.75, 4);
```

### 4.2 Client-side batching pattern (Python)

```python
import clickhouse_connect

client = clickhouse_connect.get_client(host='your-service.clickhouse.cloud', secure=True, ...)

batch = []
batch_size = 50_000

for row in event_stream():           # your own data source
    batch.append(row)
    if len(batch) >= batch_size:
        client.insert('events_store', batch,
                       column_names=['event_time', 'user_id', 'event_type', 'page'])
        batch = []

if batch:                            # flush remainder
    client.insert('events_store', batch,
                   column_names=['event_time', 'user_id', 'event_type', 'page'])
```

### 4.3 Async inserts — when client-side batching isn't possible

If you have many independent, uncoordinated sources (IoT devices, microservices each inserting a few rows), use **async inserts** to shift batching responsibility to the server:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 1;      -- block until the buffer flushes (safer, still async server-side)
SET async_insert_max_data_size = 10485760;  -- 10 MB buffer
SET async_insert_busy_timeout_ms = 1000;    -- flush at least every 1s

INSERT INTO events_store VALUES (now(), 42, 'click', '/home');
```

```sql
-- Verify async inserts are being buffered/flushed correctly
SELECT event_time, table, rows, bytes, status
FROM system.asynchronous_insert_log
ORDER BY event_time DESC
LIMIT 20;
```

> As of recent ClickHouse versions, deduplication (`deduplicate_insert`, enabled by default) applies to async inserts too, so retried inserts after a timeout or dropped connection are safe rather than creating duplicate rows — as long as the table keeps a deduplication log (`ReplicatedMergeTree` does by default; plain `MergeTree` needs `non_replicated_deduplication_window` set).

### 4.4 Loading directly from files/object storage (avoids the client round-trip entirely)

```sql
INSERT INTO events_store
SELECT * FROM s3(
    'https://your-bucket.s3.amazonaws.com/events/*.parquet',
    'Parquet'
);
```

Table functions (`s3`, `gcs`, `url`, `file`, `hdfs`) let ClickHouse pull and parse data server-side in large blocks — usually the most efficient batch path when your data already lives in object storage.

### 4.5 Common pitfalls checklist

- ❌ Row-by-row `INSERT` loops from application code → ✅ batch client-side or use async inserts
- ❌ Writing through a `Distributed` table for high-frequency inserts (adds a network hop and buffering complexity) → ✅ insert directly into a local/shard table when possible, or ensure `internal_replication` and batching are configured correctly
- ❌ Ignoring `system.parts` / "Too many parts" errors → ✅ monitor `system.part_log` and `system.parts`, and increase batch size or async buffer thresholds
- ❌ Assuming synchronous inserts always create exactly one part → ✅ know that large batches may still be split; check part count after load

### 4.6 Exercise: measure the difference

```sql
-- Time 1,000 single-row inserts (small demo, don't run more in production!)
-- vs. one 1,000-row batched insert
-- Compare system.parts counts and system.query_log query_duration_ms for each approach
SELECT type, query_duration_ms, read_rows, written_rows
FROM system.query_log
WHERE query LIKE '%events_store%' AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 20;
```

**Exercise 4:** Run both approaches (many single-row inserts vs. one batched insert of the same total rows) against a scratch table, then compare part counts in `system.parts` and total wall-clock time. Write a 2–3 sentence conclusion.

---

## 5. Wrap-Up Comparison

| Approach | Ingestion style | Key ClickHouse feature | When to use |
|---|---|---|---|
| Bulk `INSERT ... SELECT` / table functions | Batch | `s3`/`gcs`/`file` table functions | Periodic loads, historical backfills, ETL |
| Kafka table engine + Materialized View | Streaming | `Kafka` engine, `system.kafka_consumers` | Continuous event streams (or use ClickPipes on Cloud) |
| Native/HTTP client batching | Either | `clickhouse-connect`, `clickhouse-client` | App-driven inserts where you control batch size |
| JDBC / ODBC | Either | Standard connectors | BI tools, JVM apps, legacy/Excel/Power BI integrations |
| Async inserts | Streaming (many small writers) | `async_insert`, `system.asynchronous_insert_log` | Many independent low-volume sources you can't batch client-side |

## 6. Cleanup

```sql
DROP DATABASE IF EXISTS ingestion_lab;
```

---

### Further Reading
- ClickHouse docs: "Selecting an insert strategy," "Bulk inserts," Kafka table engine, ClickPipes
- ClickHouse docs: JDBC driver, ODBC driver, native client libraries (`clickhouse-connect`, `clickhouse-go`, `clickhouse-driver`)
