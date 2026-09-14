# ClickHouse Performance Lab — हिंग्लिश में समझते हैं

## PART 0: SETUP
सबसे पहले हम confirm करते हैं कि ClickHouse server चल रहा है (running है), फिर `clickhouse-client` से अंदर जाते हैं — ये एक interactive shell है जहाँ से हम SQL queries चला सकते हैं। फिर एक अलग database `perflab` बनाते हैं ताकि हमारा test data production data के साथ mix ना हो। Basically ये पूरे lab का **तैयारी** वाला step है।

## PART 1: INDEXING STRATEGIES
यहाँ हम देख रहे हैं कि ClickHouse का **primary index** (जिसे ORDER BY column भी कहते हैं) कैसे काम करता है।
- हम 50 लाख (5 million) rows का एक table बनाते हैं, जिसमें `ORDER BY (event_date, user_id)` है।
- जब हम `event_date` पे filter करते हैं (जो key column है), query बहुत fast होती है क्योंकि ClickHouse को पता है data disk पे किस order में पड़ा है — ये है **sparse primary index** का फायदा।
- जब हम `event_type` पे filter करते हैं (जो key column नहीं है), ClickHouse को **पूरा table scan** करना पड़ता है — बहुत slow।
- Fix करने के लिए हम एक **skip index** (`set` type) add करते हैं `event_type` पे — ये पूरा scan नहीं करता, बल्कि granules (data के chunks) को skip कर देता है जो match नहीं करते। इससे speed improve होती है।

**Definition (परिभाषा):** Index मतलब एक ऐसी structure जो query को बताती है "यहाँ मत देखो, सिर्फ यहाँ देखो" — ताकि disk से कम data पढ़ना पड़े।

## PART 2: QUERY PROFILING & EXPLAIN
ये section है **"query अंदर क्या कर रही है"** समझने के लिए।
- `EXPLAIN PLAN` — query का logical plan दिखाता है (कौनसे steps होंगे: filter, aggregation, etc.)
- `EXPLAIN INDEXES` — बताता है कि index actually use हो रहा है या नहीं, और कितने granules scan हुए vs total granules।
- `EXPLAIN PIPELINE` — physical execution दिखाता है, जैसे कितने parallel threads use हो रहे हैं।
- `EXPLAIN ESTIMATE` — अंदाज़ा (estimate) देता है कि कितनी rows/marks पढ़ी जाएँगी।
- `system.query_log` — एक system table जिसमें हर query का history/record store होता है (कितनी rows पढ़ी, कितना memory use हुआ, कितना time लगा) — ये **debugging और performance tuning** के लिए बहुत उपयोगी है।

**Definition:** EXPLAIN का मतलब है query को actually run किए बिना उसका "X-ray" देखना।

## PART 3: COMPRESSION CODECS & QUERY TUNING
यहाँ हम data को disk पे कैसे store करते हैं उसकी **efficiency** देख रहे हैं।
- पहले current compressed vs uncompressed size check करते हैं।
- फिर एक नया table बनाते हैं जिसमें explicit codecs हैं — जैसे `DoubleDelta` (dates के लिए अच्छा क्योंकि consecutive dates का difference छोटा होता है), `T64` (integers के लिए), और `ZSTD` (general compression)।
- Trade-off ये है: better compression मतलब कम disk space, लेकिन insert/read के time ज़्यादा CPU लगेगा।
- **PREWHERE** एक special ClickHouse feature है — ये पहले हल्के filter (जैसे `event_type`) apply करता है, और सिर्फ जो rows बचती हैं उनके लिए heavy columns (जैसे `payload`) पढ़ता है। इससे unnecessary I/O बच जाता है।

**Definition:** Codec मतलब data को compress/decompress करने का तरीका; PREWHERE मतलब "column पढ़ने से पहले ही filter कर दो।"

## PART 4: ON-PREM CLUSTER SETUP (KEEPER/ZOOKEEPER + 2 SHARDS × 2 REPLICAS)
ये सबसे complex part है। यहाँ हम single-machine पे **4 अलग ClickHouse server processes** चलाते हैं (हर एक अपने अलग port पे), जो मिलके एक "cluster" जैसा simulate करते हैं।

