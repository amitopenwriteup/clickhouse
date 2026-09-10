हाँ। ClickHouse की **Window Functions** को सबसे आसान तरीके से समझते हैं। Official documentation के अनुसार, window function ऐसी calculation करता है जिसमें **current row के साथ उससे related rows को भी देखा जाता है, लेकिन rows collapse/group होकर एक row नहीं बनतीं**। यही इसका सबसे important concept है। ([ClickHouse][1])

## 1. पहले समझिए — Window Function क्या करता है?

मान लीजिए हमारे पास logs हैं:

```text
service    event_time    latency
--------   ----------    -------
api        10:00         100
api        10:01         150
api        10:02         120
web        10:00         80
web        10:01         90
```

अगर हम normal aggregation करें:

```sql
SELECT
    service,
    avg(latency)
FROM logs
GROUP BY service;
```

Output:

```text
service    avg_latency
-------    -----------
api        123.33
web        85
```

यहाँ **5 rows → 2 rows** हो गईं।

लेकिन Window Function में:

```sql
SELECT
    service,
    event_time,
    latency,
    avg(latency) OVER (PARTITION BY service)
FROM logs;
```

Output conceptually:

```text
service    time     latency    avg
-------    -----    -------    ------
api        10:00      100      123.33
api        10:01      150      123.33
api        10:02      120      123.33
web        10:00       80       85
web        10:01       90       85
```

**यही Window Function की सबसे बड़ी खूबी है।**

> Calculation भी हो रही है और original rows भी बची हुई हैं।

ClickHouse documentation भी इसी distinction को बताती है। ([ClickHouse][1])

---

# 2. Basic syntax

ClickHouse में सामान्य syntax है:

```sql
function(...)
OVER
(
    PARTITION BY ...
    ORDER BY ...
    ROWS / RANGE / GROUPS ...
)
```

Documentation के अनुसार तीन मुख्य चीजें समझनी हैं: `PARTITION BY`, `ORDER BY` और window **frame**. ([ClickHouse][1])

इसे ऐसे याद रखिए:

```text
             WINDOW
                |
       +--------+--------+
       |        |        |
   PARTITION  ORDER BY  FRAME
       |        |        |
    कौन सा    किस      कितनी
    group     order     rows
```

---

# 3. PARTITION BY क्या है?

`PARTITION BY` का मतलब:

**"Rows को calculation के लिए अलग-अलग groups में बाँट दो."**

Example:

```sql
avg(latency) OVER
(
    PARTITION BY service
)
```

अगर data है:

```text
api    100
api    150
api    120

web     80
web     90
```

तो ClickHouse internally conceptually दो windows बनाएगा:

```text
Window 1
--------
api
100
150
120


Window 2
--------
web
80
90
```

फिर `avg()` हर window पर calculate होगा।

इसलिए:

```sql
avg(latency) OVER (PARTITION BY service)
```

का मतलब:

> **हर service का average निकालो, लेकिन हर original row के साथ उसका average दिखाओ।**

---

# 4. PARTITION BY और ClickHouse PARTITION BY को confuse मत करना

यह बहुत important है, खासकर क्योंकि आप MergeTree पढ़ रहे हैं।

Table definition:

```sql
CREATE TABLE logs
(
    event_time DateTime,
    service String,
    latency UInt32
)
ENGINE = MergeTree
PARTITION BY toDate(event_time)
ORDER BY (service, event_time);
```

यह:

```sql
PARTITION BY toDate(event_time)
```

**MergeTree table partitioning है।**

लेकिन:

```sql
avg(latency) OVER
(
    PARTITION BY service
)
```

यह **Window Function का logical partition** है।

दोनों अलग concepts हैं।

```text
MergeTree PARTITION BY
        ↓
Physical data organization


Window PARTITION BY
        ↓
Query calculation का logical group
```

---

# 5. ORDER BY क्यों?

अब मान लीजिए हमें हर service के अंदर events को time के हिसाब से देखना है।

```sql
lag(latency)
OVER
(
    PARTITION BY service
    ORDER BY event_time
)
```

इसका मतलब:

```text
पहले service के हिसाब से group करो
             ↓
फिर time के हिसाब से arrange करो
             ↓
फिर previous row निकालो
```

Example:

```text
api

10:00   100
10:01   150
10:02   120
```

`lag(latency)`:

```text
10:00   100   NULL
10:01   150   100
10:02   120   150
```

यानि:

> **Current row से पिछली row की value दो।**

ClickHouse में `lag()` और `lead()` supported हैं। ([ClickHouse][1])

---

# 6. LAG — बहुत useful function

Example:

```sql
SELECT
    service,
    event_time,
    latency,

    lag(latency, 1)
        OVER (
            PARTITION BY service
            ORDER BY event_time
        ) AS previous_latency

FROM logs;
```

Result:

```text
service   time    latency   previous
-------   -----   -------   --------
api       10:00     100       NULL
api       10:01     150       100
api       10:02     120       150
```

अब हम difference निकाल सकते हैं:

```sql
latency -
lag(latency, 1) OVER
(
    PARTITION BY service
    ORDER BY event_time
)
```

