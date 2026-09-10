# Lab 3 - AggregatingMergeTree - Step by Step Explanation (हिंग्लिश)

## Objective (उद्देश्य)
ये lab सिखाता है **aggregate states** क्या होते हैं, और कैसे **-State functions** (जैसे `avgState`, `uniqState`) data store करते हैं, और **-Merge functions** (जैसे `avgMerge`, `uniqMerge`) उस data को वापस final result में convert करते हैं। ये पिछले दोनों labs (ReplacingMergeTree, SummingMergeTree) से थोड़ा **advanced** है।

---

## Step 1 - Create Database
```sql
CREATE DATABASE IF NOT EXISTS lab_aggregating;
```
एक नया database बनाया `lab_aggregating` नाम से, ताकि इस lab की सारी tables इसी के अंदर organize रहें।

---

## Step 2 - Use the Database
```sql
USE lab_aggregating;
```
अब जो भी table create/query करेंगे, वो automatically इसी database के अंदर होंगे — बार-बार database name लिखने की ज़रूरत नहीं।

---

## Step 3 - Create the Table
```sql
CREATE TABLE daily_stats
(
    order_date Date,
    avg_amount AggregateFunction(avg, Decimal(12,2)),
    unique_customers AggregateFunction(uniq, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY order_date;
```

**यहाँ सबसे important चीज़** — `avg_amount` और `unique_customers` columns **normal columns नहीं हैं**। ये `AggregateFunction(...)` type के हैं, मतलब इनमें direct number (275, 3) store नहीं होता — इनमें **aggregate का internal "state"** store होता है (एक तरह का intermediate calculation data, जिसे बाद में combine/merge करके final answer निकाला जा सकता है)।

सोच लो इसका मतलब: अगर आप avg calculate कर रहे हो, तो सिर्फ एक number store करने के बजाय, ClickHouse "sum + count" जैसी internal information store करता है — ताकि बाद में multiple parts को merge करते वक्त सही average दोबारा calculate हो सके।

---

## Step 4 - Create Raw Orders Table
```sql
CREATE TABLE raw_orders
(
    order_id UInt32,
    order_date Date,
    customer_id UInt32,
    amount Decimal(12,2)
)
ENGINE = MergeTree()
ORDER BY order_id;
```
ये एक **normal MergeTree table** है जिसमें raw (असली, बिना-process किया हुआ) order data store होगा — order ID, date, customer ID, और amount। इसी raw data से हम बाद में aggregate states बनाएंगे।

---

## Step 5 - Insert Raw Orders
4 orders insert किए 9 September के लिए:
- Order 1: Customer 101, ₹100
- Order 2: Customer 102, ₹200
- Order 3: Customer 101, ₹300 (customer 101 का **दूसरा** order)
- Order 4: Customer 103, ₹500

---

## Step 6 - Check Raw Data
```sql
SELECT * FROM raw_orders ORDER BY order_id;
```
Simple query जो सारी 4 rows दिखाती है, ताकि confirm हो जाए data सही से insert हुआ।

---

## Step 7 - Calculate the Expected Answer (Manually)
ये step **manually calculate** करने के लिए है, ताकि बाद में ClickHouse के result से compare किया जा सके:
- **Total amount** = 100+200+300+500 = 1100
- **Number of orders** = 4
- **Average** = 1100/4 = **275**
- **Unique customers** = जो customer IDs देखे: 101, 102, 101, 103 → distinct values = 101, 102, 103 → **3 unique customers**

ये numbers याद रखने हैं क्योंकि Step 10 में इन्हीं से compare करना है।

---

## Step 8 - Generate Aggregate States
```sql
INSERT INTO daily_stats
SELECT
    order_date,
    avgState(amount),
    uniqState(customer_id)
FROM raw_orders
GROUP BY order_date;
```

**ये सबसे important step है।** यहाँ `avgState(amount)` और `uniqState(customer_id)` use हो रहे हैं — ये normal `avg()` या `uniq()` नहीं हैं। `-State` suffix वाले functions final number (275 या 3) नहीं return करते — ये एक **intermediate "state"** बनाते हैं जिसमें calculation के लिए ज़रूरी raw information होती है (जैसे sum और count दोनों, ना कि सिर्फ average)।

ये state `daily_stats` table में store हो जाती है `order_date` के साथ group हो कर।

---

## Step 9 - Check the Aggregating Table
```sql
SELECT * FROM daily_stats;
```
जब आप ये query चलाओगे, तो आपको **275 या 3 जैसा simple number नहीं दिखेगा** — बल्कि कुछ weird-looking internal binary/encoded representation दिखेगा। ये बिल्कुल normal है, क्योंकि table सिर्फ **states** store कर रही है, final results नहीं।

---

## Step 10 - Read the Final Result
```sql
SELECT
    order_date,
    avgMerge(avg_amount) AS avg_order_value,
    uniqMerge(unique_customers) AS distinct_customers
FROM daily_stats
GROUP BY order_date;
```