- **Shard** मतलब data को horizontally split करना — जैसे अगर तुम्हारे पास 10 करोड़ rows हैं, तो shard 1 में 5 करोड़ और shard 2 में 5 करोड़ जाएँगी। इससे एक machine पे सारा load नहीं पड़ता।
- **Replica** मतलब एक shard का data दूसरी जगह भी copy रखना — अगर एक replica crash हो जाए तो दूसरा काम करता रहे। ये **high availability** के लिए है।
- **ZooKeeper** एक coordination service है जो बताता है कौनसा node कौनसा data रखता है, और replicas को sync में रखता है।
- हमने ZooKeeper Rocky Linux पे install किया, फिर 4 node configs बनाए — हर एक में अपना port, data path, और `{shard}/{replica}` macros थे।

इस part में हमें बहुत सारी real-world दिक्कतों (issues) का सामना भी करना पड़ा — जैसे port conflicts (Tomcat, Docker के साथ), missing `default` profile, और `include_from`/`incl` mechanism का reliably काम ना करना। इसलिए हमने हर node के config में `<remote_servers>`, `<zookeeper>`, `<distributed_ddl>` सीधे embed कर दिया, किसी include file पे depend नहीं किया।

**Definition:** Cluster मतलब multiple servers जो मिल-जुलके एक बड़ी database जैसा काम करते हैं — data split (sharding) और duplicate (replication) दोनों एक साथ।

## PART 5: SHARDING & REPLICATION IN ACTION
अब actually देखते हैं कि ये cluster काम कर रहा है या नहीं:
- `ReplicatedMergeTree` engine वाला table बनाते हैं `ON CLUSTER` keyword के साथ — ये एक ही command से सभी 4 nodes पे table create कर देता है।
- `Distributed` engine वाला table बनाते हैं — ये एक "virtual" table है जो query को सही shard पे route कर देता है automatically।
- Data insert करके check करते हैं कि total count सही है, और ये confirm करते हैं कि data दोनों shards में बंट गया (split हो गया), और हर shard का data उसके दोनों replicas पे present है।

**Definition:** Distributed table मतलब एक "façade" (सामने का मुखौटा) जो पूरे cluster को एक single table जैसा दिखाता है, लेकिन पीछे से data अलग-अलग nodes पे होता है।

## PART 6: FAILOVER DRILL
ये test करता है कि अगर एक node **मर जाए** (crash हो जाए), तो system काम करता रहेगा या नहीं।
- हम node2 को जान-बूझकर kill करते हैं।
- फिर Distributed table से query करते हैं — count फिर भी सही आता है क्योंकि shard 1 का data node1 (surviving replica) पे available है।
- नया data भी insert कर सकते हैं जबकि node2 down है — cluster लिखना (writing) बंद नहीं करता।
- जब node2 वापस start होता है, वो ZooKeeper के replication log से "catch up" कर लेता है, मतलब missed changes automatically sync हो जाते हैं।

**Definition:** Failover मतलब जब एक component fail हो जाए, तो system automatically दूसरे available component पे switch हो जाए, बिना downtime के।

## WRAP-UP / CLEANUP
आखिर में हम verify करते हैं कि क्या-क्या बना (`SHOW TABLES`, `system.replicas`), और फिर सब कुछ clean कर देते हैं — test databases drop करते हैं, और अगर ज़रूरत हो तो पूरा simulated cluster (`/opt/ch-cluster`) delete कर देते हैं, ताकि machine साफ रह जाए।

---

**Overall summary (एक line में):** ये lab तुम्हें सिखाता है कि एक single ClickHouse instance को **fast** कैसे बनाएँ (indexing, EXPLAIN, compression, PREWHERE), और फिर उसे एक **fault-tolerant, scalable cluster** में कैसे convert करें (sharding + replication + ZooKeeper coordination), और real production में आने वाली दिक्कतों को कैसे debug करें।
