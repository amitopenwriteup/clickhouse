# ClickHouse Data Types & Schema Design Workshop
## Practical Hands-on Cloud Lab Guide

**Duration:** 2 Hours  
**Format:** Live Cloud Lab with ClickHouse (ClickHouse Cloud or Self-Hosted)  
**Prerequisites:** Access to ClickHouse instance, basic SQL knowledge

---

## Workshop Overview

This workshop teaches you how to:
1. Understand all ClickHouse data types and when to use them
2. Design efficient schemas for analytical workloads
3. Choose optimal PRIMARY KEY for query performance
4. Implement effective PARTITIONING strategies
5. Practice with real-world examples in a cloud lab

---

## PART 1: DATA TYPES EXPLORATION (45 minutes)

### Setup: Create Workshop Database

**Lab 1.0: Create Workshop Database**

Connect to your ClickHouse instance:
```bash
clickhouse-client --host <host> --user <user> --password <password>
```

Create workshop database:
```sql
CREATE DATABASE IF NOT EXISTS workshop;
USE workshop;
```

---

### Lab 1.1: Numeric Data Types

**Objective:** Understand integer, float, and decimal types

**Create table with all numeric types:**

```sql
CREATE TABLE numeric_types_demo (
  id UInt8,
  counter UInt32,
  large_number UInt64,
  negative_value Int32,
  temperature Float32,
  precise_value Decimal(10, 2)
)
ENGINE = MergeTree
ORDER BY id;
```

**Insert sample data:**

```sql
INSERT INTO numeric_types_demo VALUES
  (1, 100, 1000000000, -50, 98.6, 123456.78),
  (2, 255, 2000000000, -100, 37.5, 99999.99),
  (3, 50, 500000000, 25, 0.1, 50000.01);
```

**Query and verify:**

```sql
SELECT 
  id,
  counter,
  large_number,
  negative_value,
  temperature,
  precise_value,
  toTypeName(id) as id_type,
  toTypeName(counter) as counter_type,
  toTypeName(temperature) as temp_type,
  toTypeName(precise_value) as decimal_type
FROM numeric_types_demo;
```

**Lab Exercises:**

1. Try inserting value 256 into UInt8 column - what happens?
   ```sql
   INSERT INTO numeric_types_demo VALUES (256, 100, 1000, -1, 98.6, 100.00);
   ```

2. Calculate storage size - which type uses least space?
   ```sql
   SELECT 
     sum(bytes_on_disk) as total_bytes,
     formatReadableSize(sum(bytes_on_disk)) as readable_size
   FROM system.parts
   WHERE table = 'numeric_types_demo' AND active;
   ```

---

### Lab 1.2: String Data Types

**Objective:** Compare String vs FixedString performance

**Create table with string types:**

```sql
CREATE TABLE string_types_demo (
  id UInt32,
  variable_text String,
  fixed_text FixedString(10),
  uuid_value UUID
)
ENGINE = MergeTree
ORDER BY id;
```

**Insert data:**

```sql
INSERT INTO string_types_demo VALUES
  (1, 'Hello World', 'Fixed1    ', generateUUIDv4()),
  (2, 'ClickHouse', 'Fixed2    ', generateUUIDv4()),
  (3, 'Database', 'Fixed3    ', generateUUIDv4()),
  (4, 'Analytics', 'Fixed4    ', generateUUIDv4()),
  (5, 'Performance', 'Fixed5    ', generateUUIDv4());
```

**Query data:**

```sql
SELECT 
  id,
  variable_text,
  fixed_text,
  uuid_value,
  length(variable_text) as var_len,
  length(fixed_text) as fixed_len
FROM string_types_demo;
```

**Lab Exercises:**

1. Measure storage difference:
   ```sql
   SELECT 
     'String' as type,
     sum(bytes_on_disk) as bytes
   FROM system.parts_columns
   WHERE table = 'string_types_demo' AND column = 'variable_text' AND active
   UNION ALL
   SELECT 
     'FixedString' as type,
     sum(bytes_on_disk) as bytes
   FROM system.parts_columns
   WHERE table = 'string_types_demo' AND column = 'fixed_text' AND active;
   ```