अब `-Merge` functions use करते हैं (`avgMerge`, `uniqMerge`) जो उन stored states को वापस **readable final numbers** में convert करते हैं।

**Result:** `avg_order_value = 275`, `distinct_customers = 3` — जो हमारे Step 7 के manual calculation से **match** करता है।

---

## Step 11 - Understand State and Merge (Concept Clarify)
ये purely explanation step है (कोई query नहीं):
- **Average के लिए:** raw data → `avgState()` → एक AVG state बनती है → वो state table में store होती है → बाद में `avgMerge()` से final average निकालते हैं।
- **Unique customers के लिए:** raw data → `uniqState()` → UNIQ state बनती है → store होती है → `uniqMerge()` से final unique count निकालते हैं।

Simple words में: **State = data को एक special format में "save for later" करना**, और **Merge = उस saved data को combine करके असली answer निकालना।**

---

## Step 12 - Add More Data
10 September के लिए 3 नए orders insert किए:
- Customer 101: ₹400
- Customer 104: ₹600
- Customer 105: ₹200

फिर एक `SELECT` query से confirm किया कि ये data raw_orders में सही से आ गया।

---

## Step 13 - Create Another Aggregate State
```sql
INSERT INTO daily_stats
SELECT
    order_date,
    avgState(amount),
    uniqState(customer_id)
FROM raw_orders
WHERE order_date = '2026-09-10'
GROUP BY order_date;
```
Same process जो Step 8 में किया था, लेकिन सिर्फ 10 September के data के लिए — एक और aggregate state row `daily_stats` में add हो गई।

---

## Step 14 - Read Both Dates
Same `avgMerge`/`uniqMerge` query दोनों dates के लिए चलाई:

**Result:**
- 9 September: average = 275, unique customers = 3
- 10 September: average = 400 (क्योंकि 400+600+200=1200, और 1200/3=400), unique customers = 3 (customers 101, 104, 105 — तीनों अलग)

---

## Step 15 - Cross-Verify Using the Raw Table
ये step **trust-building** के लिए है — हम सीधा raw_orders table पर normal `avg()` और `uniq()` functions चला कर check करते हैं कि वही answer आता है जो `daily_stats` (aggregate state table) से आया।

दोनों queries का result **match** होना चाहिए — ये confirm करता है कि AggregatingMergeTree का पूरा State→Merge process सही काम कर रहा है।

---

## Step 16 - Understand Why AggregatingMergeTree Is Powerful
ये explanation step बताता है कि जब आपको सिर्फ SUM चाहिए (simple addition), तो `SummingMergeTree` काफ़ी है। लेकिन जब आपको **AVG, UNIQUE COUNT, QUANTILE** जैसी complex calculations चाहिए जो simple addition से नहीं हो सकती, तब `AggregatingMergeTree` use करते हैं — जो `avgState()`, `uniqState()`, `quantileState()` जैसी states store कर सकता है और उन्हें `avgMerge()`, `uniqMerge()`, `quantileMerge()` से combine कर सकता है।

---

## Key Learning (Lab 3 का सार)
- **लिखते वक्त (Insert करते वक्त):** `-State` functions use करो — `avgState()`, `uniqState()`
- **पढ़ते वक्त (Query करते वक्त):** `-Merge` functions use करो — `avgMerge()`, `uniqMerge()`

**याद रखने का trick:**
- **State** = Calculation को prepare करके store करना
- **Merge** = उस stored calculation को combine करके final result निकालना

---

## Cleanup
```sql
DROP TABLE IF EXISTS lab_aggregating.daily_stats;
DROP TABLE IF EXISTS lab_aggregating.raw_orders;
DROP DATABASE IF EXISTS lab_aggregating;
SHOW DATABASES;
```
दोनों tables और database delete कर दिए, और `SHOW DATABASES` से confirm किया कि तीनों lab databases (`lab_replacing`, `lab_summing`, `lab_aggregating`) अब exist नहीं करते।

---

## Final Comparison — तीनों Engines का सार

| Engine | Purpose | Example | याद रखने का तरीका |
|---|---|---|---|
| **ReplacingMergeTree** | Deduplication / latest version रखना | Version 1, 2, 3 → सिर्फ Version 3 रखो | **KEEP ONE** |
| **SummingMergeTree** | Simple numeric addition | 100, 200, 300 → 600 | **ADD THEM** |
| **AggregatingMergeTree** | Complex aggregation (avg, unique count, quantile) | Raw data → State → Merge → Result | **STORE + MERGE STATES** |

**One-liner summary:**
- ReplacingMergeTree → एक record का **सबसे latest version** रखता है
- SummingMergeTree → same key वाली rows के numbers को **add** कर देता है
- AggregatingMergeTree → complex calculations (जो simple addition से possible नहीं) के लिए **states store करके बाद में merge** करता है
