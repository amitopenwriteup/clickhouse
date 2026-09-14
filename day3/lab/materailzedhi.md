# ClickHouse Materialized Views Lab — Simple समझ

Ye lab तीन तरह के Materialized Views (MVs) सिखाता है, एक online store के example से (orders aur customers tables). चलो एक-एक करके समझते हैं।

## Setup (Step 0)

पहले एक `shop` database बनाते हैं जिसमें दो tables हैं:
- `customers` — छोटी सी lookup table (customer ID, name, country)
- `orders` — actual transactions जो store होंगी (order ID, customer, category, time, amount)

बस इतना समझो — ये base tables हैं जिनके ऊपर हम MVs बनाएंगे।

## Part A — Incremental MV (सबसे common type)

**Goal:** हर category का daily revenue हमेशा up-to-date रहे, बिना पूरी `orders` table को बार-बार scan किए।

- `daily_revenue` नाम की एक target table बनाते हैं, engine है `SummingMergeTree` — इसका काम है कि same `(day, category)` वाली rows के numbers अपने आप जोड़ता रहे।
- फिर `daily_revenue_mv` नाम का MV बनाते हैं जो `orders` से data लेकर `daily_revenue` में डालता है।

**सबसे ज़रूरी बात समझो:** ये MV background में लगातार नहीं चलता। जब भी `orders` में नई rows insert होती हैं, ClickHouse सिर्फ उन नई rows पर ये query चलाता है और result को `daily_revenue` में डाल देता है। यानी ये **insert के समय पर react करता है**, चाहे order की tारीख कुछ भी हो (जैसे 3 दिन पुरानी date वाला order भी अभी insert होगा तो अभी ही count होगा)।

**गड़बड़ी (gotcha):** अगर MV बनाने से **पहले** `orders` में already data पड़ा था, तो वो पुराना data कभी `daily_revenue` में नहीं आएगा — जब तक MV बनाते वक़्त `POPULATE` keyword ना लगाओ (ये backfill करने का तरीका है)।

**Takeaway:** Incremental MV असली-time single-table rollups के लिए बढ़िया है, पर सिर्फ MV बनने के बाद वाला data ही देखता है।

## Part B — Refreshable MV (JOIN वाले cases के लिए)

**Goal:** "top spending customers" report बनाना है जिसमें `orders` aur `customers` का JOIN चाहिए। क्योंकि customer का total किसी भी पुराने order से बदल सकता है (सिर्फ नए insert से नहीं), इसलिए यहाँ incremental MV सही fit नहीं है।

- `top_customers` target table बनाते हैं।
- `top_customers_mv` में `REFRESH EVERY 30 SECOND` लगाते हैं — मतलब ये MV हर 30 seconds में **पूरी query फिर से चलाता है** aur target table को overwrite कर देता है।

**Important detail:** JOIN में हर column को explicitly `AS` से rename करना ज़रूरी है (जैसे `c.customer_id AS customer_id`), वरना ClickHouse qualified name (`c.customer_id`) रख लेगा जो target table के column name से match नहीं करेगा — और error आएगा (`THERE_IS_NO_COLUMN`)।

- `SYSTEM REFRESH VIEW` से manually turant refresh करवा सकते हो।
- `system.view_refreshes` table check करने से पता चलता है कि view last कब चला aur next कब चलेगा।
- `ALTER TABLE ... MODIFY REFRESH EVERY 5 MINUTE` से schedule बाद में भी बदल सकते हो।
- `APPEND` mode optional है अगर हर refresh का history (snapshot) रखना हो, overwrite करने की बजाय।

**Takeaway:** Refreshable MV real-time freshness की जगह पूरे JOINs aur complex logic को schedule पर चलाने की सुविधा देता है।

## Part C — Cascading MV (Chain बनाना)

**Goal:** daily rollup से monthly, aur monthly से yearly rollup बनाना — बिना दोबारा raw `orders` table को touch किए।

- `monthly_revenue_mv` का source है `daily_revenue` (Part A की target table), directly `orders` नहीं।
- यहाँ table engine `AggregatingMergeTree` है aur `sumState()` function use होता है — ये final number की जगह एक **partial aggregate state** store करता है, ताकि आगे merge किया जा सके without losing precision।
- फिर `yearly_revenue_mv` का source है `monthly_revenue`, aur यहाँ `sumMerge()` use करना पड़ता है (`sum()` नहीं) क्योंकि आ रहा data partial state है, उसे पहले merge करना ज़रूरी है फिर final aggregate।

**Flow chain:**
```
orders → daily_revenue_mv → daily_revenue → monthly_revenue_mv → monthly_revenue → yearly_revenue_mv → yearly_revenue
```

सिर्फ एक बार `orders` में insert करने से पूरी chain अपने आप चल जाती है — कुछ भी manually trigger नहीं करना पड़ता।

**Takeaway:** Cascading MVs से multi-level rollups (daily → monthly → yearly) बन जाते हैं, हर level अपने नीचे वाले level से अपने आप data लेता रहता है।

## Summary Table

| Part | Type | कैसे काम करता है | कब use करें |
|---|---|---|---|
| A | Incremental | Har insert पर fire होता है | Real-time, single-table rollups |
| B | Refreshable | Timer पर पूरी query re-run होती है | Complex JOINs, periodic reports |
| C | Cascading | एक MV की target अगले MV का source बनती है | Multi-level rollups |

## याद रखने वाली Gotchas

1. Incremental MV पुराना data नहीं देखता — जब तक `POPULATE` ना लगाओ।
2. JOIN वाले incremental MV में सिर्फ **left-most (source) table** insert पर trigger होता है, joined table पर insert करने से कुछ नहीं होगा।
3. Cascading MVs में नया computed block forward होता है, पूरी merged final state नहीं — इसलिए `xState`/`xMerge` functions चाहिए होते हैं।
4. Refreshable MVs real-time freshness के बदले JOIN support देते हैं — status check करने के लिए `system.view_refreshes` देखो।
5. JOIN-based MV में हर column explicitly alias करो, वरना `THERE_IS_NO_COLUMN` error आएगा।

कुछ specific part पर aur detail चाहिए तो बताओ!