2. Query performance - filter by string:
   ```sql
   SELECT COUNT(*) FROM string_types_demo WHERE variable_text = 'Hello World';
   SELECT COUNT(*) FROM string_types_demo WHERE fixed_text = 'Fixed1    ';
   ```

---

### Lab 1.3: Date & Time Types

**Objective:** Master DateTime types for time-series data

**Create table with date/time types:**

```sql
CREATE TABLE datetime_types_demo (
  id UInt32,
  event_date Date,
  event_datetime DateTime,
  event_datetime64 DateTime64(3),
  event_time_utc DateTime('UTC'),
  event_time_ny DateTime('America/New_York')
)
ENGINE = MergeTree
ORDER BY event_datetime64;
```

**Insert data:**

```sql
INSERT INTO datetime_types_demo VALUES
  (1, '2024-01-15', '2024-01-15 10:30:45', '2024-01-15 10:30:45.123', '2024-01-15 10:30:45', '2024-01-15 10:30:45'),
  (2, '2024-01-16', '2024-01-16 14:20:10', '2024-01-16 14:20:10.456', '2024-01-16 14:20:10', '2024-01-16 14:20:10'),
  (3, '2024-01-17', '2024-01-17 18:45:30', '2024-01-17 18:45:30.789', '2024-01-17 18:45:30', '2024-01-17 18:45:30');
```

**Query data:**

```sql
SELECT 
  id,
  event_date,
  event_datetime,
  event_datetime64,
  event_time_utc,
  event_time_ny
FROM datetime_types_demo;
```

**Lab Exercises:**

1. Extract parts from DateTime:
   ```sql
   SELECT 
     id,
     toYear(event_datetime64) as year,
     toMonth(event_datetime64) as month,
     toDayOfMonth(event_datetime64) as day,
     toHour(event_datetime64) as hour,
     toMinute(event_datetime64) as minute
   FROM datetime_types_demo;
   ```

2. Time calculations:
   ```sql
   SELECT 
     id,
     event_datetime64,
     event_datetime64 + INTERVAL '1 DAY' as next_day,
     event_datetime64 - INTERVAL '1 HOUR' as prev_hour
   FROM datetime_types_demo;
   ```

3. Group by time:
   ```sql
   SELECT 
     toYYYYMM(event_datetime64) as year_month,
     COUNT(*) as count
   FROM datetime_types_demo
   GROUP BY toYYYYMM(event_datetime64);
   ```

---

### Lab 1.4: Special Types (Enum, LowCardinality, JSON)

**Objective:** Use specialized types for efficient storage

**Create table with special types:**

```sql
CREATE TABLE special_types_demo (
  id UInt32,
  status Enum8('pending' = 1, 'active' = 2, 'completed' = 3, 'cancelled' = 4),
  country_code LowCardinality(String),
  json_data JSON
)
ENGINE = MergeTree
ORDER BY id;
```

**Insert data:**

```sql
INSERT INTO special_types_demo VALUES
  (1, 'active', 'US', '{"name": "John", "age": 30, "city": "NYC"}'),
  (2, 'pending', 'UK', '{"name": "Jane", "age": 25, "city": "London"}'),
  (3, 'completed', 'FR', '{"name": "Pierre", "age": 35, "city": "Paris"}'),
  (4, 'active', 'DE', '{"name": "Hans", "age": 28, "city": "Berlin"}'),
  (5, 'cancelled', 'US', '{"name": "Bob", "age": 40, "city": "LA"}');
```

**Query data:**

```sql
SELECT 
  id,
  status,
  country_code,
  json_data
FROM special_types_demo;
```

**Lab Exercises:**

1. Filter by Enum:
   ```sql
   SELECT COUNT(*) as active_count 
   FROM special_types_demo 
   WHERE status = 'active';
   ```

2. Group by LowCardinality:
   ```sql
   SELECT 
     country_code,
     COUNT(*) as count
   FROM special_types_demo
   GROUP BY country_code
   ORDER BY count DESC;
   ```

