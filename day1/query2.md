Yes. With your table:

```sql
ORDER BY (service, event_time)
```

the **primary/sorting key starts with `service` and then `event_time`**.

So the best basic queries are those that filter on `service`, especially `service + event_time`.

### 1. Find logs for a particular service

```sql
SELECT *
FROM logs
WHERE service = 'payment';
```

This can use the primary key because `service` is the **first column** in:

```text
(service, event_time)
```

---

### 2. Find logs for a service + time range

This is even better:

```sql
SELECT *
FROM logs
WHERE service = 'payment'
  AND event_time >= now() - INTERVAL 1 HOUR;
```

ClickHouse can narrow the data using:

```text
service → payment
        ↓
event_time → last 1 hour
```

---

### 3. Find ERROR logs for a service

```sql
SELECT *
FROM logs
WHERE service = 'payment'
  AND level = 'ERROR';
```

**Important:** `level` is not part of your primary key.

So ClickHouse can first use:

```text
service = 'payment'
```

and then scan/filter the relevant data for:

```text
level = 'ERROR'
```

---

### 4. Find HTTP 500 logs for a service

```sql
SELECT *
FROM logs
WHERE service = 'payment'
  AND status_code = 500;
```

Again:

```text
service       → primary-key filtering
status_code   → additional filtering
```

---

### 5. Find ERROR + 500 logs

```sql
SELECT *
FROM logs
WHERE service = 'payment'
  AND level = 'ERROR'
  AND status_code = 500;
```

---

### 6. Count logs by service

```sql
SELECT
    service,
    count() AS total
FROM logs
GROUP BY service
ORDER BY total DESC;
```

Example:

```text
payment    200123
auth       199876
search     200456
cart       199700
checkout   199845
```

---

### 7. Count ERROR logs by service

```sql
SELECT
    service,
    count() AS errors
FROM logs
WHERE level = 'ERROR'
GROUP BY service
ORDER BY errors DESC;
```

Here `level` isn't in the primary key, so ClickHouse cannot use the primary key to directly locate `ERROR`. It will filter the relevant data using other mechanisms.

---

### 8. Find all status codes for a service

```sql
SELECT
    status_code,
    count() AS count
FROM logs
WHERE service = 'payment'
GROUP BY status_code
ORDER BY count DESC;
```

---

### 9. Find slow requests

```sql
SELECT *
FROM logs
WHERE service = 'payment'
  AND latency_ms > 400
ORDER BY latency_ms DESC
LIMIT 20;
```

The primary key helps locate `payment`; `latency_ms` is then filtered.

---

## The key concept to remember

Your:

```sql
ORDER BY (service, event_time)
```

is effectively telling ClickHouse:

```text
PRIMARY KEY / SORTING KEY

service
   ↓
event_time
```

So queries like:

```sql
WHERE service = 'payment'
```

or:

```sql
WHERE service = 'payment'
AND event_time > ...
```

are **primary-key friendly**.

But:

```sql
WHERE level = 'ERROR'
```

or:

```sql
WHERE status_code = 500
```

are **not primary-key friendly**, because neither column appears in the beginning of the sorting key.

### One very useful experiment

Run:

```sql
EXPLAIN indexes = 1
SELECT *
FROM logs
WHERE service = 'payment'
  AND event_time >= now() - INTERVAL 1 HOUR;
```

Then compare it with:

```sql
EXPLAIN indexes = 1
SELECT *
FROM logs
WHERE level = 'ERROR';
```

You'll be able to **see the difference in how the primary key helps ClickHouse skip data**.
