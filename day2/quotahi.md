# ClickHouse Quotas & Resource Management - समझाइए (Hinglish)

## Theory समझें

**QUOTAS** का मतलब है resource consumption को limit करना - जैसे queries की संख्या, errors, execution time, rows read, bytes read - किसी user या role के लिए एक time interval के अंदर। यह cluster को runaway या abusive queries से बचाता है (protect करता है)।

**SETTINGS** के through भी resource limits enforce की जा सकती हैं - जैसे `max_memory_usage`, `max_execution_time`, `max_threads`। यह settings किसी user, role, या profile पे apply होती हैं।

**Settings PROFILES** resource-related settings को group करते हैं - बिल्कुल वैसे ही जैसे roles privileges को group करते हैं। Profiles को users को assign किया जा सकता है।

Newer ClickHouse versions में **Workload/Resource scheduling** (CPU/IO scheduling, backpressure) भी available है जो multi-tenant setups के लिए ज्यादा fine-grained control देता है।

---

## Hands-on Steps की explanation

**Step 1:** पहले `bob` नाम का user create करो (अगर Lab 1.1 में पहले से नहीं बना है)। Password के साथ `sha256_password` use करके authentication set होती है। `exit` command shell पे वापस ले जाता है user create होने के बाद।

**Step 2:** एक **settings profile** बनाओ नाम `limited_profile` का, जिसमें resource caps set हैं - memory usage ~1GB तक limit, और query runtime 30 seconds तक। `READONLY` keyword का मतलब है कि यह settings user खुद change नहीं कर सकता।

**Step 3:** उस profile को `bob` को assign करो `ALTER USER` command से। अब bob की हर query automatically इन hard limits को inherit करेगी (follow करेगी)।

**Step 4:** एक **quota** create करो नाम `daily_quota` का, जो 1 दिन (24 hours) के interval में bob की query volume limit करेगा - max 1000 queries, max 50 errors, और max 1 hour का total execution time। `exit` admin session को close करता है ताकि तुम bob के रूप में switch कर सको।

**Step 5:** अब `bob` के user से connect करो और limits test करो। `SELECT sleep(35)` command 35 seconds तक sleep करेगी, जो `max_execution_time = 30` से ज्यादा है - इसलिए यह query terminate हो जानी चाहिए (error के साथ, quota/settings violation की वजह से)।

**Step 6:** `SHOW QUOTA` command चलाओ (जब तुम अभी भी bob के रूप में connect हो) - यह दिखाता है कि bob ने अब तक कितना quota consume किया है (kितनी queries, कितना समय, वगैरह)। उसके बाद `exit` से shell पे वापस आ जाओ।

**Step 7:** वापस admin के रूप में connect करो और `system.query_log` table को query करो सबसे ज्यादा duration वाली top 10 queries देखने के लिए। इससे तुम्हें पता चलता है कि cluster पे सबसे ज्यादा resource-heavy (expensive) queries कौन सी चल रही हैं, ताकि तुम उन्हें identify और optimize कर सको।
