# ClickHouse Rows को Columns से वापस कैसे बनाता है

## सबसे बड़ा सवाल

ClickHouse डेटा को **column-by-column** स्टोर करता है डिस्क पर, लेकिन जब query करो तो result **rows** में मिलता है। तो ये दोनों चीज़ें कैसे जुड़ जाती हैं — बिना columnar storage का फायदा खोए?

**छोटा जवाब:** Rows कभी भी individually store या fetch नहीं होते। वो सिर्फ **सबसे last step** में बनाए जाते हैं, और वो भी fast bulk (vectorized) operations से — एक-एक row करके नहीं।

---

## 1. Storage Layout - Columns, Not Rows

इस table के लिए:

```sql
CREATE TABLE logs (
  event_time DateTime,
  service String,
  level String,
  status_code UInt16,
  latency_ms UInt32,
  message String
) ENGINE = MergeTree ORDER BY event_time;
```

हर column अपनी **अलग file** में स्टोर होता है:

```
event_time.bin
service.bin
level.bin
status_code.bin
latency_ms.bin
message.bin
```

हर column file में values **same row order** में होती हैं। डिस्क पर कोई "row" structure नहीं होता — सिर्फ **position (index)** ही बताता है कि कौनसी value किस row की है across सारी files।

```
Position:        0        1        2        3
service.bin:     payment  payment  orders   payment
latency_ms.bin:  52       152      89       252
```

Position 3, `service.bin` में और position 3, `latency_ms.bin` में — दोनों **same row** के हैं, सिर्फ इसलिए क्योंकि इनका index same है। कोई joins नहीं, कोई pointers नहीं।

---

## 2. Granules और Marks - Sparse Index

Column files को एक बड़े stream की तरह नहीं पढ़ा जाता। डेटा **granules** में divide होता है (default 8,192 rows per granule)। एक अलग **marks file** (`.mrk`) में लिखा होता है कि हर granule कौनसे byte से start होता है compressed column file में।

इससे दो फायदे होते हैं:
- वो **पूरी granules skip** हो जाती हैं जो query के `WHERE` clause से match नहीं करतीं (primary key के sparse index की मदद से)
- ClickHouse सीधा **सही byte offset** पर jump कर सकता है, file की शुरुआत से scan नहीं करना पड़ता

---

## 3. Query Execution - एक-एक Row नहीं, Bulk में

जब आप ये query चलाते हो:

```sql
SELECT * FROM logs WHERE service = 'payment';
```

Actual sequence ये होती है:

### Step 1 — Filter वाले column की पूरी granule पढ़ो
ClickHouse `service` column की **पूरी granule** (जैसे 8,192 values) एक ही बार में sequentially पढ़ता है — कुछ selective positions नहीं, सब कुछ।

### Step 2 — पूरे array पर एक साथ predicate check करो
वो `service == 'payment'` को सारी 8,192 values पर एक ही bulk (vectorized/SIMD) pass में compare करता है, और एक **filter mask** बनाता है — matching positions की list:

```
matching positions: [0, 1, 3, 4, 7, ...]
```

ये एक bulk operation है, row-by-row check/loop नहीं।

### Step 3 — बाकी selected columns की granule भी पूरी पढ़ो, और वही mask लगाओ
`latency_ms`, `event_time`, `message` वगैरह के लिए ClickHouse:
1. पूरी granule sequentially पढ़ता है (I/O सस्ता होता है)
2. Step 2 वाला **वही mask** apply करता है सिर्फ matching values निकालने के लिए — एक fast bulk filter-copy operation से (`IColumn::filter()`), individual lookups से नहीं

क्योंकि हर column का row order same है, **वही mask** सब columns पर बिल्कुल वैसे ही काम करता है।

---

## 4. Blocks - In-Memory Unit

Filter हुई column arrays एक **`Block`** में group होती हैं — ये ClickHouse का अंदर का fundamental working unit है:

```
Block {
  IColumn(event_time)   →  [values...]
  IColumn(service)      →  [values...]
  IColumn(latency_ms)   →  [values...]
  ...
}
```

आगे के सारे operations — aggregation, sorting, और filtering — इस columnar `Block` पर होते हैं। ClickHouse कभी भी अंदर से row objects नहीं बनाता जैसे `{event_time: ..., service: ...}` processing के दौरान।

---

## 5. Row Reconstruction - सिर्फ Output Time पर

**सिर्फ एक जगह** rows actually बनती हैं — final **output formatter** में (जैसे `Pretty`, `JSON`, `CSV`, या Native protocol जो clients/UIs use करते हैं)।

Formatter position `i` को `0` से लेकर block की row count तक चलता है, और हर `i` के लिए पढ़ता है:

```
column[0][i], column[1][i], column[2][i], ...
```

सारे selected columns से — और उसको एक output row बना कर देता है। ये **already-filtered, छोटी हो चुकी** result set पर होता है, पूरी original granule पर नहीं।

---

## पूरा Execution Chain (Summary)

```
Disk: अलग column files, row position के हिसाब से aligned, granules में divide
   ↓
Sparse index / marks फिज़ूल granules को prune करते हैं
   ↓
ज़रूरी granules पढ़ो और decompress करो, हर column के लिए (sequential I/O)
   ↓
IColumn arrays में load करो → Block में group करो
   ↓
Vectorized predicate check → filter mask बनाओ
   ↓
Mask को सब columns पर uniformly apply करो (bulk filter-copy)
   ↓
(Aggregation / sorting / etc. — अब भी columnar ही है)
   ↓
Output formatter position i को columns पर चलाता है → row i emit करता है
```

---

## ये इतना Fast क्यों है

| Aspect | Row-based DB | ClickHouse (columnar) |
|---|---|---|
| I/O per query | पूरी row पढ़ता है (सारे columns) चाहे 2 ही चाहिए हो | सिर्फ जो columns select किए हैं वही पढ़ता है |
| Filtering | Row-by-row branching/checks | पूरे array पर एक साथ bulk comparison |
| Row assembly | Rows already disk पर बनी होती हैं | सिर्फ output time पर बनती हैं, वो भी already-filtered छोटे set पर |
| Access pattern | Random seeks हो सकते हैं | Granules के अंदर हमेशा sequential reads |

## सबसे important बात (याद रखने वाली)

> **Columnar storage optimize करता है कि क्या पढ़ा जाए और filtering कैसे compute हो; row output तो बस एक सस्ता, आख़िरी "transpose" step है — जो एक बहुत छोटे, already-reduce हो चुके dataset पर होता है।**

इसी वजह से ClickHouse लाखों rows को milliseconds में scan कर लेता है — जो महंगा काम है (I/O, filtering) वो bulk में columns पर होता है, और row बनाने का काम सिर्फ उन थोड़ी सी rows पर होता है जो अंत तक बच गईं।
