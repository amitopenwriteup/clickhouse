# ClickHouse में Primary Key कैसे चुनें — Hinglish में विस्तृत व्याख्या

## सबसे पहले समझ लो की ClickHouse में Primary Key अलग होती है

Traditional databases (जैसे Postgres) में primary key सिर्फ एक identifier होती है। लेकिन **ClickHouse में बिल्कुल अलग काम है**।

```
Normal Database:        Primary Key = सिर्फ unique identifier
ClickHouse:           Primary Key = डिस्क पर डेटा का physical order
```

---

## PRIMARY KEY का असली काम क्या है?

### 1️⃣ **Data को Disk पर कैसे store करता है**

ClickHouse में primary key से:
- डेटा को **sorted order** में डिस्क पर store होता है
- यही वजह से **compression बेहतर** होती है
- **Queries faster** हो जाती हैं

### 2️⃣ **Sparse Index बनाता है**

हर block के लिए एक index entry बनता है (हर row के लिए नहीं)।

```
❌ Dense Index:  Row 1 → Row 2 → Row 3 → ... (लाखों entries)
✅ Sparse Index: Block 1 → Block 2 → Block 3 (सैकड़ों entries)
```

### 3️⃣ **WHERE clause में तेजी**

जब query में filter लगता है, तो unnecessary blocks को **skip** कर देता है।

---

## PRIMARY KEY चुनने के 2 मुख्य नियम

### नियम #1: अक्सर Filter में आने वाली columns को चुनो

```sql
SELECT count()
FROM orders
WHERE date >= '2024-01-01' AND category = 'electronics'
```

अगर ये query अक्सर चलता है, तो PRIMARY KEY में `(category, date)` रखना चाहिए।

### नियम #2: Columns को सही order में arrange करो

```
RULE: Low cardinality columns पहले, फिर high cardinality
```

**Cardinality** = कितने unique values हैं।

```
PostTypeId:      8 values (Question, Answer, Wiki...)  ← Low cardinality
CreationDate:    1000+ values                          ← High cardinality
```

**इसलिए**: `ORDER BY (PostTypeId, toDate(CreationDate))`

---

## Real Example: Stack Overflow Posts

### Scenario: "2024 के बाद कितने Questions हैं?"

#### ❌ बिना Primary Key (समस्या)

```sql
CREATE TABLE posts_unordered
ENGINE = MergeTree
ORDER BY tuple()  -- कोई PRIMARY KEY नहीं!
```

Query का परिणाम:
```
Rows Processed: 59.82 MILLION
Time: 0.055 sec
```

**पूरी table स्कैन करनी पड़ी!** 😱

#### ✅ सही Primary Key के साथ (समाधान)

```sql
CREATE TABLE posts_ordered
ENGINE = MergeTree
ORDER BY (PostTypeId, toDate(CreationDate))
```

Query का परिणाम:
```
Rows Processed: 196.53 THOUSAND  (पहले से 300x कम!)
Time: 0.013 sec
```

**Speed में 4x सुधार!** 🚀

---

## Index कैसे काम करता है?

### Sparse Index का concept

```
Index (Block headers):
Block 1: PostTypeId=1, Date=2023-01-01
Block 2: PostTypeId=1, Date=2023-06-01
Block 3: PostTypeId=1, Date=2024-01-01  ← ये block खोल
Block 4: PostTypeId=2, Date=2023-01-01  ← ये block skip

Query: WHERE PostTypeId=1 AND Date >= 2024-01-01
```

- Total 7578 granules (blocks)
- सिर्फ 39 granules को check किया
- बाकी 7539 को **skip** कर दिया

---

## PRIMARY KEY के 4-5 Columns कितने काफी हैं?

आमतौर पर:

```
1. सबसे frequently filtered column (low cardinality)
2. दूसरा filtered column
3. तीसरा filtered column
4. Grouping के लिए आने वाली column
5. (Optional) Time-based filtering के लिए
```

**ज्यादा columns = ज्यादा overhead, कम benefit**

