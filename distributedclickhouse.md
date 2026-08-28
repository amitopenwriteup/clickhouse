# Lab: Sharding, Replication & Failover in ClickHouse
### Cluster Topology Design • ZooKeeper/Keeper Coordination • Failover Scenarios

**Environment:** ClickHouse Cloud (trial account) + a local Docker Compose cluster for the parts a managed trial account can't expose

---

## 0. Read this first — what a Cloud trial account can and can't show you

ClickHouse Cloud deliberately **removes** the operational surface this topic is normally taught on. Under the hood, Cloud uses **SharedMergeTree** instead of classic `ReplicatedMergeTree`: all durable data lives in shared object storage (S3/GCS/Azure Blob), compute nodes are stateless and interchangeable, and Keeper is a fully managed internal service you never configure, SSH into, or restart yourself. There is no `config.xml` to edit, no shard/replica macros to assign, and no way to `docker kill` a replica.

This means:

| What you asked for | Can you do it in a Cloud trial account? |
|---|---|
| Query cluster topology (`system.clusters`, `system.replicas`) | ✅ Yes — read-only, fully visible |
| Understand replication behavior and consistency | ✅ Yes — observable via system tables |
| Design a sharding key / cluster topology on paper | ✅ Yes — a design exercise |
| Manually configure shards, replicas, or Keeper ensembles | ❌ No — fully managed, no config access |
| Kill a Keeper node or a ClickHouse replica to test failover | ❌ No — no node-level access in a trial account |
| Observe *simulated* failover behavior hands-on | ✅ Yes — via a local Docker Compose cluster (Part 5 of this lab) |

So this lab is split in two: **Parts 1–3 run entirely in your ClickHouse Cloud trial account** (topology inspection, replication concepts, cluster design). **Parts 4–5 use a small local Docker Compose cluster** — the only way to actually pull the plug on a node and watch Keeper/replication react, which is the whole point of a failover exercise. This mirrors real-world practice: you'd rehearse failover drills on a self-managed staging cluster (or ClickHouse's own reference architecture), not in a black-box managed service.

---

## Part 1 — Cluster Topology Inspection (ClickHouse Cloud)

### 1.1 Setup

```sql
CREATE DATABASE IF NOT EXISTS ha_lab;
USE ha_lab;
```

### 1.2 Inspect your service's logical cluster

```sql
SELECT cluster, shard_num, shard_weight, replica_num, host_name, host_address, is_local
FROM system.clusters;
```

In a Cloud trial service, expect to see a single logical cluster with **one shard** and **multiple replicas** (compute nodes) — Cloud scales by adding replicas/compute, not by manually adding shards, because storage is shared rather than partitioned.

### 1.3 Inspect replica state for a table

```sql
CREATE TABLE ha_events
(
    event_time DateTime,
    user_id    UInt64,
    event_type String
)
ENGINE = MergeTree()   -- Cloud provisions this as SharedMergeTree under the hood
ORDER BY (user_id, event_time);

INSERT INTO ha_events VALUES (now(), 1, 'click'), (now(), 2, 'view');
```

```sql
SELECT database, table, is_leader, total_replicas, active_replicas, absolute_delay, queue_size
FROM system.replicas
WHERE table = 'ha_events';
```

> On Cloud, `system.replicas` may show fields as not applicable / trivially healthy compared to a self-managed `ReplicatedMergeTree`, since there's no physical part-copying between nodes — all compute nodes read the same shared storage. This is expected and itself an important lesson: **replication semantics change fundamentally when storage is shared.**

**Exercise 1:** Compare the shard/replica counts you see in `system.clusters` to what your Cloud trial service's compute settings show in the console (number of replicas provisioned). Write one sentence connecting the two.

---

## Part 2 — Replication Concepts (Cloud-observable, on shared storage)

### 2.1 Multi-master writes

Cloud allows writes to land on any compute replica. Insert from one session and read from another (if your trial exposes multiple endpoints/nodes) to confirm strong read-after-write consistency, since all replicas read the same underlying shared storage rather than waiting for asynchronous part propagation.