3. Extract JSON fields:
   ```sql
   SELECT 
     id,
     json_data.name as name,
     json_data.age as age,
     json_data.city as city
   FROM special_types_demo;
   ```

---

### Lab 1.5: Complex Types (Array, Tuple, Map, Nested)

**Objective:** Handle complex data structures

**Create table with complex types:**

```sql
CREATE TABLE complex_types_demo (
  id UInt32,
  tags Array(String),
  coordinates Tuple(Float64, Float64),
  attributes Map(String, String),
  events Nested(
    event_type String,
    timestamp DateTime,
    value Float32
  )
)
ENGINE = MergeTree
ORDER BY id;
```

**Insert data:**

```sql
INSERT INTO complex_types_demo VALUES
  (1, ['python', 'analytics', 'fast'], (40.7128, -74.0060), {'color': 'blue', 'size': 'large'}, (['click', 'view'], [now(), now()], [100.5, 200.3])),
  (2, ['sql', 'database'], (51.5074, -0.1278), {'color': 'red', 'size': 'medium'}, (['login', 'logout'], [now(), now()], [50.0, 75.5])),
  (3, ['javascript', 'frontend', 'web'], (48.8566, 2.3522), {'color': 'green', 'size': 'small'}, (['purchase', 'review'], [now(), now()], [150.75, 200.0]));
```

**Query data:**

```sql
SELECT 
  id,
  tags,
  coordinates,
  attributes,
  events
FROM complex_types_demo;
```

**Lab Exercises:**

1. Array operations:
   ```sql
   SELECT 
     id,
     arrayLength(tags) as tag_count,
     tags[1] as first_tag,
     arrayConcat(tags, ['new_tag']) as tags_with_new
   FROM complex_types_demo;
   ```

2. Tuple extraction:
   ```sql
   SELECT 
     id,
     coordinates.1 as latitude,
     coordinates.2 as longitude,
     sqrt(pow(coordinates.1, 2) + pow(coordinates.2, 2)) as distance
   FROM complex_types_demo;
   ```

3. Map operations:
   ```sql
   SELECT 
     id,
     attributes['color'] as color,
     attributes['size'] as size,
     mapKeys(attributes) as attribute_keys,
     mapValues(attributes) as attribute_values
   FROM complex_types_demo;
   ```

4. Nested data - flatten:
   ```sql
   SELECT 
     id,
     events.event_type,
     events.timestamp,
     events.value
   FROM complex_types_demo
   ARRAY JOIN events;
   ```

---

## PART 2: SCHEMA DESIGN (50 minutes)

### Lab 2.1: Query-Driven Schema Design

**Objective:** Design schema based on actual query patterns

**Scenario:** E-commerce analytics system

**Define query patterns FIRST:**
```
1. Query: "Events by hour for last 30 days"
   Filter: timestamp >= now() - INTERVAL '30 DAY'
   Group: toStartOfHour(timestamp)

2. Query: "Top products by region"
   Filter: product_category = 'electronics'
   Group: region, product_id

3. Query: "User engagement metrics"
   Filter: user_country = 'US'
   Join: user_id with dimension tables
```

**Design Schema Based on Queries:**

```sql
CREATE TABLE events (
  timestamp DateTime64(3),
  event_id UUID,
  user_id UInt64,
  product_id UInt32,
  region LowCardinality(String),
  product_category LowCardinality(String),
  product_name String,
  quantity UInt16,
  amount Decimal(12, 2),
  user_country LowCardinality(String),
  user_age UInt8,
  device_type LowCardinality(String),
  is_purchase Boolean
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp, product_id, user_id)
PARTITION BY toYYYYMM(timestamp);
```

**Insert sample data:**

```sql
INSERT INTO events SELECT
  now() - INTERVAL (number % 100000) SECOND as timestamp,
  generateUUIDv4() as event_id,
  (number % 10000) as user_id,
  (number % 500) + 1 as product_id,
  ['US', 'EU', 'APAC'][1 + (number % 3)] as region,
  ['electronics', 'clothing', 'home'][1 + (number % 3)] as product_category,
  'Product_' || toString((number % 500) + 1) as product_name,
  (number % 100) + 1 as quantity,
  (number % 10000) / 100.0 as amount,
  ['US', 'UK', 'DE', 'FR', 'JP'][1 + (number % 5)] as user_country,
  (number % 70) + 18 as user_age,
  ['web', 'mobile', 'tablet'][1 + (number % 3)] as device_type,
  (number % 2) = 0 as is_purchase
FROM numbers(100000);
```

