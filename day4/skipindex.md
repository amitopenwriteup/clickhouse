# ClickHouse Skip Indexes — Detail mein, आसान भाषा में

## पहले समझो असली Problem क्या है

सोचो आपके पास एक बहुत बड़ी किताब है — 10 करोड़ पन्नों की। हर पन्ने पे एक row का data लिखा है।

अगर आप किताब में कुछ ढूंढना चाहते हो जो **क्रम से (sorted) लगा है** — जैसे "page number 5000 से 6000 के बीच क्या है" — तो सीधे उन पन्नों पे जा सकते हो। ये है **primary key** का फायदा।

लेकिन अगर आप ढूंढ रहे हो "किस-किस पन्ने पे 'customer_id = 1001' लिखा है" — और ये customer_id पूरी किताब में बिखरा (scattered) पड़ा है — तो आपको **हर एक पन्ना खोलकर चेक करना पड़ेगा।** यही है slow query की वजह।

Traditional databases (जैसे MySQL) में इसका solution है **secondary index** — एक अलग सूची जो exactly बताती है "customer_id = 1001 पन्ना नंबर 45, 892, 50123... पे है।" लेकिन ClickHouse में ये तरीका काम नहीं करता, क्योंकि ClickHouse डेटा को **column के हिसाब से** स्टोर करता है, row के हिसाब से नहीं। तो "पन्ना नंबर" जैसी individual row-location कॉन्सेप्ट ही exist नहीं करती।

## Skip Index का Idea — एक Analogy से समझो

सोचो आपकी किताब के 10 करोड़ पन्ने, 8192-8192 पन्नों के **bundles (गड्डियों)** में बंधे हैं। अब हर bundle के ऊपर एक chit (पर्ची) लगा दो जिसमें लिखा हो:

> "इस bundle में customer_id के values 1 से 500 तक हैं" (अगर minmax index है)

या

> "इस bundle में सिर्फ ये customer_ids मौजूद हैं: 12, 45, 78, 99" (अगर set index है)

अब जब आप customer_id = 1001 ढूंढ रहे हो, तो आपको हर bundle खोलने की ज़रूरत नहीं — बस chit पढ़ो। अगर chit कहती है "1001 यहाँ नहीं हो सकता", तो पूरा bundle **skip** कर दो, बिना खोले। सिर्फ उन्हीं bundles को खोलो जिनकी chit कहती है "हो सकता है यहाँ हो।"

**यही है Skip Index — एक chit/पर्ची जो हर block (bundle) के ऊपर लगी होती है, ताकि पूरा block बिना पढ़े skip किया जा सके।**

ज़रूरी बात: chit कभी गलत "yes, ये block ज़रूर स्किप करो" नहीं बोलेगी। लेकिन कभी-कभी वो कहेगी "शायद यहाँ हो सकता है" जबकि असल में नहीं है (false positive) — तब आपको बेकार में वो block खोलना पड़ेगा, पर गलत result कभी नहीं मिलेगा।

## Real Example (Docs से)

```sql
CREATE TABLE skip_table
(
  my_key UInt64,
  my_value UInt64
)
ENGINE MergeTree primary key my_key
SETTINGS index_granularity=8192;

INSERT INTO skip_table SELECT number, intDiv(number,4096) FROM numbers(100000000);
```

अब ये query चलाओ (बिना index के):
```sql
SELECT * FROM skip_table WHERE my_value IN (125, 700)
```
**Result:** पूरे 10 करोड़ rows scan हुए — 0.079 sec लगे।

अब एक skip index add करो:
```sql
ALTER TABLE skip_table ADD INDEX vix my_value TYPE set(100) GRANULARITY 2;
ALTER TABLE skip_table MATERIALIZE INDEX vix;  -- पुराने data पे भी लगाओ
```

अब वही query फिर चलाओ:
**Result:** सिर्फ 32,768 rows scan हुए (यानी सिर्फ 4 blocks खुले) — 0.051 sec लगे। बाकी सब blocks skip हो गए क्योंकि उनकी "chit" में 125 या 700 था ही नहीं।

## Index कैसे बनता है — 4 चीज़ें चाहिए

| हिस्सा | मतलब |
|---|---|
| **Name** | Index का नाम (जैसे `vix`) — बाद में drop/materialize करने के लिए चाहिए |
| **Expression** | किस column या calculation पे index लगाना है |
| **TYPE** | कौनसा तरीका इस्तेमाल होगा (minmax, set, bloom_filter, वगैरह) |
| **GRANULARITY** | कितने primary-index granules मिलकर एक "chit वाला block" बनेंगे। जैसे अगर primary granularity 8192 rows है और आपने GRANULARITY 4 दिया, तो एक skip-index block = 8192 × 4 = 32768 rows |

Index बनाने पे disk पे 2 नई files बनती हैं हर data part में:
- `.idx` file — chit के actual values (ordered)
- `.mrk2` file — ये chit किस data location से जुड़ी है, उसका offset

## Index के Types — डिटेल में

### 1. **minmax** (सबसे हल्का/सस्ता)
हर block का सिर्फ **minimum और maximum value** स्टोर करता है।

**कब use करें:** जब column की values, primary key के साथ **loosely sorted/correlated** हों। जैसे — अगर आपका primary key `timestamp` है, और आप `temperature` column पे index लगाना चाहते हो, और temperature धीरे-धीरे दिन भर बढ़ती-घटती है (अचानक random नहीं बदलती), तो हर time-block में temperature की एक tight range होगी — minmax बहुत अच्छा काम करेगा।

