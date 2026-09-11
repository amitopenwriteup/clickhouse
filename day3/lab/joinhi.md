Chaliye is ClickHouse JOINs lab ko bhi step-by-step samjhte hain, Hinglish mein.

## Step 0: Setup — दो Tables बनाना

```sql
CREATE TABLE orders (order_id, customer_id, amount) ORDER BY order_id;
CREATE TABLE customers (customer_id, name) ORDER BY customer_id;
```

- **orders** table में 4 rows डाली गई हैं (order_id 101-104), जिनमें customer_id 1, 2, 1, 3 हैं।
- **customers** table में सिर्फ 2 rows हैं: customer_id 1 (Amit) और 2 (Rahul)।
- **जानबूझकर गलती छोड़ी गई है**: customer_id = 3 का कोई row customers table में **नहीं** है (order 104 के लिए)। ये आगे JOIN types का फर्क समझाने के लिए है।

## Step 1: Basic JOIN (= INNER JOIN)

```sql
SELECT o.order_id, o.amount, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

- ClickHouse में plain **JOIN का मतलब INNER JOIN** होता है।
- Rule सीधा है: सिर्फ वही rows result में आएंगी जिनका match **दोनों** tables में मिले।
- चूंकि customer_id = 3 (order 104) customers table में नहीं है, इसलिए वो row पूरी तरह **गायब** हो जाएगी।
- **Result: सिर्फ 3 rows** (orders में 4 थीं, पर 1 drop हो गई)।

## Step 2: LEFT JOIN — सब orders दिखाना, match मिले या न मिले

```sql
SELECT o.order_id, o.amount, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

- **LEFT table (orders) की हर row** result में ज़रूर आएगी, चाहे match मिले या न मिले।
- अगर RIGHT table (customers) में match नहीं मिलता, तो उस column में **NULL/empty** भर दिया जाता है।
- यहाँ order 104 दिखेगा, पर उसके `name` column में **NULL** होगा (क्योंकि customer_id=3 customers में है ही नहीं)।
- **Result: सारी 4 rows** आएंगी।

**याद रखने वाली बात:** LEFT table = हमेशा पूरी दिखेगी; RIGHT table = जहाँ match नहीं, वहाँ NULL।

## Step 3: Duplicate Rows Problem — ANY बनाम ALL

पहले एक duplicate डालते हैं customers table में:

```sql
INSERT INTO customers VALUES (1, 'Amit Kumar');
```

अब customer_id = 1 के लिए customers table में **दो names** हैं: "Amit" और "Amit Kumar"।

**Plain JOIN चलाने पर:**
```sql
SELECT o.order_id, o.amount, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```
- customer_id=1 वाले हर order (101, 103) के लिए **दो-दो matches** मिलेंगे customers table में।
- इसका मतलब है orders 101 और 103 की rows **duplicate (double)** हो जाएंगी result में — एक "Amit" वाली, एक "Amit Kumar" वाली।
- ये एक **row-duplication problem** है, जो आगे chalke sum()/count() लगाने पर गलत result देगी।

**अब ANY JOIN try करते हैं:**
```sql
SELECT o.order_id, o.amount, c.name
FROM orders o
ANY LEFT JOIN customers c ON o.customer_id = c.customer_id;
```
- **ANY JOIN** सिर्फ **पहला match** लेता है, बाकी duplicates को ignore कर देता है।
- हर order अब सिर्फ **एक बार** दिखेगा।

**फर्क याद रखो:**
- **ALL (default behavior)** — अगर duplicate match मिले, तो rows multiply हो जाती हैं।
- **ANY** — सिर्फ पहला match लिया जाता है, extra rows नहीं बनतीं (aggregation से पहले safe रहता है)।

## Step 4: JOIN + Aggregation — यहीं गलती होती है

```sql
-- WRONG: amount double count हो सकता है duplicate की वजह से
SELECT sum(o.amount) AS total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- CORRECT: ANY JOIN duplicate हटा देता है
SELECT sum(o.amount) AS total
FROM orders o
ANY LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

- पहली query में orders 101 और 103 (जो customer_id=1 वाले हैं) **दो-दो बार count** होंगे, इसलिए total **ज़्यादा (गलत)** आएगा।
- दूसरी query ANY JOIN की वजह से हर order को सिर्फ एक बार गिनती है, तो total **सही** आएगा।

**Lesson:** जब भी JOIN के बाद SUM/COUNT/AVG लगाओ, पहले चेक करो कि RIGHT table में duplicate keys तो नहीं हैं — अगर हैं, तो **ANY JOIN** का इस्तेमाल करो।

## Step 5: Performance Rule — RIGHT table को छोटा रखो

- ClickHouse का default JOIN (**hash join**) पूरी **RIGHT table को RAM में load** कर लेता है।
- इसलिए हमेशा:
  - **बड़ी table (orders) को LEFT** में रखो
  - **छोटी table (customers) को RIGHT** में रखो

```sql
-- GOOD
SELECT o.order_id, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

अगर उल्टा किया (एक huge table को RIGHT में रखा) तो RAM खत्म होने का खतरा है।

**Simple rule याद रखो:** "बड़ी table हमेशा LEFT, छोटी table हमेशा RIGHT।"

## Wrap-up Challenge — हर customer का total दिखाना (भले ही उसका कोई order न हो)

**Question:** हर customer का नाम और उसके सारे orders का total amount दिखाओ — भले ही किसी customer का कोई order न हो, फिर भी वो दिखना चाहिए, total = 0 के साथ।

```sql
SELECT
    c.customer_id,
    any(c.name) AS name,
    sum(o.amount) AS total_amount
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY c.customer_id
ORDER BY c.customer_id;
```

यहाँ ध्यान देने वाली बातें:
- अब **LEFT table customers है**, orders नहीं — क्योंकि हमें हर customer दिखाना है, चाहे उसका order हो या न हो।
- **`any(c.name)`** इस्तेमाल हुआ है क्योंकि GROUP BY के साथ हर non-aggregated column को किसी aggregate function में wrap करना पड़ता है (चूंकि customers में duplicate names की वजह से confusion हो सकता है)।
- `sum(o.amount)` उस customer के सारे orders का total निकालेगा; अगर कोई order नहीं है तो ClickHouse में ये 0 या NULL आ सकता है (table engine और settings पर depend करता है)।

## छह Steps का सार (Summary)

| JOIN Type | क्या करता है |
|---|---|
| **INNER (plain JOIN)** | सिर्फ matching rows, बाकी drop |
| **LEFT JOIN** | LEFT table की सारी rows, no-match पर NULL |
| **ANY JOIN** | सिर्फ पहला match, duplicates से बचाव |
| **ALL (default)** | duplicate match पर rows multiply |
| **Performance rule** | बड़ी table LEFT, छोटी table RIGHT (RAM बचाने के लिए) |

Chahiye toh main dono labs (primary key + joins) ko ek combined practice script mein bhi bana sakta hoon jisse aap ek hi baar mein pura run kar sako.