**Run designed queries:**

```sql
-- Query 1: Events by hour for last 30 days
SELECT 
  toStartOfHour(timestamp) as hour,
  COUNT(*) as event_count,
  SUM(amount) as total_amount
FROM events
WHERE timestamp >= now() - INTERVAL '30 DAY'
GROUP BY toStartOfHour(timestamp)
ORDER BY hour DESC
LIMIT 10;

-- Query 2: Top products by region
SELECT 
  region,
  product_id,
  product_name,
  COUNT(*) as event_count,
  SUM(amount) as total_sales
FROM events
WHERE product_category = 'electronics'
GROUP BY region, product_id, product_name
ORDER BY total_sales DESC
LIMIT 10;

-- Query 3: User engagement metrics
SELECT 
  user_country,
  COUNT(DISTINCT user_id) as unique_users,
  COUNT(*) as total_events,
  SUM(is_purchase) as purchases,
  SUM(amount) as total_spent,
  ROUND(SUM(is_purchase) * 100.0 / COUNT(*), 2) as purchase_rate
FROM events
WHERE user_country = 'US'
GROUP BY user_country;
```

---

### Lab 2.2: PRIMARY KEY Design

**Objective:** Understand PRIMARY KEY impact on query performance

**Create two tables - good and poor PRIMARY KEY:**

```sql
-- GOOD PRIMARY KEY (for time-range + region queries)
CREATE TABLE events_good (
  timestamp DateTime64(3),
  region LowCardinality(String),
  product_id UInt32,
  user_id UInt64,
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp, product_id)
PARTITION BY toYYYYMM(timestamp);

-- POOR PRIMARY KEY (high cardinality first)
CREATE TABLE events_poor (
  timestamp DateTime64(3),
  region LowCardinality(String),
  product_id UInt32,
  user_id UInt64,
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (user_id, product_id)
ORDER BY (user_id, product_id, timestamp)
PARTITION BY toYYYYMM(timestamp);
```

**Insert same data into both:**

```sql
INSERT INTO events_good SELECT
  now() - INTERVAL (number % 100000) SECOND,
  ['US', 'EU', 'APAC'][1 + (number % 3)],
  (number % 500) + 1,
  (number % 10000),
  (number % 10000) / 100.0
FROM numbers(100000);

INSERT INTO events_poor SELECT * FROM events_good;
```

**Compare query performance:**

```sql
-- Query with good PRIMARY KEY
EXPLAIN indexes = 1
SELECT SUM(amount) 
FROM events_good
WHERE region = 'US' AND timestamp >= now() - INTERVAL '1 DAY';

-- Query with poor PRIMARY KEY
EXPLAIN indexes = 1
SELECT SUM(amount) 
FROM events_poor
WHERE region = 'US' AND timestamp >= now() - INTERVAL '1 DAY';
```

**Analyze query logs:**

```sql
SELECT 
  query,
  query_duration_ms,
  read_rows,
  read_bytes,
  formatReadableSize(read_bytes) as readable_read_bytes
FROM system.query_log
WHERE query_kind = 'Select' 
  AND query LIKE '%events_good%' OR query LIKE '%events_poor%'
ORDER BY event_time DESC
LIMIT 10;
```

---

### Lab 2.3: PARTITIONING Strategy

**Objective:** Design partitions for efficient data pruning

**Create tables with different partitioning strategies:**

```sql
-- Monthly Partitioning (Most Common)
CREATE TABLE events_monthly (
  timestamp DateTime64(3),
  region String,
  user_id UInt64,
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp)
PARTITION BY toYYYYMM(timestamp);

-- Daily Partitioning (High Volume)
CREATE TABLE events_daily (
  timestamp DateTime64(3),
  region String,
  user_id UInt64,
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp)
PARTITION BY toYYYYMMDD(timestamp);

-- No Partitioning (Small Tables)
CREATE TABLE events_no_partition (
  timestamp DateTime64(3),
  region String,
  user_id UInt64,
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp);
```

