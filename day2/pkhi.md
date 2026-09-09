Chalo, is doc ko Hinglish mein samjhate hain — ClickHouse ke primary key ka concept MySQL/Postgres se बिल्कुल अलग है।

## Table Setup

```sql
CREATE TABLE logs (...)
ENGINE = MergeTree()
PARTITION BY toDate(event_time)
ORDER BY (service, event_time);
```

यहाँ `PRIMARY KEY` explicitly नहीं दिया, तो वो automatically `ORDER BY (service, event_time)` बन जाता है।

## 1. Primary Key करता क्या है

Traditional RDBMS (MySQL/Postgres) में primary key का मतलब होता है **uniqueness** — duplicate rows allowed नहीं होते, और एक B-tree बनता है fast lookup के लिए।

ClickHouse में बिल्कुल different सोच है:
- **Uniqueness enforce नहीं होती** — same `(service, event_time)` pair multiple बार आ सकता है। Logs data के लिए ये perfectly fine है (एक ही second में multiple log entries हो सकती हैं)।
- Primary key basically दो काम करता है: **data को disk पे sort करके रखना**, और एक **sparse index** बनाना।
- "Sparse" का मतलब — हर row के लिए index entry नहीं बनती, बल्कि हर 8,192 rows (एक "granule") के लिए एक entry बनती है।

## 2. Data Physically Store कैसे होता है

- पहले `PARTITION BY toDate(event_time)` के हिसाब से data **daily partitions** में divide होता है (मतलब हर दिन का data अलग partition में)।
- फिर हर partition के अंदर, rows को `service` के हिसाब से group करके sort किया जाता है, और **हर service के अंदर** `event_time` के हिसाब से sort होता है।

मतलब data कुछ ऐसा दिखता है: पहले सारे `auth-api` के rows time-order में, फिर सारे `checkout-api` के rows time-order में, वैसे ही आगे।

## 3. Sparse Index काम कैसे करता है

Index में सिर्फ हर granule के **पहले row की key value** store होती है (जैसे mark 0 पे `auth-api, 00:01:02`)। जब query आती है, ClickHouse इस छोटे से index पे **binary search** करता है ये पता करने के लिए कि कौनसे granules में match हो सकता है — और सिर्फ वही disk से पढ़ता है।

## 4. Query Performance Patterns

### ✅ Fast — primary key efficiently use होती है

```sql
SELECT * FROM logs WHERE service = 'checkout-api';

SELECT * FROM logs
WHERE service = 'checkout-api'
  AND event_time >= now() - INTERVAL 1 HOUR;
```
Yahan ClickHouse seedha `checkout-api` वाले block पे jump kar jaata hai, फिर उसके अंदर time के हिसाब से binary search karta hai.

### ⚠️ Partial benefit

```sql
SELECT * FROM logs
WHERE event_time >= '2026-09-08 00:00:00'
  AND event_time <  '2026-09-09 00:00:00';
```
Date partition key में hai isliye पूरे din skip ho jaate hain, लेकिन एक din ke andar sab services ke across scan करना padta hai.

### ❌ Slow — index use नहीं हो पाती

```sql
SELECT * FROM logs WHERE event_time >= now() - INTERVAL 1 HOUR;

SELECT * FROM logs WHERE status_code >= 500;
SELECT * FROM logs WHERE level = 'ERROR';
```
`event_time` key का दूसरा column है, तो बिना `service` filter के binary search नहीं हो सकता। और `status_code`/`level` key में हैं ही नहीं — पूरे granule scan करने पड़ते हैं (सिर्फ date partition pruning ही help करता है).

## 5. क्या यही Key Sahi Choice Hai?

Ye depend karta hai aapke typical query pattern pe:

| Aapki typical query | Best key choice |
|---|---|
| "Service X ke logs, time range Y mein" | `(service, event_time)` ← current setup ✅ |
| "Sabhi services ke recent errors" | `(event_time)` ya `(toStartOfHour(event_time), service)` |
| "Service X ke errors, recent pehle" | `(service, level, event_time)` |

Agar aap aksar `status_code` ya `level` pe filter karte ho `service` ke bina, toh primary key change karne ke bajaay **secondary skip indexes** add karo:

```sql
ALTER TABLE logs ADD INDEX idx_status status_code TYPE minmax GRANULARITY 4;
ALTER TABLE logs ADD INDEX idx_level  level       TYPE set(10) GRANULARITY 4;
```

## 6. Summary

- **Primary key = sort order + sparse index**, uniqueness constraint नहीं।
- Column का **order matters** — सिर्फ left-to-right *prefix* hi efficiently use हो sakta है `WHERE` filters mein।
- `PARTITION BY` पूरे din prune karta hai, primary key फिर उस din ke andar granules prune karta hai।
- Is table ke liye, `service` se scope की गई queries cheap हैं; सिर्फ `event_time` या दूसरे columns se scope की गई queries expensive हैं — apne actual query pattern ke hisaab se key order choose karo।
