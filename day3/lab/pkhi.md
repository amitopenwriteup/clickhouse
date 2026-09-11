चलिए इस पूरे ClickHouse lab को step-by-step समझते हैं, Hinglish में (Hindi words देवनागरी में, बाकी English terms as-is).

## Step 1: CREATE TABLE

```sql
CREATE TABLE events (...)
ENGINE = MergeTree
ORDER BY (tenant_id, event_time, user_id)
PRIMARY KEY (tenant_id, event_time)
```

यहाँ दो अलग-अलग चीज़ें define हो रही हैं:

- **ORDER BY (tenant_id, event_time, user_id)** — ये बताता है कि disk पे data physically किस क्रम में **sorted** रहेगा। ClickHouse हमेशा rows को इसी order में arrange करके store करता है, हर part के अंदर।
- **PRIMARY KEY (tenant_id, event_time)** — ये ORDER BY का एक **prefix (subset)** है, और सिर्फ इतने columns का ही एक छोटा सा **sparse index** RAM में रखा जाता है। पूरा `user_id` column index में नहीं जाता।
- अगर आप PRIMARY KEY बिल्कुल न लिखो, तो ClickHouse by default ORDER BY को ही primary key मान लेता है।

समझने वाली बात: primary key की length ORDER BY से कम या बराबर हो सकती है, ज़्यादा नहीं।

## Step 2: डेटा इंसर्ट करना (1,000,000 rows)

```sql
INSERT INTO events
SELECT
    (rand() % 50) + 1 AS tenant_id,
    toDateTime('2026-01-01 00:00:00') + toIntervalSecond(rand() % (...)) AS event_time,
    ['click','view','purchase','signup','logout'][(rand() % 5) + 1] AS event_type,
    rand64() % 5000000 AS user_id,
    hex(randomPrintableASCII(64)) AS payload
FROM numbers(1000000);
```

- **`numbers(1000000)`** — ये एक virtual table है जो 0 से 999999 तक integers generate करता है, बिना किसी real data source के। हर number = एक row का "seed"।
- **tenant_id** — `rand() % 50 + 1` से 1 से 50 के बीच random value आती है। यहाँ **low cardinality** जानबूझकर रखी गई है (सिर्फ 50 unique values), क्योंकि यही column primary key का पहला column है — कम unique values = अच्छा pruning candidate।
- **event_time** — दो dates के बीच का second-wise difference निकाल कर उसमें random offset add किया गया है, ताकि 8 महीने के range में data spread हो जाए।
- **user_id** — `rand64() % 5000000` से बहुत ज़्यादा **high cardinality** मिलती है, जानबूझकर, क्योंकि आगे यही column दिखाएगा कि non-prefix column पर filtering slow क्यों होती है।
- **index_granularity = 8192** का मतलब है हर 8192 sorted rows का एक "**granule**" बनता है, और primary index में हर granule के लिए सिर्फ एक entry (उस granule की पहली row का tenant_id + event_time) store होती है।
- 1,000,000 rows / 8192 ≈ **122 granules**, यानी सिर्फ 122 index entries RAM में — पूरे 10 लाख rows नहीं।

**Practical नोट:** एक ही बड़ी INSERT...SELECT statement अक्सर बहुत कम parts (शायद 1) बनाती है। अगर lab में दिखाए गए जैसे "Parts: 4/4" चाहिए, तो insert को 4 batches में तोड़ें (हर एक में 250,000 rows), और बीच में `OPTIMIZE TABLE ... FINAL` मत चलाना — नहीं तो parts merge होकर एक हो जाएंगे।

## Step 3: Query A — Prefix column पर filter (tenant_id)

```sql
SELECT count() FROM events WHERE tenant_id = 7;
```

- यहाँ **tenant_id** primary key का **पहला column** है, तो ClickHouse sparse index पर **binary search** कर सकता है।
- Result: सिर्फ **3 granules out of 122** पढ़े जाते हैं — बाकी 119 granules को disk से पढ़ा ही नहीं जाता (skip हो जाते हैं)।
- यही होता है असली "index pruning" — बहुत fast query, क्योंकि disk I/O बहुत कम हुआ।

## Step 4: Query B — Prefix + दूसरा column पर filter

```sql
SELECT count() FROM events 
WHERE tenant_id = 7 AND event_time >= '2026-06-01' AND event_time < '2026-07-01';
```

- यहाँ दोनों filtered columns (tenant_id, event_time) primary key का **लगातार prefix** हैं, सही order में।
- ClickHouse पहले tenant_id=7 वाला range निकालता है, फिर उस range के अंदर event_time वाला sub-range निकालता है — यानी **दो बार narrowing**।
- Result: सिर्फ **1 granule out of 122** पढ़ा जाता है — Query A से भी ज़्यादा fast, क्योंकि pruning maximum हुआ।

## Step 5: Query C — Non-prefix column पर filter (सिर्फ user_id)

```sql
SELECT count() FROM events WHERE user_id = 123456;
```

- **user_id** primary key में शामिल नहीं है (सिर्फ ORDER BY में है, और तीसरे नंबर पर, वो भी बिना tenant_id/event_time filter किए)।
- Sparse index sorted है tenant_id → event_time के हिसाब से, तो एक particular user_id वाली rows पूरे table में **scattered** पड़ी होंगी — किसी एक contiguous range में नहीं।
- इसलिए ClickHouse binary search नहीं कर सकता, और उसे **सारे 122 granules** पढ़ने पड़ते हैं (Granules: 122/122) — कोई pruning नहीं हुआ।
- फिर भी ये traditional RDBMS के row-by-row scan से तेज़ है (columnar storage + compression की वजह से), पर primary key का कोई फायदा नहीं मिला यहाँ।

## Step 6: Takeaways (मुख्य सीख)

1. **ORDER BY** = disk पर physical sort order (compulsory)।
2. **PRIMARY KEY** = ORDER BY का prefix, जो RAM के sparse index में रहता है (optional, default = ORDER BY)।
3. Primary key columns को **left-to-right prefix** के हिसाब से filter करने पर ही binary search + granule pruning मिलता है।
4. सिर्फ बाद वाले (non-prefix) columns पर filter करने से पूरा granule scan होता है — कोई pruning benefit नहीं।
5. **Design rule:** सबसे ज़्यादा frequently filter होने वाले, low-cardinality columns (जैसे tenant_id, date) को पहले रखो; high-cardinality columns (जैसे user_id) को बाद में।

Chahiye toh main ready-to-run script bhi bana sakta hoon जिसमें ये सारे INSERT + EXPLAIN queries एक साथ execute करने के लिए हों।