**Insert data:**

```sql
INSERT INTO events_monthly SELECT * FROM events;
INSERT INTO events_daily SELECT * FROM events;
INSERT INTO events_no_partition SELECT * FROM events;
```

**Check partition info:**

```sql
-- View monthly partitions
SELECT 
  partition,
  COUNT(*) as rows,
  formatReadableSize(sum(bytes_on_disk)) as size
FROM system.parts
WHERE table = 'events_monthly' AND active
GROUP BY partition
ORDER BY partition;

-- View daily partitions  
SELECT 
  partition,
  COUNT(*) as rows,
  formatReadableSize(sum(bytes_on_disk)) as size
FROM system.parts
WHERE table = 'events_daily' AND active
GROUP BY partition
ORDER BY partition DESC
LIMIT 20;
```

**Query performance comparison:**

```sql
-- Monthly partition query
SELECT SUM(amount)
FROM events_monthly
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-02-01';

-- Daily partition query
SELECT SUM(amount)
FROM events_daily
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-02-01';

-- No partition query
SELECT SUM(amount)
FROM events_no_partition
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-02-01';
```

---

### Lab 2.4: Data Type Optimization

**Objective:** Optimize data types for storage and performance

**Create table with non-optimized types:**

```sql
CREATE TABLE events_non_optimized (
  timestamp String,
  region String,
  product_id String,
  user_id String,
  category String,
  device String,
  amount String
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp)
PARTITION BY substring(timestamp, 1, 7);
```

**Create optimized version:**

```sql
CREATE TABLE events_optimized (
  timestamp DateTime64(3),
  region LowCardinality(String),
  product_id UInt32,
  user_id UInt64,
  category LowCardinality(String),
  device LowCardinality(String),
  amount Decimal(12, 2)
)
ENGINE = MergeTree
PRIMARY KEY (region, timestamp)
ORDER BY (region, timestamp)
PARTITION BY toYYYYMM(timestamp);
```

**Insert data into both:**

```sql
-- Non-optimized
INSERT INTO events_non_optimized SELECT
  toString(now() - INTERVAL (number % 100000) SECOND),
  ['US', 'EU', 'APAC'][1 + (number % 3)],
  toString((number % 500) + 1),
  toString((number % 10000)),
  ['electronics', 'clothing'][1 + (number % 2)],
  ['web', 'mobile'][1 + (number % 2)],
  toString((number % 10000) / 100.0)
FROM numbers(100000);

-- Optimized
INSERT INTO events_optimized SELECT
  now() - INTERVAL (number % 100000) SECOND,
  ['US', 'EU', 'APAC'][1 + (number % 3)],
  (number % 500) + 1,
  (number % 10000),
  ['electronics', 'clothing'][1 + (number % 2)],
  ['web', 'mobile'][1 + (number % 2)],
  (number % 10000) / 100.0
FROM numbers(100000);
```

**Compare storage size:**

```sql
SELECT 
  'non_optimized' as table_name,
  formatReadableSize(sum(bytes_on_disk)) as total_size,
  COUNT(*) as parts,
  ROUND(sum(bytes_on_disk) / 100000, 2) as bytes_per_row
FROM system.parts
WHERE table = 'events_non_optimized' AND active
UNION ALL
SELECT 
  'optimized' as table_name,
  formatReadableSize(sum(bytes_on_disk)) as total_size,
  COUNT(*) as parts,
  ROUND(sum(bytes_on_disk) / 100000, 2) as bytes_per_row
FROM system.parts
WHERE table = 'events_optimized' AND active;
```

**Compare compression:**

