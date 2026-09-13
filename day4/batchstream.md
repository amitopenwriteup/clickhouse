# ClickHouse में Batch vs Streaming Ingestion — आसान भाषा में

सोचो ClickHouse एक लाइब्रेरियन (librarian) है जिसे नई किताबें shelves पे रखनी होती हैं। जब भी आप किताबें देते हो file करने के लिए, लाइब्रेरियन को एक fixed effort लगता है — shelf तक चलना, sort करना, check करना — चाहे आप 1 किताब दो या 10,000 किताबें एक साथ।

## Batch Ingestion

**क्या है:** आप पहले बहुत सारी rows इकट्ठा (collect) करते हो, फिर सबको एक साथ एक INSERT में ClickHouse को भेज देते हो।

```sql
INSERT INTO events VALUES (...), (...), (...), ... -- हज़ारों rows
```

**क्यों ज़रूरी है:** ClickHouse का storage engine (MergeTree) हर INSERT को एक नया **"part"** (disk पे files का एक छोटा chunk) बनाकर लिखता है। Background में ये parts धीरे-धीरे merge होते रहते हैं। अगर आप बहुत सारे *छोटे-छोटे* inserts बहुत तेज़ी से भेजो, तो merge होने से पहले ही parts की बाढ़ आ जाती है — इससे मशहूर **"Too many parts"** error आती है और सब कुछ धीमा हो जाता है।

**Rule of thumb:** एक insert में **~10,000 से 100,000+ rows** होना ideal है। कम लेकिन बड़े inserts = ClickHouse खुश रहेगा।

## Streaming Ingestion

**क्या है:** Data लगातार (continuously) आता रहता है — एक-एक row करके या छोटे-छोटे groups में — अक्सर Kafka, application events, IoT sensors, logs वगैरह से, और आप चाहते हो कि वो लगभग तुरंत (immediately) query करने लायक हो जाए।

**Problem क्या है:** अगर आप हर event के लिए एक अलग INSERT करो, तो असली scale पे तुरंत "too many parts" वाली problem आ जाएगी।

**ClickHouse इसे कैसे solve करता है:**

1. **पहले data को buffer करो**, फिर उसे batch की तरह flush करो। आम tools/patterns:
   - **Kafka table engine** — ClickHouse खुद Kafka से data खींचता है और एक materialized view के ज़रिए internally batch करके insert करता है।
   - **Buffer table engine** — एक in-memory buffer table जो rows को इकट्ठा करता रहता है और समय-समय पे असली table में flush करता है।
   - **Client-side buffering** — tools जैसे `clickhouse-client` का async insert mode, या ingestion tools (Vector, Fluentd) कुछ समय (जैसे 1 second) या कुछ rows तक data इकट्ठा करके भेजते हैं।

2. **Async inserts** (ClickHouse का built-in feature) — आप बहुत सारे छोटे inserts भेज सकते हो और ClickHouse को बोल सकते हो: "इन्हें server-side पे buffer कर लो और अपने आप combine करके एक part बना दो।"
   ```sql
   SET async_insert = 1;
   SET wait_for_async_insert = 1;
   ```

## आसान Analogy (उदाहरण से समझो)

| | Batch | Streaming |
|---|---|---|
| जैसा है... | हफ्ते भर का एक साथ बड़ा grocery shopping करना | दिन भर fridge से बार-बार snacks निकालना |
| Efficiency | बहुत efficient, कम overhead | Buffering ना हो तो गड़बड़ हो जाती है |
| Latency (देरी) | Data सिर्फ पूरा batch लगने के बाद दिखता है | Data जल्दी दिखता है (near real-time) |
| गलत तरीके से करने का risk | कुछ खास नहीं — ये "safe" तरीका है | "Too many parts" errors, धीमे merges, ज़्यादा CPU |

## आखिर में — Bottom Line

- **Batch = default best practice।** अगर pipeline आपके control में है, तो हमेशा कम लेकिन बड़े inserts को prefer करो।
- **Streaming = real-time use cases के लिए ज़रूरी है**, लेकिन उसमें एक buffering layer (Kafka engine, Buffer table, या async_insert) जोड़ना ही पड़ेगा — ताकि ClickHouse को अंदर ही अंदर efficient batches में data मिले, भले ही user को लगे कि data लगातार (continuously) आ रहा है।