**कब काम नहीं करेगा:** अगर column की values पूरे block में बहुत ज़्यादा फैली (spread out) हों — तब min-max range बहुत wide हो जाएगी और लगभग हर block "matching हो सकता है" बोल देगा, यानी कुछ भी skip नहीं होगा।

### 2. **set**
हर block में मौजूद **सारे distinct (अलग-अलग) values** की एक सूची स्टोर करता है (एक limit तक, जैसे max_size=100)।

**कब use करें:** जब एक block के अंदर values की **variety कम** हो (जैसे एक ही block में सिर्फ 5-10 अलग values हों), लेकिन पूरे table में overall variety बहुत ज़्यादा हो।

**Warning:** अगर एक block में values की variety बहुत ज़्यादा है (max_size से ज़्यादा distinct values), तो ClickHouse उस block की सूची खाली छोड़ देगा — मतलब index का कोई फायदा नहीं होगा, बल्कि उल्टा overhead बढ़ेगा।

### 3. **text** (Full-text search के लिए)
ये एक असली **inverted index** है — जैसे Google search के पीछे काम करने वाला तरीका। Words/phrases को tokenize (टुकड़ों में तोड़) करके index करता है।

**कब use करें:** जब आपको बड़े text columns में words या phrases ढूंढने हों — functions जैसे `hasAnyToken`, `hasAllTokens` के लिए ये best (recommended) है।

### 4. **Bloom Filter based indexes** (bloom_filter, tokenbf_v1, ngrambf_v1)

**Bloom filter क्या है (सरल भाषा में):** ये एक तरीका है ये चेक करने का "क्या ये value इस set में है?" — बहुत कम जगह (space) लेकर। लेकिन इसमें कभी-कभी **false positive** आ सकता है (कहेगा "हाँ हो सकता है" जबकि असल में नहीं है) — पर कभी false negative नहीं आता (अगर value वाकई है तो हमेशा "हाँ" बोलेगा)।

- **bloom_filter** — basic version, सिर्फ false-positive rate (0 से 1 के बीच) set करना होता है
- **tokenbf_v1** *(deprecated)* — strings को words में तोड़कर index करता है (जैसे "full text search" → tokens: full, text, search)
- **ngrambf_v1** *(deprecated)* — strings को fixed-length character chunks (n-grams) में तोड़ता है — भाषाओं के लिए अच्छा है जहाँ words के बीच space नहीं होता (जैसे Chinese)

**कब use करें:** Arrays, maps, या strings के अंदर values ढूंढने के लिए — जहाँ हर block में बहुत सारे unique values हो सकते हैं।

## सबसे ज़रूरी बात — Skip Index हमेशा फायदेमंद नहीं होता!

ये सबसे बड़ी गलतफहमी है जो लोग करते हैं। Traditional databases की आदत की वजह से लोग सोचते हैं "index लगाओ, query fast हो जाएगी" — पर ClickHouse में ऐसा नहीं है।

**याद रखो:** अगर एक block में सिर्फ **एक बार भी** target value मौजूद है, तो पूरा block पढ़ना पड़ेगा — और index लगाने का खर्चा (cost) बेकार गया।

### Example जहाँ Skip Index काम नहीं करेगा:
मान लो primary key `timestamp` है, और आप `visitor_id` column पे index लगाते हो। लेकिन visitor_id की values हर block में randomly बिखरी हुई हैं (कोई correlation नहीं timestamp के साथ)। अगर आप `visitor_id = 1001` ढूंढो, तो हर 8192-row वाले block में लगभग हमेशा कोई ना कोई ऐसी row मिल ही जाएगी — तो कोई भी block skip नहीं होगा, और index का पूरा benefit ज़ीरो हो जाएगा (बल्कि extra cost लगेगा)।

### Skip Index कब सच में फायदा देगा:
1. **Strong correlation हो primary key के साथ** — जैसे अगर आप insert करते समय data को इस तरह group करें कि एक particular `site_id` की सारी entries साथ-साथ आएं, तो हर block में सिर्फ 1-2 site_ids होंगी — तब site_id पे index लगाना बहुत फायदेमंद होगा।

2. **High-cardinality लेकिन rare values** — जैसे किसी observability/monitoring system में error codes। ज़्यादातर requests normal होती हैं (कोई error नहीं), लेकिन कभी-कभार कोई खास error code आता है। ऐसे columns पे `set` index लगाने से error-वाली queries बहुत तेज़ हो जाती हैं, क्योंकि ज़्यादातर blocks में error होता ही नहीं और वो सारे skip हो जाते हैं।

## दो Important Settings

- **use_skip_indexes** (default: 1/ON) — अगर किसी query के लिए पता है कि skip index से फायदा नहीं होगा, तो इसे 0 करके index को ignore किया जा सकता है (उस specific query के लिए)।
- **force_data_skipping_indices** — अगर आप चाहते हो कि कोई भी query बिना किसी specific index इस्तेमाल किए ना चले (ताकि गलती से कोई बहुत भारी/expensive query ना चल जाए), तो इस setting से उसे enforce कर सकते हो।

## आखिर में — Best Practice क्या है?

1. पहले primary key (ORDER BY) design सही करो — ये सबसे बड़ा performance factor है
2. Skip index को **last resort** समझो, हर column पे मत लगाओ
3. लगाने से पहले सोचो: "क्या इस column की values, primary key के साथ किसी तरह group/cluster हो रही हैं?"
4. हमेशा **real data पे test करो** — अलग-अलग TYPE और GRANULARITY try करके देखो, क्योंकि behavior predict करना मुश्किल होता है
5. `send_logs_level='trace'` set करके देख सकते हो कि asal mein कितने granules drop (skip) हुए — इससे पता चलेगा index काम कर भी रहा है या नहीं