```sql
INSERT INTO ha_events VALUES (now(), 3, 'purchase');
SELECT count() FROM ha_events;  -- run again from a different session/tab
```

### 2.2 Distributed DDL — `ON CLUSTER`

```sql
SHOW CLUSTERS;
```

```sql
CREATE TABLE ha_events_v2 ON CLUSTER default
(
    event_time DateTime,
    user_id    UInt64,
    event_type String
)
ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

> On a self-managed cluster, `ON CLUSTER` DDL is queued in Keeper (`/clickhouse/task_queue/ddl`) and each node executes it independently, reporting completion back through Keeper. If Keeper quorum is lost, DDL freezes. On Cloud, this queueing still conceptually happens, but the managed Keeper service means you'll rarely (if ever) observe it failing — that's exactly the tradeoff of a managed service: less to operate, less to observe.

**Exercise 2:** Run `EXISTS TABLE ha_events_v2;` and describe, based on what you know of `ON CLUSTER` semantics, what would happen to this statement mid-flight if a self-managed cluster's Keeper ensemble lost quorum at that exact moment.

---

## Part 3 — Cluster Topology Design (paper exercise, informed by Cloud limits)

Even though your trial account won't let you build a multi-shard cluster by hand, topology design is still something you're expected to reason about — e.g., for self-managed deployments, or for choosing between Cloud's single-shard model and a self-managed sharded one at large scale.

### 3.1 Design scenario

You're designing ingestion for a 50 TB/day clickstream dataset with these query patterns:
- 80% of queries filter by `tenant_id`
- 15% are cross-tenant aggregate reports
- Data must survive a full single-node/AZ failure with < 5 minutes of unavailability

**Exercise 3 (design, no SQL required):** Sketch a topology answering:
1. How many shards, and what sharding key? (Consider the hot-shard risk of sharding directly by `tenant_id` vs. hashing it.)
2. How many replicas per shard, and across how many availability zones?
3. How many Keeper nodes, and why must it be an odd number ≥ 3?
4. Would you choose ClickHouse Cloud (shared storage, no manual sharding) or a self-managed sharded cluster for this workload, and why?

*(Reference answer sketch: `intHash64(tenant_id)` or `cityHash64(tenant_id, user_id)` as sharding key to avoid hot shards while colocating tenant data; at least 2–3 replicas per shard spread across 3 AZs; Keeper as a 3- or 5-node quorum — always odd, since Raft needs a majority and an even number adds a node without adding fault tolerance. At 50 TB/day, either can work: Cloud trades some sharding control for zero operational burden; self-managed gives full control over sharding key and hardware at the cost of running Keeper and replication yourself.)*

---

## Part 4 — Local Cluster: Sharding, Replication & Keeper Hands-On

Since a Cloud trial account has no node-level access, this part uses a local Docker Compose cluster (ClickHouse's own reference architecture) to actually configure and observe replication + Keeper. This doesn't require paying for anything — just Docker on your machine.

### 4.1 Get the reference cluster

```bash
git clone https://github.com/ClickHouse/examples.git
cd examples/docker-compose-recipes/recipes/cluster_1S_2R
docker compose up -d
```

This spins up: 1 shard, 2 replicas of ClickHouse, and a 3-node Keeper ensemble.

### 4.2 Verify the cluster

```bash
docker compose exec clickhouse-01 clickhouse-client --query "SELECT * FROM system.clusters FORMAT Pretty"
```

### 4.3 Create a replicated table across the cluster

```sql
CREATE TABLE events_local ON CLUSTER cluster_1S_2R
(
    event_time DateTime,
    user_id    UInt64,
    event_type String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events_local', '{replica}')
ORDER BY (user_id, event_time);
```

### 4.4 Confirm replication of an insert

```bash
docker compose exec clickhouse-01 clickhouse-client --query \
  "INSERT INTO events_local VALUES (now(), 1, 'click')"

# Read from the OTHER replica — should see the row after a short propagation delay
docker compose exec clickhouse-02 clickhouse-client --query \
  "SELECT * FROM events_local"
```

```sql
SELECT database, table, is_leader, total_replicas, active_replicas, absolute_delay
FROM system.replicas
WHERE table = 'events_local';
```

**Exercise 4:** Insert 5 rows on `clickhouse-01`, immediately query `clickhouse-02`, and note whether all 5 rows are visible yet. Explain the `absolute_delay` field in terms of asynchronous replication.

---

## Part 5 — Failover Scenarios

### 5.1 Scenario: kill a replica, verify continued availability

```bash
docker compose stop clickhouse-02
```

```bash
# The surviving replica should still serve reads and accept writes
docker compose exec clickhouse-01 clickhouse-client --query \
  "INSERT INTO events_local VALUES (now(), 2, 'view')"
docker compose exec clickhouse-01 clickhouse-client --query \
  "SELECT count() FROM events_local"
```

Bring it back and confirm it catches up:

```bash
docker compose start clickhouse-02
sleep 10
docker compose exec clickhouse-02 clickhouse-client --query \
  "SELECT count() FROM events_local"
```

```sql
SELECT database, table, replica_name, is_session_expired, absolute_delay
FROM system.replicas
WHERE table = 'events_local';
```

### 5.2 Scenario: lose Keeper quorum

```bash
docker compose ps | grep keeper
docker compose stop keeper-01 keeper-02
```

With 2 of 3 Keeper nodes down, the ensemble loses Raft majority.

```bash
# Existing replicated reads of already-committed data usually still work...
docker compose exec clickhouse-01 clickhouse-client --query "SELECT count() FROM events_local"

# ...but new DDL, or anything that must write to Keeper's replication log, will stall/fail
docker compose exec clickhouse-01 clickhouse-client --query \
  "INSERT INTO events_local VALUES (now(), 3, 'error_case')"
```

Restore quorum and confirm recovery:

```bash
docker compose start keeper-01 keeper-02
sleep 15
docker compose exec clickhouse-01 clickhouse-client --query \
  "INSERT INTO events_local VALUES (now(), 4, 'recovered')"
```

**Exercise 5:** Document what happened to the `INSERT` while quorum was lost — did it hang, error immediately, or queue and retry? Compare that behavior to what you'd expect from losing a *single* Keeper node (1 of 3 down, quorum intact) instead.

### 5.3 Scenario: split-brain avoidance discussion

**Exercise 6 (discussion, no commands):** Explain, in 3–4 sentences, why an even number of Keeper nodes (e.g., 4) is *worse* than 3 for fault tolerance, in terms of Raft majority math, and why ClickHouse recommends dedicating separate hosts to Keeper rather than co-locating it with ClickHouse server processes in production.

---

## 6. Wrap-Up Comparison

| Concept | ClickHouse Cloud (trial) | Self-managed (Docker lab) |
|---|---|---|
| Sharding | Abstracted — shared object storage, no manual shard config | You define shards, sharding key, and `Distributed` tables yourself |
| Replication | Implicit via SharedMergeTree + shared storage | Explicit via `ReplicatedMergeTree` + Keeper-coordinated part fetches |
| Keeper/ZooKeeper | Fully managed, invisible | You deploy, size, and monitor a Keeper/ZooKeeper ensemble yourself |
| Failover testing | Not directly possible (no node access) | Fully testable — stop/start containers, observe `system.replicas` |
| Cluster topology design | Still a valid design exercise (Part 3) | Directly implemented and observable |

## 7. Cleanup

**Cloud:**
```sql
DROP DATABASE IF EXISTS ha_lab;
```

**Local:**
```bash
docker compose down -v
```

---

### Further Reading
- ClickHouse docs: "Replicating data" (Docker Compose reference cluster), architecture overview
- ClickHouse docs: ClickHouse Keeper reference, `system.replicas`, `system.clusters`, `system.replication_queue`
- ClickHouse blog: SharedMergeTree and Cloud architecture
