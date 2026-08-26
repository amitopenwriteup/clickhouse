# ClickHouse Installation & Configuration Guide

## Table of Contents
1. [Single-Node Installation](#1-single-node-installation)
2. [Cluster Installation](#2-cluster-installation)
3. [Docker-Based Setup](#3-docker-based-setup)
4. [Cloud vs Self-Managed](#4-cloud-vs-self-managed)
5. [Core Configuration Files](#5-core-configuration-files)

---

## 1. Single-Node Installation

### 1.1 Debian/Ubuntu (APT)

```bash
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/clickhouse-keyring.gpg \
  --keyserver keyserver.ubuntu.com --recv 8919F6BD2B48D754

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/clickhouse.list

sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client

sudo service clickhouse-server start
clickhouse-client
```

### 1.2 Rocky Linux (DNF/YUM)

Rocky Linux 8 and 9 both use the official RPM repo. `dnf` is the preferred tool (yum is aliased to it on Rocky), but either works.

```bash
# Install prerequisites
sudo dnf install -y yum-utils

# Add the ClickHouse RPM repo
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo

# Install server + client
sudo dnf install -y clickhouse-server clickhouse-client

# Enable and start the service
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
```

**SELinux (Rocky ships it enforcing by default):**

```bash
# Check status
getenforce

# If SELinux blocks the server (denied file access, etc.), check logs:
sudo ausearch -m avc -ts recent

# Preferred fix: generate and load a policy module rather than disabling SELinux
sudo grep clickhouse /var/log/audit/audit.log | audit2allow -M clickhouse_policy
sudo semodule -i clickhouse_policy.pp

# Or, if you must relabel context (less ideal):
sudo semanage fcontext -a -t var_lib_t "/var/lib/clickhouse(/.*)?"
sudo restorecon -Rv /var/lib/clickhouse
```

**firewalld (Rocky ships it enabled by default):**

```bash
sudo firewall-cmd --permanent --add-port=8123/tcp   # HTTP interface
sudo firewall-cmd --permanent --add-port=9000/tcp   # Native TCP
sudo firewall-cmd --permanent --add-port=9009/tcp   # Interserver
sudo firewall-cmd --permanent --add-port=9181/tcp   # Keeper (if used)
sudo firewall-cmd --reload
```

**Legacy note:** on older Rocky 8 minimal installs, `yum-config-manager` may require `dnf-plugins-core` instead of `yum-utils`:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```

### 1.3 Generic (tarball / precompiled binary)

```bash
curl https://clickhouse.com/ | sh
./clickhouse install
sudo clickhouse start
```

### 1.4 Verify Installation

```bash
clickhouse-client --query "SELECT version()"
```

### 1.5 Key Paths (default)

| Purpose            | Path                                |
|--------------------|--------------------------------------|
| Binary             | `/usr/bin/clickhouse`                |
| Config directory   | `/etc/clickhouse-server/`            |
| Data directory     | `/var/lib/clickhouse/`               |
| Logs               | `/var/log/clickhouse-server/`        |
| PID file           | `/var/run/clickhouse-server/`        |

---

## 2. Cluster Installation

A ClickHouse cluster is defined logically via configuration (there's no built-in cluster "installer" — each node runs a standalone server, and cluster topology is declared in XML config, typically alongside **ZooKeeper** or **ClickHouse Keeper** for coordination).

### 2.1 Prerequisites

- 3+ nodes for HA (odd number recommended for Keeper quorum)
- Synchronized clocks (NTP)
- Open ports: `9000` (native), `8123` (HTTP), `9009` (interserver), `9181` (Keeper)

### 2.2 Step 1 — Set up Coordination (ClickHouse Keeper)

On each Keeper node, configure `/etc/clickhouse-server/config.d/keeper.xml`:

```xml
<clickhouse>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>1</server_id> <!-- unique per node: 1, 2, 3 -->
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>warning</raft_logs_level>
        </coordination_settings>

        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>keeper1.local</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>keeper2.local</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>keeper3.local</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

### 2.3 Step 2 — Define Cluster Topology (Sharding/Replication)

In `/etc/clickhouse-server/config.d/remote_servers.xml` on every data node:

```xml
<clickhouse>
    <remote_servers>
        <my_cluster>
            <shard>
                <replica>
                    <host>node1.local</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>node2.local</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>node3.local</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>node4.local</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>

    <zookeeper>
        <node><host>keeper1.local</host><port>9181</port></node>
        <node><host>keeper2.local</host><port>9181</port></node>
        <node><host>keeper3.local</host><port>9181</port></node>
    </zookeeper>

    <macros>
        <shard>01</shard>
        <replica>node1</replica>
    </macros>
</clickhouse>
```

> `<macros>` values are unique per node and referenced in `ReplicatedMergeTree` table definitions.

### 2.4 Step 3 — Create Replicated Tables

```sql
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_date Date,
    event_id UInt64,
    payload String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events_local', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_id);

CREATE TABLE events ON CLUSTER my_cluster AS events_local
ENGINE = Distributed(my_cluster, default, events_local, rand());
```

### 2.5 Validate Cluster

```sql
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';
SELECT * FROM system.zookeeper WHERE path = '/';
```

---

## 3. Docker-Based Setup

### 3.1 Quick Start (Single Container)

```bash
docker run -d \
  --name clickhouse-server \
  -p 8123:8123 -p 9000:9000 \
  --ulimit nofile=262144:262144 \
  -v ch_data:/var/lib/clickhouse \
  -v ch_logs:/var/log/clickhouse-server \
  clickhouse/clickhouse-server:latest

docker exec -it clickhouse-server clickhouse-client
```

### 3.2 Mounting Custom Configs

```bash
docker run -d \
  --name clickhouse-server \
  -p 8123:8123 -p 9000:9000 \
  -v $(pwd)/config.xml:/etc/clickhouse-server/config.xml \
  -v $(pwd)/users.xml:/etc/clickhouse-server/users.xml \
  -v ch_data:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

### 3.3 Docker Compose (Multi-Node Cluster + Keeper)

```yaml
version: "3.8"

services:
  keeper1:
    image: clickhouse/clickhouse-keeper:latest
    hostname: keeper1
    volumes:
      - ./keeper1/config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ports:
      - "9181:9181"

  ch-node1:
    image: clickhouse/clickhouse-server:latest
    hostname: node1
    depends_on:
      - keeper1
    volumes:
      - ./node1/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ./node1/users.xml:/etc/clickhouse-server/users.d/users.xml
      - ch1_data:/var/lib/clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"

  ch-node2:
    image: clickhouse/clickhouse-server:latest
    hostname: node2
    depends_on:
      - keeper1
    volumes:
      - ./node2/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ./node2/users.xml:/etc/clickhouse-server/users.d/users.xml
      - ch2_data:/var/lib/clickhouse
    ports:
      - "8124:8123"
      - "9001:9000"

volumes:
  ch1_data:
  ch2_data:
```

```bash
docker compose up -d
docker compose logs -f ch-node1
```

### 3.4 Notes

- Use the official `clickhouse/clickhouse-server` and `clickhouse/clickhouse-keeper` images (Altinity images also exist for older/patched builds).
- Always set `ulimit nofile` high enough (ClickHouse opens many file descriptors under load).
- For production, avoid bind-mounting the entire `/etc/clickhouse-server` — instead mount only files into `config.d/` and `users.d/` so defaults aren't clobbered.

---

## 4. Cloud vs Self-Managed

| Dimension | ClickHouse Cloud | Self-Managed |
|---|---|---|
| **Provisioning** | Fully managed control plane; deploy via web console/API/Terraform | You install, patch, and manage OS + ClickHouse binaries |
| **Scaling** | Automatic vertical/horizontal scaling, scale-to-zero for idle services | Manual — add nodes, reshard, rebalance yourself |
| **Storage** | Separated storage/compute on object storage (S3/GCS/Azure Blob) by default | You choose: local disk, or configure `S3`/`MergeTree` storage policies manually |
| **HA / Replication** | Built-in, managed automatically | You configure ReplicatedMergeTree + Keeper/ZooKeeper yourself |
| **Backups** | Automated, managed | You configure `BACKUP`/`RESTORE` or external tooling (e.g., clickhouse-backup) |
| **Upgrades** | Handled by provider, rolling, minimal downtime | You plan and execute version upgrades, test compatibility |
| **Config access** | Limited — many `config.xml`/`users.xml` settings abstracted into cloud settings/roles UI | Full access to `config.xml`, `users.xml`, and all `config.d`/`users.d` overrides |
| **Security** | Built-in network isolation, SSO/SAML, IP allowlists, managed TLS | You configure firewalls, TLS certs, LDAP/Kerberos, user management |
| **Cost model** | Usage-based (compute + storage consumption) | Infrastructure cost (VMs/bare metal) + operational overhead |
| **Best fit** | Teams wanting minimal ops burden, fast time-to-value, variable workloads | Teams needing full control, custom hardware, air-gapped/on-prem, or specific compliance/config needs |

**When self-managed makes sense:**
- Strict data-residency/on-prem/air-gapped requirements
- Deep customization of storage engines, disks, or OS-level tuning
- Existing Kubernetes/infra investment (e.g., via the `clickhouse-operator`)

**When Cloud makes sense:**
- Small ops teams / no dedicated DBA
- Rapid prototyping or unpredictable workloads (autoscaling)
- Preference for consumption-based pricing over CapEx

---

## 5. Core Configuration Files

### 5.1 `config.xml` — Server Configuration

Location: `/etc/clickhouse-server/config.xml` (with overrides in `config.d/*.xml`)

Controls server-level behavior: networking, logging, storage, merge tree settings, cluster topology.

```xml
<clickhouse>
    <!-- Logging -->
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>10</count>
    </logger>

    <!-- Network -->
    <listen_host>0.0.0.0</listen_host>
    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <interserver_http_port>9009</interserver_http_port>

    <!-- Paths -->
    <path>/var/lib/clickhouse/</path>
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
    <user_files_path>/var/lib/clickhouse/user_files/</user_files_path>

    <!-- Resource limits -->
    <max_connections>4096</max_connections>
    <max_concurrent_queries>200</max_concurrent_queries>
    <mark_cache_size>5368709120</mark_cache_size>
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>

    <!-- Include external users/cluster config -->
    <users_config>users.xml</users_config>
    <include_from>/etc/clickhouse-server/config.d/metadata.xml</include_from>
</clickhouse>
```

**Best practice:** don't edit `config.xml` directly — drop overrides as separate files into `/etc/clickhouse-server/config.d/`. ClickHouse merges them at startup, which keeps upgrades clean.

### 5.2 `users.xml` — Users, Roles & Quotas

Location: `/etc/clickhouse-server/users.xml` (with overrides in `users.d/*.xml`)

Controls authentication, per-user resource profiles, and access quotas.

```xml
<clickhouse>
    <profiles>
        <default>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
        </default>
        <readonly_profile>
            <readonly>1</readonly>
            <max_memory_usage>5000000000</max_memory_usage>
        </readonly_profile>
    </profiles>

    <users>
        <default>
            <password_sha256_hex>PUT_SHA256_HASH_HERE</password_sha256_hex>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
            <access_management>1</access_management>
        </default>

        <analyst>
            <password_sha256_hex>PUT_SHA256_HASH_HERE</password_sha256_hex>
            <networks>
                <ip>10.0.0.0/8</ip>
            </networks>
            <profile>readonly_profile</profile>
            <quota>default</quota>
            <databases>
                <analytics_db>
                    <events>
                        <filter>tenant_id = currentUser()</filter>
                    </events>
                </analytics_db>
            </databases>
        </analyst>
    </users>

    <quotas>
        <default>
            <interval>
                <duration>3600</duration>
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>
</clickhouse>
```

**Generating a password hash:**

```bash
echo -n "your_password" | sha256sum
```

> Modern ClickHouse also supports simpler `<password>` (plaintext, dev-only), `<password_double_sha1_hex>` (MySQL compat), and SQL-driven user management (`CREATE USER`, `GRANT`) as an alternative to XML — recommended for dynamic environments since it avoids restarts.

### 5.3 Config Precedence & Merging

1. `config.xml` / `users.xml` — base defaults
2. `config.d/*.xml` / `users.d/*.xml` — override fragments (alphabetical merge order)
3. Environment-specific overrides (e.g., via Docker volume mounts or Kubernetes ConfigMaps)
4. SQL-driven changes (`CREATE USER`, `ALTER SETTINGS PROFILE`) — stored in `access_management` storage, take precedence at runtime if `access_management: 1` is enabled

### 5.4 Validating Configuration

```bash
clickhouse-server --config-file=/etc/clickhouse-server/config.xml --check-config
sudo clickhouse-client --query "SELECT * FROM system.settings WHERE changed"
```

---

## Quick Reference Summary

| Task | Command / File |
|---|---|
| Install (Debian) | `apt-get install clickhouse-server clickhouse-client` |
| Start service | `systemctl start clickhouse-server` |
| Docker quick start | `docker run clickhouse/clickhouse-server` |
| Cluster topology | `config.d/remote_servers.xml` |
| Coordination | `config.d/keeper.xml` + ZooKeeper/Keeper |
| Server settings | `/etc/clickhouse-server/config.xml` |
| Users & quotas | `/etc/clickhouse-server/users.xml` |
| Check version | `clickhouse-client --query "SELECT version()"` |