Result:

```text
100 - NULL = NULL
150 - 100  = 50
120 - 150  = -30
```

इससे आप आसानी से detect कर सकते हैं:

> **Latency पिछली observation से कितनी बदली?**

---

# 7. LEAD क्या करता है?

`lag()` = पीछे देखो

`lead()` = आगे देखो

```sql
lead(latency, 1)
OVER
(
    PARTITION BY service
    ORDER BY event_time
)
```

Example:

```text
time     latency    lead
------   -------    ----
10:00      100      150
10:01      150      120
10:02      120      NULL
```

Documentation में `lag(x, offset)` को current row से offset rows पहले की value और `lead(x, offset)` को offset rows बाद की value के रूप में define किया गया है। ([ClickHouse][1])

---

# 8. ROW_NUMBER()

यह सबसे simple window function है।

```sql
row_number()
OVER
(
    ORDER BY latency DESC
)
```

अगर:

```text
A  500
B  400
C  300
```

तो:

```text
A  500  1
B  400  2
C  300  3
```

मतलब:

> **हर row को क्रम संख्या दो।**

अगर service-wise चाहिए:

```sql
row_number()
OVER
(
    PARTITION BY service
    ORDER BY latency DESC
)
```

तो:

```text
api    500    1
api    400    2
api    300    3

web    700    1
web    600    2
```

ध्यान दें कि numbering **हर partition में फिर से 1 से शुरू होती है**। ([ClickHouse][1])

---

# 9. RANK vs DENSE_RANK vs ROW_NUMBER

यह interview और real-world दोनों में बहुत important है।

Data:

```text
salary
------
100
100
90
80
```

### ROW_NUMBER

```sql
row_number() OVER (ORDER BY salary DESC)
```

Result:

```text
100   1
100   2
90    3
80    4
```

हर row का unique number।

---

### RANK

```sql
rank() OVER (ORDER BY salary DESC)
```

Result:

```text
100   1
100   1
90    3
80    4
```

100 दोनों को rank 1 मिला।

लेकिन अगला rank **3** है।

क्यों?

क्योंकि दो लोग पहले स्थान पर हैं।

---

### DENSE_RANK

```sql
dense_rank() OVER (ORDER BY salary DESC)
```

Result:

```text
100   1
100   1
90    2
80    3
```

यहाँ gap नहीं आता।

ClickHouse documentation भी इसी distinction को example में दिखाती है। ([ClickHouse][1])

याद रखने का तरीका:

```text
ROW_NUMBER
हर row अलग

RANK
tie होने पर gap

DENSE_RANK
tie होने पर gap नहीं
```

---

# 10. सबसे important concept — Window Frame

अब थोड़ा advanced लेकिन बहुत important हिस्सा।

Example:

```sql
sum(latency)
OVER
(
    ORDER BY event_time
)
```

यह सिर्फ पूरा dataset नहीं देख रहा होता।

Window में एक **frame** भी हो सकता है।

Frame का मतलब:

> **Current row के लिए calculation करते समय exactly कौन-कौन सी rows देखनी हैं?**

ClickHouse `ROWS`, `RANGE` और `GROUPS` frames support करता है। ([ClickHouse][1])

---

# 11. ROWS

मान लीजिए:

```text
time    value
----    -----
1       10
2       20
3       30
4       40
5       50
```

Query:

```sql
sum(value) OVER
(
    ORDER BY time
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)
```

अब हर row पर पीछे की **2 physical rows + current row** देखें।

Result:

```text
value    calculation          sum
-----    -----------          ---
10       10                   10
20       10+20                30
30       10+20+30             60
40       20+30+40             90
50       30+40+50             120
```

यह moving window है।

Observability में यह बहुत useful है:

```text
3-event moving average
5-event moving average
10-event moving average
```

---

# 12. RANGE

`ROWS` और `RANGE` में important difference है।

Documentation के अनुसार:

* `ROWS` = physical rows count करता है
* `RANGE` = `ORDER BY` की values के आधार पर range बनाता है
* `GROUPS` = peer groups count करता है ([ClickHouse][1])

इसलिए `RANGE` को simple भाषा में ऐसे समझें:

```text
ROWS
↓
"मुझे पिछली 2 rows दो"

RANGE
↓
"मुझे ORDER BY value के हिसाब से इस range के अंदर
आने वाली rows दो"
```

---

# 13. Default frame बहुत important है

ClickHouse documentation के अनुसार अगर frame explicitly नहीं दिया गया है तो `RANGE` default होता है:

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

([ClickHouse][1])

इसलिए यह:

```sql
sum(value) OVER
(
    ORDER BY time
)
```

conceptually default frame के साथ काम करता है।

---

# 14. Running Total

यह Window Functions का बहुत common use case है।

Data:

```text
day    orders
---    ------
1      10
2      20
3      15
4      25
```

Query:

```sql
SELECT
    day,
    orders,

    sum(orders) OVER
    (
        ORDER BY day
    ) AS running_total

FROM orders;
```

Result:

```text
day   orders   running_total
---   ------   -------------
1       10          10
2       20          30
3       15          45
4       25          70
```

यानि:

```text
10
10 + 20
10 + 20 + 15
10 + 20 + 15 + 25
```

---

# 15. PARTITION + ORDER BY + FRAME together

अब असली power समझिए।

```sql
sum(orders) OVER
(
    PARTITION BY customer
    ORDER BY order_time
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

इसका अर्थ:

```text
             customer
                ↓
       अलग-अलग customer
                ↓
          order_time
             ↓
        chronological
                ↓
       beginning → current
                ↓
        running total
```

Example:

```text
customer   time    orders   running
--------   -----   ------   -------
A          10:00     10       10
A          11:00     20       30
A          12:00     15       45

B          10:00      5        5
B          11:00     10       15
```

हर customer का running total अलग है।

---

# 16. FIRST_VALUE और LAST_VALUE

### FIRST_VALUE

```sql
first_value(latency)
OVER
(
    PARTITION BY service
    ORDER BY event_time
)
```

मतलब:

> इस service की window में पहली value क्या थी?

Example:

```text
api

10:00   100
10:01   150
10:02   120
```

Result:

```text
100
100
100
```

---

### LAST_VALUE

यह थोड़ा tricky है क्योंकि **frame important है**।

```sql
last_value(latency)
OVER
(
    PARTITION BY service
    ORDER BY event_time
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND UNBOUNDED FOLLOWING
)
```

यह पूरे partition की आखिरी value देगा:

```text
100   120
150   120
120   120
```

ClickHouse documentation में `first_value`, `last_value` और `nth_value` window-only functions के रूप में दिए गए हैं। ([ClickHouse][1])

---

# 17. ClickHouse में Window Functions की पूरी family

Official documentation के अनुसार प्रमुख functions हैं: ([ClickHouse][1])

| Function                  | आसान मतलब                             |
| ------------------------- | ------------------------------------- |
| `row_number()`            | row number                            |
| `rank()`                  | ranking with gaps                     |
| `dense_rank()`            | ranking without gaps                  |
| `lag()`                   | पिछली row                             |
| `lead()`                  | अगली row                              |
| `lagInFrame()`            | frame के अंदर पिछली row               |
| `leadInFrame()`           | frame के अंदर अगली row                |
| `first_value()`           | पहली value                            |
| `last_value()`            | आखिरी value                           |
| `nth_value()`             | Nth value                             |
| `ntile()`                 | rows को buckets में बाँटो             |
| `percent_rank()`          | relative ranking                      |
| `cume_dist()`             | cumulative distribution               |
| `nonNegativeDerivative()` | metric का non-negative rate of change |

`nonNegativeDerivative()` ClickHouse-specific extension है। ([ClickHouse][1])

---

# 18. Observability में इसका सबसे अच्छा example

आप ClickHouse में logs store कर रहे हैं:

```text
service
event_time
status_code
latency_ms
```

अब आप पूछ सकते हैं:

### Previous latency?

```sql
lag(latency_ms)
OVER (
    PARTITION BY service
    ORDER BY event_time
)
```

### Change from previous latency?

```sql
latency_ms -
lag(latency_ms)
OVER (
    PARTITION BY service
    ORDER BY event_time
)
```

### Service का average latency?

```sql
avg(latency_ms)
OVER (
    PARTITION BY service
)
```

### Service में latency ranking?

```sql
rank()
OVER (
    PARTITION BY service
    ORDER BY latency_ms DESC
)
```

### Moving average?

```sql
avg(latency_ms)
OVER (
    PARTITION BY service
    ORDER BY event_time
    ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
)
```

इसका मतलब:

> **हर service के लिए पिछले 5 events का moving average latency।**

---

# 19. एक बहुत important mental model

Window Function को हमेशा इस तरीके से सोचिए:

```text
Original Rows
     │
     ▼
PARTITION BY
"किस group में calculation करनी है?"
     │
     ▼
ORDER BY
"group के अंदर किस order में?"
     │
     ▼
FRAME
"कितनी rows/value देखनी है?"
     │
     ▼
WINDOW FUNCTION
"क्या calculation करनी है?"
     │
     ▼
Original rows + calculated value
```

उदाहरण:

```sql
avg(latency)
OVER
(
    PARTITION BY service
    ORDER BY event_time
    ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
)
```

इसे हिन्दी में पढ़ें:

> **हर service को अलग करो → time के हिसाब से लगाओ → current row सहित पिछली 4 rows देखो → उनका average निकालो → result को उसी row के साथ दिखाओ।**

यही **Window Function** का पूरा concept है।

और एक बात विशेष रूप से याद रखें:

```text
GROUP BY
    ↓
rows को collapse करता है

WINDOW FUNCTION
    ↓
rows को collapse नहीं करता
calculation को rows के साथ attach करता है
```

यह difference समझ गया तो ClickHouse Window Functions का लगभग पूरा foundation clear हो जाता है। ([ClickHouse][1])

[1]: https://clickhouse.com/docs/reference/functions/window-functions "Window Functions - ClickHouse Documentation"
