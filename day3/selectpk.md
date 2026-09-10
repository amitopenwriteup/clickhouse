Bilkul. Is slide ko **ClickHouse ke MergeTree Primary Key / ORDER BY** ke context mein samjho.

### 1. Sabse important point

ClickHouse mein:

```sql
ORDER BY (...)
```

sirf rows ko sort karne ke liye nahi hai. **MergeTree mein यही sorting key sparse Primary Index ka basis banti hai.**

OLTP database jaise MySQL mein:

```sql
PRIMARY KEY (id)
```

ka मतलब अक्सर **unique identification** hota hai.

Lekin ClickHouse mein:

```sql
ORDER BY (event_type, event_time, user_id)
```

ka मुख्य उद्देश्य **data ko efficiently locate/skip karna** hai — uniqueness enforce karna nahi.

---

## 2. Sparse index kya karta hai?

मान लो ClickHouse mein **1 crore rows** hain.

ClickHouse har individual row ke liye index nahi banata. Data ko **granules** mein divide karta hai.

Example:

```text
1 Crore rows

Granule 1  → rows 1 - 8192
Granule 2  → rows 8193 - 16384
Granule 3  → rows 16385 - 24576
...
```

Primary index roughly batata hai:

```text
Granule 1 → event_type=login, event_time=10:00
Granule 2 → event_type=login, event_time=11:00
Granule 3 → event_type=logout, event_time=09:00
...
```

Agar query aaye:

```sql
WHERE event_type = 'login'
  AND event_time >= '2026-09-10 10:00:00'
```

to ClickHouse कह सकता है:

> "मुझे पता है कि required data इन granules में नहीं है, इसलिए बाकी granules पढ़ने की जरूरत नहीं है।"

इसे **granule pruning** कहते हैं.

---

# 3. Poorly chosen key

Slide में है:

```sql
ORDER BY event_id
```

और `event_id` एक UUID है:

```text
event_id

550e8400-e29b-41d4...
8f14e45f-ceea-...
a7c3b9...
...
```

UUID की **cardinality बहुत high** है।

मतलब लगभग हर row का value अलग है।

अब imagine करो data sorted है:

```text
UUID A
UUID B
UUID C
UUID D
UUID E
UUID F
...
```

लेकिन आपकी normal query है:

```sql
SELECT *
FROM events
WHERE event_type = 'login';
```

Primary index में `event_type` के बारे में useful ordering ही नहीं है।

इसलिए ClickHouse के लिए यह मुश्किल है कि:

> "कौन से granules में login events हैं और कौन से granules में नहीं हैं?"

Result:

```text
Granule 1 → पढ़ो
Granule 2 → पढ़ो
Granule 3 → पढ़ो
Granule 4 → पढ़ो
...
```

यानि **granule pruning बहुत कम या practically ineffective हो सकती है।**

---

# 4. Well-chosen key

Slide में:

```sql
ORDER BY (event_type, event_time, user_id)
```

अब data कुछ ऐसा organize होगा:

```text
login
 ├── 10:00
 ├── 10:01
 ├── 10:02
 └── 10:03

logout
 ├── 10:00
 ├── 10:01
 └── 10:02

purchase
 ├── 10:00
 ├── 10:01
 └── 10:02
```

अगर query है:

```sql
WHERE event_type = 'login'
```

तो ClickHouse जल्दी identify कर सकता है:

```text
login data → इन granules में है
logout data → skip
purchase data → skip
```

इससे **कम data read होगा → query faster होगी।**

---

# 5. `event_time` दूसरा क्यों?

अब query:

```sql
WHERE event_type = 'login'
AND event_time >= '2026-09-10 10:00:00'
AND event_time <  '2026-09-10 11:00:00'
```

ORDER BY है:

```sql
(event_type, event_time, user_id)
```

तो ClickHouse पहले:

```text
event_type = login
```

का area खोजेगा।

उस area के अंदर:

```text
event_time = 10:00–11:00
```

का relevant हिस्सा खोजेगा।

बाकी granules **prune** हो सकते हैं।

---

# 6. Cardinality को simple भाषा में समझो

**Cardinality = किसी column में कितने अलग-अलग values हैं।**

Example:

```text
country
----------------
India
India
USA
India
UK
USA
```

Low cardinality:

```text
country → 3 unique values
```

लेकिन:

```text
user_id
----------------
U001
U002
U003
U004
...
```

अगर 1 crore users हैं:

```text
user_id → very high cardinality
```

UUID:

```text
event_id → almost every row unique
```

इसलिए UUID को generally `ORDER BY` की **पहली column** बनाना अच्छा choice नहीं होता, अगर आपकी queries UUID पर filtering नहीं करतीं।

---

## 7. एक important correction

Slide की line:

> "Order columns from lowest to highest cardinality"

एक **useful rule of thumb** है, लेकिन इसे absolute rule मत समझना।

असल rule ज्यादा important है:

> **ORDER BY को आपके सबसे common query filters और access patterns के हिसाब से design करो।**

उदाहरण:

अगर आपकी queries mostly हैं:

```sql
WHERE customer_id = ?
AND event_time BETWEEN ? AND ?
```

तो:

```sql
ORDER BY (customer_id, event_time)
```

बहुत अच्छा हो सकता है।

अगर queries हैं:

```sql
WHERE event_type = ?
AND event_time BETWEEN ? AND ?
```

तो:

```sql
ORDER BY (event_type, event_time)
```

बेहतर हो सकता है।

इसलिए सिर्फ यह मत सोचो:

```text
Low cardinality → first
High cardinality → last
```

बल्कि सोचो:

```text
मेरी queries किस column पर सबसे ज्यादा filter करती हैं?
                ↓
क्या उस column से granules effectively skip हो सकते हैं?
                ↓
फिर ORDER BY की column sequence बनाओ
```

### याद रखने वाली एक लाइन

**ClickHouse Primary Key का काम "कौन-सी row unique है?" बताना नहीं, बल्कि "कौन-से granules पढ़ने की जरूरत नहीं है?" बताना है।**

यही वजह है कि ClickHouse में **Primary Key / ORDER BY design query performance के लिए बहुत महत्वपूर्ण है।**
