# ClickHouse Keeper - Replication & Merge Coordination Lab

**एक सरल हिंदी में समझाया गया गाइड (A Simple Guide in Hinglish)**

---

## Table of Contents

1. [ClickHouse Keeper क्या है? (What is Keeper?)](#section1)
2. [Replication कैसे काम करती है? (How Replication Works)](#section2)
3. [Lab का Architecture (Lab Architecture)](#section3)
4. [Test 1: Basic Replication](#test1)
5. [Test 2: Merge Coordination](#test2)
6. [Test 3: Leader Election](#test3)
7. [Test 4: Replica Catch-Up](#test4)
8. [Keeper में Key Data Structures (Key Data Structures)](#section4)
9. [सीखने योग्य अवधारणाएं (Key Concepts)](#section5)
10. [Lab को कैसे चलाएं? (How to Run the Lab)](#section6)

---

## <a name="section1"></a>1. ClickHouse Keeper क्या है?

### साधारण भाषा में समझो (In Simple Terms)

ClickHouse Keeper एक **database coordinator** है जो कई ClickHouse servers को आपस में communicate करवाता है। यह Apache ZooKeeper की जगह लेता है पर ClickHouse के लिए optimize किया हुआ है।

**एक analogies से समझो:**

```
मान लो तुम्हारे पास 3 दुकानें हैं जो एक ही सामान बेचती हैं।

❌ बिना Keeper के:
- हर दुकान को अपने inventory के लिए फैसले लेने पड़ते हैं
- सभी दुकानें एक जैसा सामान नहीं रख पाती
- merge करते समय सब एक ही काम दोहराते हैं
- ज्यादा मेहनत, कम फायदा

✓ Keeper के साथ:
- एक central authority है जो बताता है क्या करो
- सभी दुकानें एक जैसा data रखती हैं
- merge करते समय सिर्फ एक दुकान काम करती है, बाकी result ले लेते हैं
- efficient और coordinated
```

### Keeper की जिम्मेदारियाँ (Core Responsibilities)

Keeper ये काम करता है:

| **काम** | **हिंदी में** | **क्यों जरूरी है?** |
|---------|--------------|------------------|
| Replica metadata store करना | हर server का IP, port, name सहेजना | ताकि सब एक-दूसरे को पहचान सकें |
| Replication log maintain करना | सभी INSERT/UPDATE/DELETE का record | सब replicas को सही order में apply करने के लिए |
| Leader election करना | जो replica merge करेगा उसे चुनना | ताकि एक ही replica merge करे, बाकी result download करें |
| Part tracking करना | कौन सी data किस replica में है | replica failure को handle करने के लिए |
| Replica failures detect करना | कौन सा server down है | automatic recovery के लिए |

---

## <a name="section2"></a>2. Replication कैसे काम करती है?

### Replication का मतलब (What is Replication?)

**Replication = Data को multiple servers पर copy रखना**

```
Original Server (Primary):
  Data: [Row1, Row2, Row3]
          ↓↓↓ Keeper broadcasts करता है
         
Replica 1:  [Row1, Row2, Row3]
Replica 2:  [Row1, Row2, Row3]
Replica 3:  [Row1, Row2, Row3]

सभी को same data मिलता है!
```

### क्यों Replication चाहिए? (Why Replication?)

**4 बड़े फायदे हैं:**

**1. High Availability (ऊंची उपलब्धता)**
```
अगर एक server खराब हो जाए:
  - Data loss नहीं होता
  - दूसरे servers से data आ सकता है
  - बिना रुकावट continue रहता है
```

**2. Data Consistency (डेटा की एकता)**
```
सभी replicas को same state में रखना:
  - कोई conflicting data नहीं
  - हर जगह से same result आता है
```

**3. Coordinated Merges (आपसी मेल से merge करना)**
```
बजाय सब replicas merge करने के:
  - सिर्फ एक merge करता है (मेहनत कम)
  - बाकी result download कर लेते हैं (तेज)
```

**4. Automatic Catch-Up (अपने आप update हो जाना)**
```
अगर एक replica fail हो जाए:
  - Keeper सभी operations सहेज लेता है
  - Restart होने पर automatically update हो जाता है
  - कोई manual work नहीं
```

### Keeper को Trust करना (Trust the Log)

**याद रखने वाली बात:**
> "Keeper का replication log **source of truth** है। सभी replicas इसी log को follow करते हैं। अगर log में कुछ है, तो सभी replicas को वो करना है।"

---

## <a name="section3"></a>3. Lab का Architecture

### Setup क्या है? (What's the Setup?)

```
┌─────────────────────────────────────────────────┐
│         Docker Network: ch-lab                   │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────────────┐                           │
│  │   Keeper1        │                           │
│  │   Port: 9181     │      Source of Truth      │
│  │ (Coordinator)    │      सभी decisions यहाँ   │
│  └──────────┬───────┘                           │
│             │                                   │
│    ┌────────┼────────┬─────────┐               │
│    │        │        │         │               │
│    ▼        ▼        ▼         ▼               │
│  ┌────┐  ┌────┐  ┌────┐  ┌──────────────────┐ │
│  │ch1 │  │ch2 │  │ch3 │  │ सभी Replicas     │ │
│  │9000│  │9001│  │9002│  │ एक जैसा data     │ │
│  │9000│  │9001│  │9002│  │ रखते हैं         │ │
│  │TCP │  │TCP │  │TCP │  └──────────────────┘ │
│  │9000│  │9000│  │9000│                       │
│  │HTTP│  │HTTP│  │HTTP│  Internal ports       │
│  └────┘  └────┘  └────┘                       │
│                                                  │
│ सभी तीनों replicas:                            │
│ - Keeper से register करते हैं                 │
│ - एक ही table "events" बनाते हैं             │
│ - Shard ID: 01                                 │
│ - Replica names: ch1, ch2, ch3                │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Data Flow (डेटा की यात्रा)

```
अगर ch1 में data डालो:

Step 1: Insert operation
  ch1 में: INSERT INTO events VALUES (...)
  
Step 2: Keeper को inform करो
  ch1 → Keeper: "मैंने ये data add किया"
  Keeper सहेज लेता है

Step 3: Broadcast to others
  Keeper → ch2, ch3: "ये operations करो"
  
Step 4: Apply करो
  ch2: Operation apply करता है
  ch3: Operation apply करता है
  
Result: सभी के पास same data!
```

---

## <a name="test1"></a>4. Test 1: Basic Replication

### क्या test करते हैं?

```
एक row insert करो ch1 में
↓
Keeper broadcast करे
↓
ch2 और ch3 को पता चले
↓
~1 second में सभी के पास same row हो
```

### Code Example

```bash
# Step 1: ch1 में data insert करो
docker compose exec ch1 clickhouse-client --query \
  "INSERT INTO events VALUES (1, now(), 'hello-from-ch1')"

# Step 2: Wait करो (1-2 seconds)
sleep 2

# Step 3: ch2 से check करो - row दिखेगी
docker compose exec ch2 clickhouse-client --query \
  "SELECT * FROM events"

# Output:
# 1  2024-01-15 10:30:45  hello-from-ch1

# Step 4: ch3 से भी check करो - same row होगी
docker compose exec ch3 clickhouse-client --query \
  "SELECT * FROM events"

# Output:
# 1  2024-01-15 10:30:45  hello-from-ch1
```

### क्या होता है internally?

```
Timeline:
  T=0s   : INSERT on ch1
  T=0.1s : Keeper को entry add हुई
  T=0.5s : ch2 को Keeper से notification मिली
  T=0.6s : ch2 ने data apply किया
  T=0.5s : ch3 को Keeper से notification मिली
  T=0.6s : ch3 ने data apply किया
  
  Result: सब के पास same state!
```

### System Tables से verify करो

```bash
# Replication queue देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.replication_queue WHERE table='events'"

# Keeper में क्या है देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.zookeeper WHERE path = '/clickhouse/tables/01/events'"
```

---

## <a name="test2"></a>5. Test 2: Merge Coordination

### समस्या (The Problem)

```
अगर Keeper नहीं होता:

Insert करो 10 बार:
  ch1: 10 small parts बन जाते हैं
  ch2: 10 small parts बन जाते हैं  
  ch3: 10 small parts बन जाते हैं

Merge करनी हो:
  ch1: सब 10 parts को merge करता है (expensive!)
  ch2: सब 10 parts को merge करता है (expensive!)
  ch3: सब 10 parts को merge करता है (expensive!)
  
  Total: 3 expensive merges! ❌

वाह! यह redundant है और slow है।
```

### Solution: Keeper से Coordination

```
Keeper के साथ:

Insert करो 10 बार:
  सब replicas में 10 small parts बन जाते हैं
  
OPTIMIZE करो:
  Keeper: "Leader election करो"
  → ch1 elected (leader)
  → ch1: merge करेगा
  → ch2, ch3: wait करेंगे
  
Merge complete:
  ch1: merge कर लिया (1 merge, expensive)
  ch2: merged part download करता है (fast!)
  ch3: merged part download करता है (fast!)
  
  Total: 1 merge + 2 downloads! ✓

बचत:
  - समय: 3 merges की जगह 1 merge + 2 downloads
  - CPU: 66% कम usage
  - Disk: 66% कम I/O
```

### Code Example

```bash
# Step 1: 10 छोटे parts बनाओ
for i in $(seq 1 10); do
  docker compose exec ch1 clickhouse-client --query \
    "INSERT INTO events VALUES ($i, now(), 'part-$i')"
done

# Step 2: Parts को देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT name, active FROM system.parts WHERE table='events' ORDER BY name"

# Output: 10 different part names

# Step 3: OPTIMIZE करो (merge)
docker compose exec ch1 clickhouse-client --query \
  "OPTIMIZE TABLE events FINAL"

# Step 4: Replication queue देखो - merge operations दिखेंगे
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.replication_queue WHERE table='events'"

# Step 5: सब replicas के parts check करो
docker compose exec ch1 clickhouse-client --query \
  "SELECT name FROM system.parts WHERE table='events' AND active ORDER BY name"

docker compose exec ch2 clickhouse-client --query \
  "SELECT name FROM system.parts WHERE table='events' AND active ORDER BY name"

docker compose exec ch3 clickhouse-client --query \
  "SELECT name FROM system.parts WHERE table='events' AND active ORDER BY name"

# Output: सभी replicas के पास **same** merged parts होंगे!
```

### क्या Keeper में happens?

```
/clickhouse/tables/01/events/log:
  [entry1] INSERT 1
  [entry2] INSERT 2
  ...
  [entry10] INSERT 10
  [entry11] MERGE START [part_names: ...]
  [entry12] MERGE COMPLETE [result_part: ...]

हर replica:
  - Log पढ़ता है
  - Same order में operations apply करता है
  - Result: सभी के पास same data
```

---

## <a name="test3"></a>6. Test 3: Leader Election

### समस्या बिना Leader Election के (Problem without it)

```
अगर एक replica fail हो जाए:

❌ बिना coordination के:
  ch1 merge करता है → part: 20240115_1_1
  ch2 merge करता है → part: 20240115_2_2  ← Different!
  ch3 fail हो जाता है
  
  अब सभी के पास different data है! ❌
  Conflict, inconsistency, disaster!
```

### Solution: Leader Election

```
✓ Keeper से:
  Keeper elections करता है: "ch1 is leader for this merge"
  
  ch1 (Leader):
    - Merge करता है
    - Result: part_merged_1
  
  ch2, ch3:
    - Merge नहीं करते
    - ch1 से part_merged_1 download करते हैं
  
  Result: सभी के पास **same** part! ✓
```

### Code Example

```bash
# Step 1: Leader election state देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.zookeeper WHERE path = '/clickhouse/tables/01/events/leader_election'"

# Output: कुछ replica का name दिखेगा (current leader)

# Step 2: Leader को kill करो
docker compose stop ch1

# Step 3: दूसरे replicas से insert करो
docker compose exec ch2 clickhouse-client --query \
  "INSERT INTO events VALUES (999, now(), 'after-leader-stop')"

# Step 4: ch3 से verify करो - row दिखेगी
docker compose exec ch3 clickhouse-client --query \
  "SELECT * FROM events WHERE id=999"

# Output: row दिखेगी, even though ch1 down है!

# Step 5: Leader को revive करो
docker compose start ch1

# Step 6: Check - ch1 भी data को catch up कर लेगा
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM events WHERE id=999"

# Output: row दिखेगी!
```

### क्या होता है internally?

```
Timeline:

T=0s   : ch1 को kill किया
T=0.5s : Keeper detect करता है: "ch1 down है"
T=1s   : New leader election होती है → ch2 elected
T=2s   : ch2 को insert command आता है
T=2.1s : ch2 log में entry add करता है
T=2.2s : ch3 को notification
T=2.3s : ch3 apply करता है

T=5s   : ch1 को revive किया
T=5.5s : ch1 connect होता है Keeper से
T=6s   : ch1 sees: "मैं missed entries पर हूँ"
T=6.5s : ch1 replays missed entries
T=7s   : ch1 catch up हो जाता है!
```

---

## <a name="test4"></a>7. Test 4: Replica Catch-Up

### समस्या (Problem Scenario)

```
अगर एक replica बहुत दिन के लिए down हो:

Before:
  ch1: [Row1, Row2, Row3]
  ch2: [Row1, Row2, Row3]
  ch3: DOWN ❌

बीच में operations:
  ch1 को: INSERT Row4, Row5
  ch2 को: operations मिल गए (updated)
  ch3: missed करता है ❌

अब ch3 restart होता है:
  पर ch3 को Row4, Row5 कहाँ से आएंगे?
```

### Solution: Keeper का Log

```
✓ Keeper पूरा log store करता है:

/clickhouse/tables/01/events/log:
  [1] INSERT 1
  [2] INSERT 2
  [3] INSERT 3
  [4] INSERT 4    ← ch3 को पता नहीं था
  [5] INSERT 5    ← ch3 को पता नहीं था

जब ch3 restart होता है:

Step 1: Connect to Keeper
  ch3: "नमस्ते! मैं alive हूँ"

Step 2: Check position
  Keeper: "तुम entry 3 तक update हो, बाकी करो"

Step 3: Replay entries
  ch3 replays entry 4, 5
  Apply करता है

Step 4: Catch-up complete
  ch3: [Row1, Row2, Row3, Row4, Row5] ✓
```

### Code Example

```bash
# Step 1: ch3 को stop करो
docker compose stop ch3

# Step 2: ch1 में नया data insert करो
docker compose exec ch1 clickhouse-client --query \
  "INSERT INTO events VALUES (2000, now(), 'while-ch3-down')"

# Step 3: ch3 को restart करो
docker compose start ch3

# Step 4: Replication queue देखो - ch3 को replay करते दिख जाएगा
docker compose exec ch3 clickhouse-client --query \
  "SELECT * FROM system.replication_queue WHERE table='events'"

# Step 5: Wait करो 2-3 seconds

# Step 6: Verify - ch3 catch up हो गया
docker compose exec ch3 clickhouse-client --query \
  "SELECT count() FROM events"

# Output: वो ही count जो ch1 और ch2 में है!

# Step 7: Specific row भी आ गई
docker compose exec ch3 clickhouse-client --query \
  "SELECT * FROM events WHERE id=2000"

# Output: row दिखेगी
```

### महत्वपूर्ण Note (Important!)

```
✓ Keeper replicas को कभी data loss नहीं होने देता:

1. हर operation Keeper के log में जाता है
2. Entry को सहेज लिया जाता है
3. सब replicas को भेजा जाता है
4. कोई भी replica miss नहीं कर सकता (अगर persistent है)

यह **durability** का guarantee है!
```

---

## <a name="section4"></a>8. Keeper में Key Data Structures

### Path Structure (पाथ संरचना)

```
सभी metadata यहाँ store होता है:
/clickhouse/tables/{shard}/{table}

हमारे lab में:
/clickhouse/tables/01/events
```

### Important Znodes (महत्वपूर्ण Znodes)

**1. /replicas** - सभी replicas की list

```
/clickhouse/tables/01/events/replicas/
  ├── ch1
  │   └── is_active: 1  (alive है)
  ├── ch2
  │   └── is_active: 1  (alive है)
  └── ch3
      └── is_active: 0  (offline है)

Keeper track करता है कौन online है!
```

**2. /log** - Shared replication log

```
/clickhouse/tables/01/events/log/
  ├── log-0000000000
  │   └── content: INSERT 1, now(), 'hello'
  ├── log-0000000001
  │   └── content: INSERT 2, now(), 'world'
  ├── log-0000000002
  │   └── content: MERGE START ...
  └── log-0000000003
      └── content: MERGE COMPLETE ...

हर operation log entry में जाती है!
हर replica इसे follow करता है!
```

**3. /blocks** - Merge blocks (concurrency control)

```
/clickhouse/tables/01/events/blocks/
  ├── block_0000000000
  │   └── range: [0, 10000]
  │       replica: ch1
  │       reason: MERGE in progress

यह ensure करता है:
- सिर्फ एक replica किसी range को merge करे
- दूसरे उसी range पर काम न करें
- No conflicts! ✓
```

**4. /leader_election** - Leader tracking

```
/clickhouse/tables/01/events/leader_election/
  └── node: ch1

Current leader: ch1
अगर ch1 fail हो:
  - Keeper automatic election करेगा
  - नया leader चुना जाएगा
  - Merges continue होंगे
```

**5. /metadata** - Table schema

```
/clickhouse/tables/01/events/metadata
  └── content:
      columns: id, event_time, payload
      types: UInt64, DateTime, String
      order_key: id
      compression: LZ4

हर replica को पता रहता है schema क्या है!
```

### How to Inspect (कैसे देखें)

```bash
# Keeper में सब देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.zookeeper WHERE path LIKE '/clickhouse/tables/01/events%' LIMIT 20"

# Specific path देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.zookeeper WHERE path = '/clickhouse/tables/01/events/replicas'"
```

---

## <a name="section5"></a>9. सीखने योग्य अवधारणाएं (Key Concepts)

### Concept 1: Source of Truth (सच का स्रोत)

```
❌ बिना Keeper के:
  हर replica अपना decision लेता है
  → Conflicts
  → Inconsistency
  → Chaos

✓ Keeper के साथ:
  Keeper अंतिम निर्णय लेता है
  → सब अनुसरण करते हैं
  → Consistency
  → Order

💡 याद रखो: 
"Keeper का log = सत्य"
```

### Concept 2: Coordinated Merges (आपसी merge)

```
Traditional approach:
  3 replicas × (expensive merge) = waste

Keeper approach:
  1 leader merge + 2 downloads = efficient

💡 फायदे:
  - CPU usage 66% कम
  - Disk I/O 66% कम
  - Time: same or faster
  - All consistent ✓
```

### Concept 3: Asynchronous Replication

```
आपको data दे दिया गया = क्या सब replicas को मिल गया?

Answer: नहीं! अभी कुछ देर लगेगी

Timeline:
  T=0s   : Client को: "OK, data सहेज लिया"
  T=0.1s : Keeper को भेजा
  T=0.5s : दूसरे replicas को पता चला
  T=1s   : सब replicas पर data पहुँच गया

यह async है, immediate नहीं।
पर eventual consistency है (आखिर में सब equal हो जाएंगे)
```

### Concept 4: Fault Tolerance (विफलता सहन करना)

```
बहुत ताकत है Keeper में:

अगर 1 replica fail हो:
  ✓ System continue करता है
  ✓ दूसरे 2 से काम होता रहता है
  ✓ Data loss नहीं होता
  ✓ Restart पर automatic recovery

अगर 2 replicas fail हो:
  ⚠ Risky! सिर्फ 1 बचा है
  ⚠ Replication limit के करीब

अगर Keeper itself fail हो:
  ❌ System काम नहीं करेगा
  → इसलिए production में 3+ Keeper nodes चलाएं!
```

### Concept 5: No Split-Brain

```
Split-brain = दोनों side अपने को "correct" समझें

❌ बिना Keeper:
  ch1: मुझे लगता है मैं leader हूँ
  ch2: मुझे लगता है मैं leader हूँ
  
  दोनों merge करते हैं!
  → Different results
  → Disaster!

✓ Keeper के साथ:
  Keeper: "ch1 is leader"
  ch2: OK, सुन लिया
  ch1: Merge करूँ
  ch2: नहीं, मैं नहीं करूँगा
  
  Result: एक ही merge, no conflicts ✓
```

---

## <a name="section6"></a>10. Lab को कैसे चलाएं? (How to Run)

### Prerequisites (जरूरी चीजें)

```bash
# Docker installed है?
docker --version

# Docker Compose है?
docker compose version

# अगर नहीं है:
# Ubuntu/Debian के लिए:
apt-get update && apt-get install docker-ce docker-compose-plugin

# या Docker Desktop use करो (Windows/Mac)
```

### Files तैयार करो

```bash
# Lab script को save करो
# Filename: run_lab.sh

# या मेरी script download करो:
# clickhouse_keeper_lab.txt

# Copy करो:
cp clickhouse_keeper_lab.txt run_lab.sh
chmod +x run_lab.sh
```

### Lab चलाओ

```bash
# Option 1: पूरी script चलाओ
./run_lab.sh

# Option 2: Step-by-step चलाओ
cd /root/ch-lab-docker

# Clean करो
docker compose down -v
docker system prune -f

# Start करो
docker compose up -d

# Check करो
docker compose ps
```

### Expected Output

```
Container                    Status
---------------------------------------------
keeper1                      running ✓
ch1                          running ✓
ch2                          running ✓
ch3                          running ✓
```

### Ports को Check करो

```bash
# सब ports खुले हैं?
lsof -i :9000 -i :9001 -i :9002 -i :9181 -i :8123 -i :8124 -i :8125

# या:
netstat -tlnp | grep -E '9000|9001|9002|9181|8123|8124|8125'
```

### Manually Tests चलाओ

```bash
# Test 1: Basic Replication
docker compose exec ch1 clickhouse-client --query \
  "INSERT INTO events VALUES (1, now(), 'test1')"
sleep 2
docker compose exec ch2 clickhouse-client --query "SELECT * FROM events"

# Test 2: Merge
for i in $(seq 1 10); do
  docker compose exec ch1 clickhouse-client --query \
    "INSERT INTO events VALUES ($i, now(), 'part-$i')"
done
docker compose exec ch1 clickhouse-client --query "OPTIMIZE TABLE events FINAL"

# Test 3: Leader Election
docker compose stop ch1
docker compose exec ch2 clickhouse-client --query \
  "INSERT INTO events VALUES (999, now(), 'after-stop')"
docker compose exec ch3 clickhouse-client --query "SELECT * FROM events WHERE id=999"
docker compose start ch1

# Test 4: Catch-up
docker compose stop ch3
docker compose exec ch1 clickhouse-client --query \
  "INSERT INTO events VALUES (2000, now(), 'while-down')"
docker compose start ch3
sleep 3
docker compose exec ch3 clickhouse-client --query "SELECT count() FROM events"
```

### Cleanup करो

```bash
# सब containers बंद करो
docker compose down

# Volumes को भी delete करो
docker compose down -v

# System cleanup करो
docker system prune -f
```

---

## Summary: एक-एक लाइन में (One-Line Summary)

| Concept | समझाइश |
|---------|---------|
| **Keeper** | Database का **dictator** है - सब commands यहाँ से आते हैं |
| **Replication** | Data को copies करना ताकि **redundancy** हो |
| **Log** | Keeper का memory - सब operations यहाँ लिखे होते हैं |
| **Leader** | **एक** replica को merge करने का काम, बाकी copy करते हैं |
| **Coordination** | सब replicas एक-दूसरे से **बात** करके काम करते हैं |
| **Consistency** | सब के पास **same** data होता है (eventually) |
| **Durability** | Data कभी **खोता** नहीं, Keeper सहेज लेता है |
| **Catch-up** | Offline हुए replica को restart पर सब operations **replay** किए जाते हैं |

---

## अंतिम सीख (Final Takeaway)

```
ClickHouse Keeper बिना:
  ❌ Inconsistent data
  ❌ Redundant work
  ❌ No failover
  ❌ Data loss risk

ClickHouse Keeper के साथ:
  ✓ Consistent replicas
  ✓ Coordinated operations  
  ✓ Automatic failover
  ✓ Durable data
  ✓ High availability
```

**अब तुम समझ गए कि Keeper क्यों जरूरी है!** 🎉

---

## Further Learning (और सीखने के लिए)

### Useful Commands (उपयोगी commands)

```bash
# Replicas को देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.replicas WHERE table='events'"

# Replication status को देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT hostname, queue_size, inserts_in_queue FROM system.replication_queue WHERE table='events'"

# Parts को विस्तार से देखो
docker compose exec ch1 clickhouse-client --query \
  "SELECT name, rows, bytes, active FROM system.parts WHERE table='events' ORDER BY name"

# Keeper से connection status
docker compose exec ch1 clickhouse-client --query \
  "SELECT * FROM system.zookeeper WHERE path = '/clickhouse/tables/01/events/replicas'"

# Database को देखो
docker compose exec ch1 clickhouse-client --query "SHOW DATABASES"

# Tables को देखो
docker compose exec ch1 clickhouse-client --query "SHOW TABLES"
```

### Interesting Experiments (दिलचस्प परीक्षण)

```bash
# 1. Multiple inserts simultaneously
for i in {1..100}; do
  docker compose exec ch1 clickhouse-client --query \
    "INSERT INTO events VALUES ($i, now(), 'batch-$i')" &
done
wait

# 2. Replication lag को measure करो
# ch1 पर insert करो
docker compose exec ch1 clickhouse-client --query \
  "INSERT INTO events VALUES (5000, now(), 'test')"

# तुरंत ch2 से check करो
docker compose exec ch2 clickhouse-client --query "SELECT * FROM events WHERE id=5000"
# ये fail हो सकता है (replication lag)

# 2 seconds बाद फिर से try करो
sleep 2
docker compose exec ch2 clickhouse-client --query "SELECT * FROM events WHERE id=5000"
# अब दिखेगा!

# 3. Replication queue को देखो जब operations चल रहे हों
# एक terminal में:
docker compose exec ch1 clickhouse-client --query \
  "WATCH SELECT * FROM system.replication_queue WHERE table='events' EVENTS LIMIT 10"

# दूसरे में:
for i in {1..50}; do
  docker compose exec ch1 clickhouse-client --query "INSERT INTO events VALUES ($((i+10000)), now(), 'watch-$i')"
done
```

---

**Happy Learning! 🎓**

**अगर कुछ समझ न आए, तो script run करो और output को observe करो।**

**The lab will teach you!** ✨
