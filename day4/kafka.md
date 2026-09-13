# Kafka Table Engine — ClickHouse में Hinglish समझाओ

## ये है क्या?

Kafka table engine ClickHouse को सीधे Apache Kafka से जोड़ने का तरीका है। इससे आप Kafka topics को **publish** (data भेजना) या **subscribe** (data पढ़ना) कर सकते हो, fault-tolerant तरीके से data store कर सकते हो, और streams को जैसे-जैसे data आता है वैसे-वैसे process कर सकते हो।

**Tip:** अगर आप ClickHouse Cloud पे हो, तो ClickHouse recommend करता है **ClickPipes** use करने की — वो private networks, independent scaling, और monitoring की सुविधा देता है, Kafka engine के मुकाबले ज़्यादा managed तरीका है।

## Table कैसे बनाएं

```sql
CREATE TABLE queue
(
    timestamp UInt64,
    level String,
    message String
)
ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');
```

### ज़रूरी (Required) Parameters
- **kafka_broker_list** — Kafka brokers की list (जैसे `localhost:9092`)
- **kafka_topic_list** — किन-किन topics को पढ़ना है, उनकी list
- **kafka_group_name** — Consumer group का नाम। अगर आप नहीं चाहते कि message duplicate हो cluster में, तो हर जगह same group name use करो।
- **kafka_format** — Message किस format में है (जैसे `JSONEachRow`)

### कुछ ज़रूरी Optional Parameters
- **kafka_num_consumers** — एक table के लिए कितने consumers चाहिए (default 1)। ज़्यादा throughput चाहिए तो बढ़ाओ, लेकिन ये Kafka topic में partitions से ज़्यादा नहीं होना चाहिए।
- **kafka_max_block_size** — एक बार में poll करने पे कितने messages आएंगे
- **kafka_skip_broken_messages** — अगर कोई message parse नहीं हो पा रहा (schema मैच नहीं कर रहा), तो कितने ऐसे broken messages को skip कर देना है बिना error दिए
- **kafka_thread_per_consumer** — हर consumer को अपना अलग thread देना है या नहीं (default: सब मिलकर एक साथ काम करते हैं)
- **kafka_handle_error_mode** — Error आने पे क्या करना है: `default` (exception throw करो), `stream` (error को अलग columns `_error` और `_raw_message` में save करो), `dead_letter_queue` (error data को एक अलग system table में डालो)

## बहुत ज़रूरी बात — सीधे SELECT मत करो!

`SELECT * FROM queue` सिर्फ **debugging** के लिए है — क्योंकि हर message सिर्फ **एक बार** पढ़ा जा सकता है (जैसे ही आप SELECT करते हो, वो message consume हो जाता है, फिर दोबारा नहीं मिलेगा)।

**असली तरीका है — Materialized View बनाओ:**

1. Kafka engine से एक consumer table बनाओ (जो data stream की तरह काम करेगा)
2. एक normal table बनाओ जिसमें असली data रखना है (जैसे MergeTree)
3. एक **Materialized View** बनाओ जो Kafka table से data उठाकर असली table में डालता रहे

```sql
CREATE TABLE queue (
    timestamp UInt64,
    level String,
    message String
) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

CREATE TABLE daily (
    day Date,
    level String,
    total UInt64
) ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(day)
ORDER BY (day, level);

CREATE MATERIALIZED VIEW consumer TO daily
    AS SELECT toDate(toDateTime(timestamp)) AS day, level, count() AS total
    FROM queue GROUP BY day, level;
```

जैसे ही Materialized View, Kafka table से जुड़ता है, वो background में automatically data collect करना शुरू कर देता है — और आपको हमेशा `daily` table से query करना है, `queue` table से नहीं।

**एक Kafka table पे कई Materialized Views** भी बना सकते हो — हर एक अलग-अलग तरीके से data process करके अलग-अलग tables में डाल सकता है (जैसे एक raw detail के साथ, दूसरा aggregated summary के साथ)।

## Data कैसे Flush होता है (Batching internally)

Messages को छोटे-छोटे blocks में group किया जाता है — block का size `max_insert_block_size` setting से control होता है। अगर पूरा block बनने में `stream_flush_interval_ms` से ज़्यादा time लग गया, तो जितना data आया है उतना ही flush कर दिया जाएगा — पूरे block का इंतज़ार नहीं करेगा।

(ये बिल्कुल वही concept है जो हमने पहले "batch vs streaming" में देखा था — Kafka engine internally ही buffering/batching manage कर लेता है ताकि छोटे-छोटे parts ना बनें।)

## Consumption रोकनी हो तो

```sql
DETACH TABLE consumer;   -- रोक दो
ATTACH TABLE consumer;   -- फिर से शुरू करो
```

अगर target table को `ALTER` करना है, तो पहले Materialized View को detach करना recommended है — ताकि target table और view के बीच mismatch ना हो।

## Virtual Columns (extra जानकारी हर message के साथ मिलती है)

- **_topic** — किस topic से message आया
- **_key** — Message की key
- **_offset** — Message का offset (position)
- **_timestamp** — Message का timestamp
- **_partition** — किस Kafka partition से आया

अगर `kafka_handle_error_mode='stream'` set है, तो parsing fail होने पे `_raw_message` और `_error` columns में raw data और error message मिलेगा।

## Data Durability — ज़रूरी Warning

Kafka engine कभी-कभी data **silently lose** कर सकता है अगर OS का page cache disk पे लिखे जाने से पहले crash हो जाए (जैसे power loss)। सामान्य process kill से ये problem नहीं आती (OS खुद बाद में लिख देता है), लेकिन hardware-level crash से आ सकती है।

**Fix:** Target MergeTree tables पे ये settings ON करो:
```sql
SETTINGS fsync_after_insert = 1, fsync_part_directory = 1;
```
ये हर उस MergeTree table पे लगाना ज़रूरी है जहाँ data cascaded होकर जाता है (materialized view की chain में भी)। ध्यान रहे, अगर बीच में कोई `Distributed` table है और वो async मोड में insert कर रही है, तो उसके लिए अलग से durability settings लगानी पड़ेंगी।

## एक-लाइन Summary

Kafka table engine = ClickHouse के अंदर एक "live pipe" जो Kafka से data खींचता रहता है। इसे सीधे query मत करो — इसके ऊपर एक Materialized View लगाओ जो data को असली (MergeTree जैसी) table में push करता रहे, और production में durability settings (`fsync_after_insert`) ज़रूर चेक करो अगर data loss afford नहीं कर सकते।