```sql
SELECT 
  'non_optimized' as table_name,
  SUM(data_uncompressed_bytes) as uncompressed,
  SUM(data_compressed_bytes) as compressed,
  formatReadableSize(SUM(data_uncompressed_bytes)) as uncompressed_readable,
  formatReadableSize(SUM(data_compressed_bytes)) as compressed_readable,
  ROUND(SUM(data_uncompressed_bytes) / SUM(data_compressed_bytes), 2) as compression_ratio
FROM system.parts_columns
WHERE table = 'events_non_optimized' AND active
GROUP BY table_name
UNION ALL
SELECT 
  'optimized' as table_name,
  SUM(data_uncompressed_bytes) as uncompressed,
  SUM(data_compressed_bytes) as compressed,
  formatReadableSize(SUM(data_uncompressed_bytes)) as uncompressed_readable,
  formatReadableSize(SUM(data_compressed_bytes)) as compressed_readable,
  ROUND(SUM(data_uncompressed_bytes) / SUM(data_compressed_bytes), 2) as compression_ratio
FROM system.parts_columns
WHERE table = 'events_optimized' AND active
GROUP BY table_name;
```

---

### Lab 2.5: Complete Real-World Schema

**Objective:** Design production-grade schema for observability (logs/metrics)

**Create logs schema:**

```sql
CREATE TABLE logs (
  timestamp DateTime64(3),
  service_name LowCardinality(String),
  environment LowCardinality(String),
  level LowCardinality(String),
  message String,
  trace_id UUID,
  user_id Nullable(UInt64),
  duration_ms UInt32,
  status_code UInt16,
  attributes Map(String, String)
)
ENGINE = MergeTree
PRIMARY KEY (service_name, timestamp)
ORDER BY (service_name, timestamp, level, duration_ms)
PARTITION BY toYYYYMMDD(timestamp);
```

**Create metrics schema:**

```sql
CREATE TABLE metrics (
  timestamp DateTime64(3),
  metric_name LowCardinality(String),
  service_name LowCardinality(String),
  instance_id LowCardinality(String),
  value Float32,
  tags Map(String, String)
)
ENGINE = SummingMergeTree()
PRIMARY KEY (metric_name, service_name, timestamp)
ORDER BY (metric_name, service_name, timestamp, instance_id)
PARTITION BY toYYYYMMDD(timestamp);
```

**Insert sample log data:**

```sql
INSERT INTO logs SELECT
  now() - INTERVAL (number % 10000) SECOND as timestamp,
  ['api', 'worker', 'database'][1 + (number % 3)] as service_name,
  ['prod', 'staging'][1 + (number % 2)] as environment,
  ['INFO', 'WARN', 'ERROR'][1 + (number % 3)] as level,
  'Log message ' || toString(number) as message,
  generateUUIDv4() as trace_id,
  if(number % 5 = 0, (number % 10000), NULL) as user_id,
  (number % 1000) + 50 as duration_ms,
  [200, 201, 400, 500][1 + (number % 4)] as status_code,
  {'request_id': toString(number), 'method': 'GET', 'path': '/api/v1'} as attributes
FROM numbers(50000);
```

**Run analytical queries:**

```sql
-- Logs by level
SELECT 
  level,
  COUNT(*) as count,
  AVG(duration_ms) as avg_duration,
  MAX(duration_ms) as max_duration
FROM logs
GROUP BY level
ORDER BY count DESC;

-- Errors by service
SELECT 
  service_name,
  COUNT(*) as error_count,
  COUNT(DISTINCT user_id) as affected_users
FROM logs
WHERE level = 'ERROR'
GROUP BY service_name
ORDER BY error_count DESC;

-- Performance by service
SELECT 
  service_name,
  COUNT(*) as request_count,
  AVG(duration_ms) as avg_response_time,
  quantile(0.95)(duration_ms) as p95_response_time,
  quantile(0.99)(duration_ms) as p99_response_time,
  SUM(level = 'ERROR') as error_count,
  ROUND(SUM(level = 'ERROR') * 100.0 / COUNT(*), 2) as error_rate
FROM logs
WHERE timestamp >= now() - INTERVAL '1 DAY'
GROUP BY service_name
ORDER BY error_rate DESC;
```

---

## PART 3: VALIDATION & OPTIMIZATION (25 minutes)

### Lab 3.1: Query Performance Analysis

**Check if queries use PRIMARY KEY:**

