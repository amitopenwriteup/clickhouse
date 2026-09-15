# ClickHouse Lab — Sections 3, 4, 5 (Hinglish, हिंदी शब्द देवनागरी में)

## Section 3 — Compression Codecs & Query Tuning

**क्या है ये:** ClickHouse एक column store है — मतलब हर column डिस्क पे अलग से store और compress होता है। Codec decide करता है कि column का data डिस्क पे जाने से पहले कैसे transform और compress होगा।

- **LZ4** — default codec, fast है लेकिन compression ratio इतना ज़्यादा नहीं।
- **DoubleDelta** — जब values monotonically बढ़ रही हों (जैसे dates, timestamps), तब consecutive differences का difference store करता है — बहुत compact हो जाता है।
- **T64** — 64-bit integers को transpose करता है ताकि repeated bit patterns better compress हों — हमने `user_id` पे इस्तेमाल किया।
- **ZSTD(3)** — LZ4 से strong compressor, better ratio देता है लेकिन CPU ज़्यादा खाता है।

**हमने lab में क्या किया:** `events_zstd` table बनाया explicit codecs के साथ, same 5M rows load किए, और `system.columns` से `data_compressed_bytes` compare किया दोनों tables का। **Result:** size छोटा हो गया, लेकिन trade-off ये है कि insert/read पे CPU cost बढ़ जाता है — ये एक measured trade-off है, मुफ़्त में कुछ नहीं मिलता।

### Query Tuning — कम डेटा पढ़ना
- **Column pruning:** `count()` लगभग कुछ नहीं पढ़ता, `SELECT *` हर column पढ़ता है हर matching row के लिए। हमने `read_bytes` compare किया `system.query_log` में दोनों queries का।
- **PREWHERE:** पहले सस्ता condition (`event_type`) check करता है, फिर बचे हुए rows के लिए ही महंगा column (`payload`) पढ़ता है। Plain `WHERE` में ऐसा guarantee नहीं है — वो discard होने वाले rows का भी payload पढ़ सकता है।

---

## Section 4 — On-Prem Cluster Setup (Keeper + 2 Shards × 2 Replicas)

**ZooKeeper/Keeper क्यों चाहिए:** ReplicatedMergeTree tables एक दूसरे को directly data replicate नहीं करते — हर replica अपनी नीयत (intentions) एक shared coordination log में लिखता है। ZooKeeper उस log को hold करता है: कौनसे data parts exist करते हैं, insert किस order में हुए, और किस replica ने क्या apply किया है या नहीं। इसी वजह से कोई node offline हो के वापस आए, तो वो missing operations pull कर लेता है।

ये ON CLUSTER DDL के लिए भी काम आता है — एक statement ZooKeeper में queue होता है और cluster के सारे nodes उसे अपने आप pick करके run करते हैं।

**हमने क्या setup किया:**
- Java 17 + Apache ZooKeeper 3.9.5 install किया Rocky Linux पे।
- `clientPort=2181` set किया, AdminServer disable किया (ताकि port 8080 conflict ना हो)।
- systemd के through चलाया ताकि reboot survive करे।

**Topology:** 4 ClickHouse server processes, एक ही machine पे, अलग-अलग ports पे (production में हर एक अपनी अलग host पे होता):

| Node | TCP Port | Shard | Replica |
|------|----------|-------|---------|
| node1 | 9001 | 1 | replica1 |
| node2 | 9002 | 1 | replica2 |
| node3 | 9003 | 2 | replica1 |
| node4 | 9004 | 2 | replica2 |

एक shared `cluster.xml` (remote_servers + zookeeper block) सभी 4 nodes के config में reuse हुआ — सिर्फ़ जो unique चीज़ है (port, path, `<macros>` shard/replica) वो हर node का अलग है। ये macros ही decide करते हैं कि `ON CLUSTER` DDL कौनसी table copy कहाँ place करेगा।

---

## Section 5 — Sharding & Replication in Action

**दो table engines, दो ज़िम्मेदारियाँ:**

- **ReplicatedMergeTree (`events_local`)** — असली local storage engine, हर node पे। Path `'/clickhouse/tables/{shard}/events_local'` + `{replica}` macro ZooKeeper को बताता है कि कौनसे nodes same shard का identical copy रखते हैं। एक replica पे insert होता है तो ZooKeeper में log होता है, sibling replica वही log पढ़ के apply कर लेता है।

- **Distributed (`events`)** — ख़ुद कोई data store नहीं करता, सिर्फ़ एक routing layer है। INSERT पे हर row एक shard को route होती है (यहाँ `rand()` से), SELECT पे query सारे shards में fan-out होती है और results merge हो के वापस आते हैं।

**हमने क्या verify किया (step-by-step):**

1. **Insert via Distributed table** — 1000 rows insert किए node1 से, `rand()` ने उन्हें दोनों shards में spread कर दिया।
2. **Read via Distributed table** — node1 से `count()` = 1000 आया, मतलब routing layer transparently दोनों shards cover कर रहा है।
3. **Replica का local table directly पढ़ा** — node2 (shard 1 का दूसरा replica) पे `events_local` का count node1 से match हुआ → replication confirm।
4. **हर shard का local table directly पढ़ा** — node1 (shard 1) और node3 (shard 2) के counts मिला के 1000 आए → मतलब data सच में split है, सिर्फ़ duplicate नहीं।

---

**एक लाइन summary:** Section 3 बताता है *कम data कैसे पढ़ें*, Section 4 बताता है *cluster कैसे coordinate होता है* (ZooKeeper के through), और Section 5 में हमने actually prove किया कि data सही से *split* (sharding) और *copy* (replication) दोनों हो रहा है — सिर्फ़ trust नहीं किया, directly check किया।
