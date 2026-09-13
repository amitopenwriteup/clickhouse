# ClickHouse में Table Shards और Replicas — Hinglish में

**Note:** ये concept **ClickHouse Cloud** पे apply नहीं होता — वहाँ इसकी जगह Parallel Replicas और object storage काम करते हैं। ये सिर्फ traditional self-managed (shared-nothing) clusters के लिए है।

## Shards क्या हैं?

जब **data बहुत ज़्यादा हो जाए** एक server पे रखने के लिए, या **एक server data process करने में बहुत slow** हो जाए — तब **sharding** करते हैं।

**Idea:** पूरे data को कई ClickHouse servers में **बाँट (split) दो**। हर server के पास data का सिर्फ एक हिस्सा (subset) होता है — इसी हिस्से को **shard** कहते हैं।

- हर shard खुद एक normal ClickHouse table की तरह काम करता है — आप उसे अकेले भी query कर सकते हो (लेकिन तब सिर्फ उसी shard का data मिलेगा, पूरा data नहीं)।
- पूरे data का "एक साथ" view पाने के लिए एक **Distributed table** बनाई जाती है। ये table खुद कोई data store नहीं करती — बस:
  - **SELECT** queries को सभी shards तक forward करती है
  - **INSERT** को सही shard पे route करती है ताकि data evenly बँटा रहे

## Distributed Table कैसे बनाएं

```sql
CREATE TABLE uk.uk_price_paid_simple_dist ON CLUSTER test_cluster
(
    date Date,
    town LowCardinality(String),
    street LowCardinality(String),
    price UInt32
)
ENGINE = Distributed('test_cluster', 'uk', 'uk_price_paid_simple', rand())
```

- **ON CLUSTER** — इससे ये DDL statement "distributed DDL" बन जाता है, मतलब ये command cluster के सारे servers पे अपने आप चल जाएगा। इसके लिए एक **Keeper** component भी चाहिए होता है (cluster की coordination के लिए)।
- **Distributed engine के parameters:**
  1. Cluster का नाम (`test_cluster`)
  2. Database का नाम (`uk`)
  3. असली sharded table का नाम (`uk_price_paid_simple`)
  4. **Sharding key** — यहाँ `rand()` use किया है, यानी हर row randomly किसी भी shard में डाल दो। लेकिन ये कोई भी expression हो सकता है, अपने use-case के हिसाब से (जैसे किसी column के आधार पे sharding करना)।

## INSERT कैसे Route होता है

1. एक INSERT (एक row के साथ) distributed table को भेजा जाता है — direct या load balancer के ज़रिए।
2. ClickHouse उस row के लिए **sharding key** calculate करता है (यहाँ `rand()`), फिर उसे **shard servers की संख्या से modulo** करता है — इससे पता चलता है कि row किस server (shard) में जाएगी।
3. Row उस सही shard में insert हो जाती है।

**सीधी भाषा में:** sharding key एक तरह की "पर्ची" है जो decide करती है कि हर row कौनसे घर (shard) में जाएगी।

## SELECT कैसे Forward होता है

1. एक SELECT aggregation query distributed table को भेजी जाती है।
2. Distributed table उस query को **सभी shards** तक forward करती है — हर server अपने पास मौजूद data पे **parallel में** अपना local result निकालता है।
3. जिस server ने originally query receive की थी, वो सारे local results collect करता है, उन्हें **merge** करके final (global) result बनाता है, और वापस user को भेज देता है।

**फायदा:** चूँकि सारे servers parallel में काम करते हैं, बड़े data पे भी query fast चलती है।

## Replicas क्या हैं?

Replication का मकसद है — **data की सुरक्षा (integrity)** और **failover** (अगर कोई server crash हो जाए तो भी data available रहे)।

- हर **shard** के multiple **copies (replicas)** बनाई जाती हैं — अलग-अलग servers पे।
- Writes (INSERT) किसी भी replica पे किए जा सकते हैं (सीधे या distributed table के ज़रिए) — बाकी replicas में changes **automatically propagate** (फैल) हो जाते हैं।
- अगर कोई server fail हो जाए, data बाकी replicas पे उपलब्ध रहता है। और जब वो server ठीक हो जाए, वो **automatically sync** होकर up-to-date हो जाता है।
- इसके लिए भी **Keeper** component ज़रूरी है।

### Example: 6 Servers, 2 Shards, हर Shard के 3 Replicas

Query processing लगभग वैसे ही काम करती है जैसे बिना replicas वाले setup में — बस फर्क इतना है कि **हर shard से सिर्फ एक replica** query execute करता है (सारे replicas नहीं)।

**Bonus फायदा:** Replicas सिर्फ data की सुरक्षा नहीं करते — ये **query throughput भी बढ़ाते हैं**, क्योंकि अलग-अलग replicas पे कई queries parallel में चल सकती हैं।

**Query Flow (Replicas के साथ):**
1. Query distributed table को भेजी जाती है (direct या load balancer से)।
2. Distributed table हर shard में से **एक replica चुनकर** query forward करती है — हर चुना हुआ replica अपना local result parallel में निकालता है।
3. बाकी सब वैसे ही होता है जैसे बिना replicas वाले setup में (collect → merge → return)।

**Default Behavior:** ClickHouse by default एक **local replica** को prefer करता है (अगर available हो) — इसे `prefer_localhost_replica` setting control करती है। चाहें तो और भी load-balancing strategies use कर सकते हैं (`load_balancing` setting से)।

## एक-लाइन Summary

- **Shard** = data का एक हिस्सा, अलग server पे रखा गया — scale-out के लिए (data बड़ा होने पे बाँटना)
- **Replica** = किसी shard की copy, अलग server पे — reliability और parallel query performance के लिए
- **Distributed table** = ऊपर से एक unified view देने वाली "virtual" table, जो असली data कहीं और (shards में) रखा हुआ होता है