```sql
EXPLAIN indexes = 1
SELECT SUM(amount)
FROM events
WHERE region = 'US' AND timestamp >= now() - INTERVAL '7 DAY';
```

**Analyze query log:**

```sql
SELECT 
  query,
  query_duration_ms,
  read_rows,
  formatReadableSize(read_bytes) as read_bytes,
  result_rows,
  formatReadableSize(result_bytes) as result_bytes,
  ROUND(read_bytes / (read_rows + 1), 2) as avg_bytes_per_row
FROM system.query_log
WHERE query_kind = 'Select' 
  AND event_time > now() - INTERVAL '10 MINUTE'
ORDER BY query_duration_ms DESC
LIMIT 10;
```

---

### Lab 3.2: Compression Codec Analysis

**Check compression by column:**

```sql
SELECT 
  column_name,
  formatReadableSize(data_uncompressed_bytes) as uncompressed,
  formatReadableSize(data_compressed_bytes) as compressed,
  ROUND(data_uncompressed_bytes / (data_compressed_bytes + 1), 2) as ratio
FROM system.parts_columns
WHERE table = 'events' AND active
ORDER BY data_compressed_bytes DESC;
```

**Check partition sizes:**

```sql
SELECT 
  partition,
  COUNT(*) as parts,
  SUM(rows) as rows,
  formatReadableSize(SUM(bytes_on_disk)) as size
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY partition DESC;
```

---

### Lab 3.3: System Tables for Monitoring

**Active tables:**

```sql
SELECT 
  database,
  table,
  formatReadableSize(total_bytes) as size,
  total_rows as row_count
FROM system.tables
WHERE database = 'workshop'
ORDER BY total_bytes DESC;
```

**MergeTree settings:**

```sql
SELECT *
FROM system.settings
WHERE name LIKE 'max_%'
LIMIT 10;
```

---

## WORKSHOP SUMMARY

### Key Takeaways

1. **Data Types Matter**
   - Choose smallest type that fits (UInt8 vs UInt64)
   - Use LowCardinality for low-variety strings
   - Use DateTime64 for millisecond precision
   - Avoid Nullable when possible

2. **Schema Design is Query-Driven**
   - Understand query patterns FIRST
   - Denormalize for analytics
   - Combine related data into wide tables
   - Design for common queries

3. **PRIMARY KEY is Critical**
   - Order by cardinality: LOW to HIGH
   - Include frequently filtered columns
   - Include timestamp for time-range queries
   - Impacts both query speed and index size

4. **PARTITIONING Enables Pruning**
   - Use time-based partitions (most common)
   - PARTITION BY doesn't have to be in PRIMARY KEY
   - Pruning reduces data scanned
   - Balance partition size (avoid too many/few)

5. **Storage Optimization**
   - Good data types reduce disk usage by 10-50x
   - Compression ratios improve with denormalization
   - Wide tables compress better than many small columns
   - Monitor with system.parts_columns

---

## OFFICIAL DOCUMENTATION REFERENCES

- **Data Types:** https://clickhouse.com/docs/en/sql-reference/data-types/
- **Schema Design:** https://clickhouse.com/docs/en/guides/best-practices/schema-design-best-practices/
- **Primary Key:** https://clickhouse.com/docs/en/guides/improving-query-performance/primary-key-guide/
- **Partitioning:** https://clickhouse.com/docs/en/guides/improving-query-performance/partitioning/
- **CREATE TABLE:** https://clickhouse.com/docs/en/sql-reference/statements/create/table/

---

## HOMEWORK EXERCISES

### Exercise 1: Design Schema for Your Use Case
Create a schema for:
- Mobile app analytics (events, users, sessions)
- Include appropriate data types
- Define PRIMARY KEY and PARTITION BY
- Explain your design decisions

### Exercise 2: Performance Comparison
Create two schemas - optimized and non-optimized
- Compare storage size
- Compare query performance
- Analyze compression ratios
- Document findings

### Exercise 3: Real Data Import
Import your own CSV/JSON data
- Map columns to correct data types
- Design appropriate schema
- Validate query performance
- Optimize based on queries

---

**End of Workshop**

Good luck with your ClickHouse journey! 🚀
