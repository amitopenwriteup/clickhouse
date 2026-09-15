# ClickHouse Materialized Views Lab — हिंग्लिश में समझते हैं

## 0. Setup — Scenario
हम एक online store का orders stream simulate कर रहे हैं। इसके लिए `shop` नाम का database बनाते हैं, जिसमें दो base tables हैं:
- `customers` — customer का ID, नाम, country
- `orders` — हर order का ID, किसने ख़रीदा, कौनसी category, कब, कितने का

फिर 3 customers insert कर देते हैं टेस्ट डेटा के तौर पे। ये पूरा setup baaki तीनों parts (A, B, C) के लिए foundation है।

**Definition (परिभाषा):** Materialized View (MV) मतलब एक ऐसा "automatic pipeline" जो नए data insert होते ही (या schedule पे) खुद-ब-खुद एक query चलाकर result को किसी दूसरे table में save कर देता है — ताकि हमें बार-बार भारी (heavy) query manually ना चलानी पड़े।

---

## Part A — Incremental Materialized View
**Goal:** हर category का daily revenue हमेशा up-to-date रखना, बिना पूरे `orders` table को बार-बार scan किए।

- **Step 1:** एक target table `daily_revenue` बनाते हैं, engine है `SummingMergeTree` — इसका खासियत ये है कि जब same `(day, category)` वाली multiple rows आती हैं, तो ये background में automatically उनके numeric columns (`total_amount`, `order_count`) को **जोड़ (add)** देता है।
- **Step 2:** `daily_revenue_mv` नाम की MV बनाते हैं जो `orders` table को source मानती है — जैसे ही `orders` में कोई नई row insert होती है, ये MV उसी नई row पे तुरंत ग्रुप-बाय (`GROUP BY day, category`) चलाकर result `daily_revenue` में डाल देती है। ये **incremental** है, यानी सिर्फ नई rows पे काम करती है, पूरे table पे नहीं।
- **Step 3:** कुछ orders insert करके check करते हैं कि `daily_revenue` insert के समय ही (query के समय नहीं) update हो गया।
- **Step 4 — गड़बड़ी (gotcha):** अगर कोई row का `order_time` पुराना (past date) है, फिर भी वो अभी पिक-अप हो जाएगी — क्योंकि MV इस बात पे react करती है कि row **कब insert हुई**, ना कि उसका timestamp क्या है। लेकिन अगर `orders` में पहले से (MV बनने से पहले) rows मौजूद थीं, तो वो कभी `daily_revenue` में नहीं आएँगी — जब तक हम MV बनाते वक़्त `POPULATE` keyword ना लगाएँ (जो existing data को भी एक बार process कर देता है)।

**Definition:** Incremental MV मतलब — हर नई insert के साथ सिर्फ उतना ही काम (delta) करना, पूरे history को दोबारा (dobara) scan नहीं करना।

---

## Part B — Refreshable Materialized View
**Goal:** "Top spending customers" वाली report बनानी है, जिसमें `orders` और `customers` का JOIN चाहिए।

चूँकि किसी customer का total spend किसी भी पुराने order से प्रभावित (affected) हो सकता है, इसलिए ये incremental तरीके से efficiently नहीं हो सकता — पूरी query को समय-समय पे फिर से चलाना पड़ता है। इसलिए यहाँ **Refreshable MV** use होती है।

- **Step 1:** Target table `top_customers` बनाते हैं।
- **Step 2:** MV बनाते हैं `REFRESH EVERY 30 SECOND` के साथ — मतलब हर 30 seconds में पूरी JOIN query फिर से चलेगी और result overwrite हो जाएगा। ये individual inserts पे react नहीं करती, सिर्फ timer पे चलती है।
- **Step 3:** `SYSTEM REFRESH VIEW` से manually तुरंत refresh करवा सकते हैं (timer का इंतज़ार किए बिना), फिर result check करते हैं।
- **Step 4:** `system.view_refreshes` table से पता चलता है कि last refresh कब successful हुआ, अगला कब होगा — यानी ये MV का "health check" है।
- **Step 5:** `ALTER TABLE ... MODIFY REFRESH` से schedule बदल सकते हैं (जैसे हर 5 minute में)।

**Definition:** Refreshable MV मतलब — पूरी query को periodically (समय-समय पे) पूरा फिर से चलाना, incremental update नहीं, इसलिए heavy JOINs/aggregations के लिए बेहतर है।

---

## Part C — Cascading Materialized Views
**Goal:** Rollups को **chain (जंजीर)** में जोड़ना — daily → monthly → yearly, बिना raw `orders` table को बार-बार छुए।

- **Step 1:** `monthly_revenue` table बनाते हैं (engine `AggregatingMergeTree`), और उसकी MV का source है `daily_revenue` (Part A का target table) — ना कि `orders` सीधे। जब भी `daily_revenue_mv` कोई नया block `daily_revenue` में डालती है, ये monthly MV अपने-आप automatically चल जाती है।
  - यहाँ `sumState()` use होता है — ये final number नहीं, बल्कि एक **partial aggregate state** (आधा-अधूरा calculation) store करता है, ताकि आगे इसे और merge किया जा सके।
- **Step 2:** `yearly_revenue` table (`SummingMergeTree`) बनाते हैं, जिसकी MV source है `monthly_revenue`। यहाँ `sumMerge()` use करना ज़रूरी है क्योंकि आने वाला data पहले से ही एक partial state है (state, final number नहीं) — इसे पहले merge करना पड़ता है, फिर आगे aggregate करते हैं।
- **Step 3:** सिर्फ एक `INSERT INTO orders` से पूरी chain अपने-आप चल जाती है:
  `orders → daily_revenue_mv → daily_revenue → monthly_revenue_mv → monthly_revenue → yearly_revenue_mv → yearly_revenue`
- **Step 4:** हर स्तर (level) — daily, monthly, yearly — पे data verify करते हैं कि सही तरीके से ऊपर की तरफ़ aggregate हो रहा है।

**Definition:** Cascading MV मतलब — एक MV के output को अगली MV का input बना देना, ताकि complex multi-level rollups (जैसे daily→monthly→yearly) automatically maintain हो जाएँ, बिना raw data को बार-बार दोबारा प्रोसेस किए।

**`sumState` vs `sumMerge` का फ़र्क़ (fark) समझना ज़रूरी है:**
- `sumState` — एक **आधा-पका (half-cooked)** result बनाता है जिसे आगे और combine किया जा सकता है।
- `sumMerge` — उन आधे-पके states को लेकर एक **final, पूरा (complete)** number देता है।

---

## 4. Cleanup
आखिर में `DROP DATABASE shop` से पूरा database (सारे tables और MVs समेत) delete कर देते हैं, ताकि lab का leftover data machine पे ना रहे।

---

**Overall summary (एक line में):** ये lab सिखाता है तीन तरह की Materialized Views — **Incremental** (हर insert पे तुरंत react करने वाली, हल्के aggregations के लिए), **Refreshable** (भारी JOINs के लिए, timer पे पूरी query दोबारा चलाने वाली), और **Cascading** (एक MV के output को दूसरी MV का input बनाकर multi-level rollups automatically बनाना) — ताकि real-time analytics बिना raw data को बार-बार scan किए मिल सके।