---

## Important ⚠️ Gotchas

### Gotcha #1: बाद में नहीं बदल सकते

```sql
-- ❌ ये नहीं हो सकता
ALTER TABLE posts_ordered MODIFY ORDER BY (new_column);

-- ✅ सिर्फ Projections add कर सकते हो (लेकिन duplicate data)
ALTER TABLE posts_ordered ADD PROJECTION proj_new
AS SELECT * ORDER BY (new_column);
```

### Gotcha #2: सभी Columns को Sort करता है

```sql
ORDER BY (PostTypeId, CreationDate)
```

**मतलब**: Title, Body, Score सब भी इसी order में sorted हो जाएंगे!

यही वजह है compression बेहतर है।

---

## Step-by-Step Guide: बेस्ट PRIMARY KEY चुनने का Process

### Step 1: सभी Queries को देखो

```sql
-- Most common queries
Q1: WHERE date >= '2024-01-01' AND category = 'electronics'
Q2: WHERE user_id = 123
Q3: SELECT sum(amount) GROUP BY category
```

### Step 2: Most selective columns पहचानो

```
date >= '2024-01-01'    ← 50% rows filter करता है (अच्छा)
category = 'electronics' ← 20% rows filter करता है (बहुत अच्छा)
user_id = 123           ← 0.01% rows filter करता है (बेहतरीन!)
```

### Step 3: Low-to-High cardinality में arrange करो

```
category (8 values)   ← पहले
date (365 values)     ← फिर
user_id (1M values)   ← आखिर में
```

### Final ORDER BY

```sql
CREATE TABLE orders
ENGINE = MergeTree
ORDER BY (category, toDate(date), user_id)
```

---

## अतिरिक्त Optimization

### Tip #1: toDate() का use करो DateTime पर

```sql
-- ❌ बड़ा index
ORDER BY (CreationDate)        -- 8 bytes per value

-- ✅ छोटा index  
ORDER BY (toDate(CreationDate)) -- 2 bytes per value
```

### Tip #2: Cardinality को ध्यान में रखो

```
Columns by Cardinality:
Low:   Status (5 values), Type (10 values)
High:  UserID (1 million), Timestamp (1 billion)

Best: ORDER BY (Status, Type, Timestamp)
```

### Tip #3: GROUP BY के columns को भी include करो

```sql
-- अगर ये query बहुत चलता है:
SELECT category, sum(amount) 
FROM orders 
GROUP BY category

-- तो category को ORDER BY में रखो
ORDER BY (category, date, user_id)
```

---

## Compression का फायदा

### Sorted Data को compress करना आसान है

```
❌ Random order:
   User 100, User 5, User 200, User 3, ...
   (बहुत variation, कम compress)

✅ Sorted order:
   User 1, User 2, User 3, User 4, ...
   (pattern दिख जाता है, ज्यादा compress)
```

**Result**: 
- कम disk space चाहिए
- कम I/O operations
- तेजी से queries

---

## Real Numbers: पहले vs बाद

| Metric | बिना Index | सही Index |
|--------|-----------|-----------|
| **Rows Processed** | 59.82M | 0.196M |
| **Bytes Processed** | 361.34 MB | 1.77 MB |
| **Query Time** | 0.055 sec | 0.013 sec |
| **Speed Gain** | - | **4x faster** |

---

## निष्कर्ष (Summary)

✅ **PRIMARY KEY चुनते समय**:
1. अपनी queries देखो — कौन से WHERE conditions चलते हैं?
2. Low cardinality columns को पहले रखो
3. High cardinality columns को बाद में रखो
4. Time-based filtering को support करो (toDate() का use करो)
5. एक बार बना दो — बाद में नहीं बदल सकते!

✅ **Benefits**:
- Sparse Index तेजी से rows filter करता है
- Data sorted रहता है, इसलिए compression बेहतर
- Overall query performance में 4x तक सुधार

🚀 **Remember**: ClickHouse का magic PRIMARY KEY से ही शुरू होता है!
