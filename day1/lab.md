# ClickHouse on Rocky Linux — What Each Part Actually Does

A plain-English walkthrough of the command reference, explaining *why* each step exists, not just what to type.

---

## Part 1 — Installing ClickHouse directly on the host (native install)

### 1. Single-node install

```
sudo dnf install -y dnf-utils yum-utils curl
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
sudo dnf install -y clickhouse-server clickhouse-client
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
```

**What's actually happening:**
- `dnf-utils` / `yum-utils` — these give you the `yum-config-manager` tool itself. You need this *before* you can add a repo.
- `add-repo ...clickhouse.repo` — Rocky Linux's default package repos don't include ClickHouse, so this tells `dnf` "here's a new place to look for packages."
- `dnf install clickhouse-server clickhouse-client` — installs two separate pieces: the **server** (the actual database process) and the **client** (a CLI tool to talk to it — like `mysql` or `psql`).
- `systemctl enable` — makes ClickHouse start automatically on every reboot.
- `systemctl start` — starts it right now, for this session.
- `clickhouse-client` at the end — opens an interactive SQL shell, connecting to the server running on the same machine (defaults to `localhost:9000`).

**Why it matters:** this is the standard "install a database as a native OS service" pattern — same shape as installing MySQL or Postgres via `dnf`.

---

### 2. Quick test

```
clickhouse-client --query "SELECT version()"
clickhouse-client --query "SHOW DATABASES"
```

**What's happening:** instead of opening the interactive shell, `--query` runs one command non-interactively and exits — useful for scripts or a quick sanity check that the server is actually up and responding.

---

### 3. Firewall (firewalld) — required on Rocky Linux

```
sudo firewall-cmd --permanent --add-port=8123/tcp   # HTTP interface
sudo firewall-cmd --permanent --add-port=9000/tcp   # Native protocol
sudo firewall-cmd --permanent --add-port=9009/tcp   # Inter-server (replication)
sudo firewall-cmd --reload
```

**Why this step exists:** Rocky Linux ships with `firewalld` active by default (unlike some Debian-based distros where the firewall is often off out of the box). If you skip this, the server runs fine *locally* but nothing outside the machine can reach it — connections will just time out.

- **8123** — HTTP interface (used for `curl`, some BI tools, health checks)
- **9000** — native TCP protocol (used by `clickhouse-client`, most drivers)
- **9009** — used for data exchange *between* ClickHouse servers in a cluster

`--permanent` writes the rule so it survives a reboot; `--reload` applies it immediately without needing to restart the whole firewall service.

`sestatus` is just a check — ClickHouse's RPM package already ships an SELinux policy for its default file paths, so in most cases you don't need to change anything; this line is there so you know what to check *if* something behaves oddly.

---

### 4. Cluster nodes — additional Keeper ports

```
sudo firewall-cmd --permanent --add-port=2181/tcp
sudo firewall-cmd --permanent --add-port=2888/tcp
sudo firewall-cmd --permanent --add-port=3888/tcp
sudo firewall-cmd --reload
```

**What this is for:** these three ports belong to **Keeper** (ClickHouse's built-in replacement for ZooKeeper), which coordinates multiple servers in a cluster — deciding things like which replica has the latest data, and managing distributed consensus.

**The instruction underneath is the important part:** a cluster isn't a special install — it's the *same* single-node install repeated on every machine, plus:
1. These extra ports opened on every node
2. A `<remote_servers>` config block (defines which hosts form which shards/replicas)
3. A `<zookeeper>` config block (points every node at the same Keeper quorum)
4. Tables created with `ReplicatedMergeTree` (instead of plain `MergeTree`) so replicas sync, plus a `Distributed` table on top that fans queries out across shards.

This section is really just a pointer to that pattern, not a copy-paste script.

---

## Part 2 — Running ClickHouse in Docker instead

This is a completely separate path from Part 1 — you'd pick **either** the native `dnf` install **or** Docker, not both.

### 5. Installing Docker itself on Rocky Linux

```
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

**Why this section exists at all:** Docker Engine isn't in Rocky's default repos either, so — same pattern as ClickHouse itself — you add Docker's own repo first, then install.

- `docker-ce` / `docker-ce-cli` / `containerd.io` — the actual Docker engine and CLI
- `docker-compose-plugin` — adds the `docker compose` command (needed for step 7)
- `systemctl enable --now docker` — enable *and* start in one command (`--now` is a shortcut so you don't need two separate commands like in Part 1)
- `usermod -aG docker $USER` — adds your user to the `docker` group so you can run `docker` commands without typing `sudo` every time. **You have to log out and back in** for a group change to actually apply to your shell session — that's why the comment is there.

---

### 6. Docker quick start (single container)

```
docker run -d \
  --name clickhouse-server \
  -p 8123:8123 \
  -p 9000:9000 \
  --ulimit nofile=262144:262144 \
  -v ch_data:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

**Breaking down every flag:**
- `-d` — run in the background (detached), instead of tying up your terminal
- `--name clickhouse-server` — gives the container a friendly name so you can refer to it later (`docker exec`, `docker stop`, etc.) instead of a random ID
- `-p 8123:8123` / `-p 9000:9000` — maps the container's internal ports to the same ports on your host machine, so the *same* HTTP/native ports from Part 1 are reachable from outside the container
- `--ulimit nofile=262144:262144` — raises the max number of open file handles. ClickHouse can open a large number of files (one per data part, roughly), and the container's default limit is often too low — this avoids "too many open files" errors under load
- `-v ch_data:/var/lib/clickhouse` — creates a **named Docker volume** and mounts it at ClickHouse's data directory. Without this, your data disappears the moment the container is deleted

Then:
- `docker exec -it clickhouse-server clickhouse-client` — runs the client *inside* the already-running container, dropping you into an interactive SQL shell
- `curl 'http://localhost:8123/?query=...'` — proves the HTTP interface works, from *outside* the container, using the port mapping above
- The closing note is a genuinely easy trap: mapping a port with `-p` only opens it on the container/Docker side. If `firewalld` is still active on the host, it will still block the traffic — so you need the same `firewall-cmd` commands from Part 1, even though you're now using Docker.

---

### 7. Docker Compose (server with custom config mounted)

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - ch_data:/var/lib/clickhouse
      - ./config.xml:/etc/clickhouse-server/config.d/custom-config.xml
      - ./users.xml:/etc/clickhouse-server/users.d/custom-users.xml
    environment:
      CLICKHOUSE_DB: observability
      CLICKHOUSE_USER: student
      CLICKHOUSE_PASSWORD: changeme
```

**Why you'd use this over plain `docker run`:** it's the same container as step 6, but declared in a file instead of a long command line — easier to version-control, reuse, and extend.

**What's new here compared to step 6:**
- Two extra volume mounts drop *your own* XML files into ClickHouse's drop-in config directories (`config.d/`, `users.d/`) — this is how you customize settings without editing the image itself
- The `environment` block uses ClickHouse's Docker image's built-in bootstrap variables to auto-create a database, a user, and a password the *first time* the container starts (on an empty data volume)
- `docker compose up -d` starts everything defined in the file, in the background

---

## Part 3 — Where everything lives on disk

```
/etc/clickhouse-server/config.xml     main server config
/etc/clickhouse-server/users.xml      users, profiles, quotas
/etc/clickhouse-server/config.d/      drop-in config overrides
/etc/clickhouse-server/users.d/       drop-in user overrides
/var/lib/clickhouse/                  data directory
/var/log/clickhouse-server/           logs
```

**Why this list matters:** these paths only apply to the **native install** (Part 1) directly on the filesystem. In the **Docker** setup (Part 2), these same paths exist *inside the container* — which is exactly why step 7 mounts local files onto `config.d/` and `users.d/`: it's injecting host files into those container-internal paths.

The `.d/` directories (`config.d/`, `users.d/`) are the important habit to take away: instead of editing `config.xml` or `users.xml` directly (which gets overwritten on upgrades), you drop small XML files into these folders and ClickHouse merges them in automatically.

---

## Part 4 — `config.xml` and `users.xml` in detail

These are the two files everything in Part 3 revolves around. They're both plain XML, both live under `/etc/clickhouse-server/`, and both get merged with drop-in fragments at startup — but they control completely different things.

### `config.xml` — how the *server* behaves

This file controls the server process itself: what it listens on, where it stores data, how it logs, and how it talks to other servers in a cluster. Nothing in here is about *who* can log in — that's `users.xml`'s job.

```xml
<?xml version="1.0"?>
<clickhouse>
    <!-- Network -->
    <listen_host>0.0.0.0</listen_host>
    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <interserver_http_port>9009</interserver_http_port>

    <!-- Where data and logs live on disk -->
    <path>/var/lib/clickhouse/</path>
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    </logger>

    <!-- Which file defines users/profiles/quotas -->
    <users_config>users.xml</users_config>
    <default_profile>default</default_profile>
    <default_database>default</default_database>

    <!-- Cluster topology, only needed if you're clustering -->
    <remote_servers>
        <my_cluster>
            <shard>
                <replica>
                    <host>node1</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>

    <!-- Keeper/ZooKeeper coordination, only needed if you're clustering -->
    <zookeeper>
        <node>
            <host>node1</host>
            <port>2181</port>
        </node>
    </zookeeper>
</clickhouse>
```

**Section by section:**

| Block | What it controls |
|---|---|
| `listen_host`, `http_port`, `tcp_port`, `interserver_http_port` | The exact ports and interfaces from the firewall section earlier — `0.0.0.0` means "listen on every network interface," not just `localhost` |
| `path`, `tmp_path` | Where table data and temporary query files actually get written — this is `/var/lib/clickhouse/` from the paths list |
| `logger` | Log verbosity and file locations — this is what fills `/var/log/clickhouse-server/` |
| `users_config` | Literally tells the server which file to go read next for user definitions — by default `users.xml`, but you could point it elsewhere |
| `remote_servers` | Defines shards and replicas for a cluster — this is the block referenced back in Part 1, section 4 |
| `zookeeper` | Points every node at the same Keeper/ZooKeeper quorum, so replicas can coordinate — pairs with the 2181/2888/3888 ports opened earlier |

You rarely edit `config.xml` itself in a real deployment — instead you drop a small file into `config.d/` containing just the tags you want to override, e.g.:

```xml
<!-- /etc/clickhouse-server/config.d/network.xml -->
<clickhouse>
    <listen_host>::</listen_host>
</clickhouse>
```

ClickHouse reads `config.xml` first, then merges every `*.xml` file in `config.d/` on top of it, alphabetically. Same tag name → your override wins; the base file stays untouched, which means an upgrade that replaces `config.xml` doesn't wipe out your customizations.

---

### `users.xml` — who can connect, and what they're allowed to do

This file defines three things, each in its own top-level block: **users** (identity), **profiles** (settings limits), and **quotas** (resource caps over time).

```xml
<?xml version="1.0"?>
<clickhouse>
    <users>
        <student>
            <password_sha256_hex>8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92</password_sha256_hex>
            <networks>
                <ip>10.0.0.0/8</ip>
            </networks>
            <profile>analyst</profile>
            <quota>default</quota>
            <access_management>0</access_management>
        </student>
    </users>

    <profiles>
        <analyst>
            <max_memory_usage>5000000000</max_memory_usage>
            <max_execution_time>30</max_execution_time>
            <readonly>1</readonly>
        </analyst>
    </profiles>

    <quotas>
        <default>
            <interval>
                <duration>3600</duration>
                <queries>1000</queries>
                <errors>100</errors>
                <result_rows>1000000000</result_rows>
            </interval>
        </default>
    </quotas>
</clickhouse>
```

**Section by section:**

| Block | What it controls |
|---|---|
| `<users>` | One entry per user. `password_sha256_hex` stores a *hash*, never the plaintext password. `networks` restricts which IPs/subnets that user is even allowed to connect from. `profile` and `quota` link the user to the other two sections below |
| `<profiles>` | Named bundles of query-level settings — memory limits, timeouts, read-only mode. A profile is assigned to a user, not the other way around, so many users can share one profile |
| `<quotas>` | Rate limits measured over a time window — e.g. "at most 1000 queries and 100 errors per hour." This caps *usage over time*, whereas a profile caps a *single query* |

**Why the split matters in practice:** if you want to add a new read-only analyst account, you don't touch `config.xml` at all — `users.xml` (or a file in `users.d/`) is the entire change. Conversely, if you're changing which port the server listens on, that's purely `config.xml` — `users.xml` never comes into it.

Just like the server config, you're not meant to edit `users.xml` directly — drop a file into `users.d/` instead:

```xml
<!-- /etc/clickhouse-server/users.d/student.xml -->
<clickhouse>
    <users>
        <student>
            <password_sha256_hex>...</password_sha256_hex>
            <profile>analyst</profile>
            <quota>default</quota>
        </student>
    </users>
</clickhouse>
```

One more practical detail: changes to `users.xml`/`users.d/` don't require restarting the server — running `SYSTEM RELOAD CONFIG` (or just waiting for the periodic auto-reload) picks them up live. `config.xml` changes are more of a mixed bag — some settings hot-reload the same way, others (like `listen_host` or `tcp_port`) need a full server restart to take effect.

**One alternative worth knowing about:** everything above is the *file-based* approach. ClickHouse also supports **SQL-driven access control** (`CREATE USER`, `GRANT`, `CREATE QUOTA` run as SQL statements instead of XML), which ClickHouse's own docs now recommend for anything beyond a handful of static users — it avoids editing XML and restarting/reloading entirely. File-based `users.xml` is still the default and is what the Docker Compose `CLICKHOUSE_USER`/`CLICKHOUSE_PASSWORD` environment variables from Part 2 are populating under the hood on first boot.

---

## TL;DR — the two paths, side by side

| | Native install (Part 1) | Docker (Part 2) |
|---|---|---|
| Install target | The OS itself, via `dnf` | A container, via Docker |
| Ports | Opened directly with `firewall-cmd` | Mapped with `-p`, **then still** need `firewall-cmd` |
| Data location | `/var/lib/clickhouse/` on the host | Named volume (`ch_data`) unless mounted elsewhere |
| Config customization | Edit files in `config.d/` / `users.d/` directly | Mount host files onto those same container paths |
| Clustering | Add Keeper ports + repeat install on every node | Same idea, but each node is a container instead |

Both paths end up in the same place — a running `clickhouse-server` process listening on 8123 (HTTP) and 9000 (native) — just reached through different tooling. And regardless of which path you took, the two config files behave the same way once the server is up: `config.xml` shapes the server, `users.xml` shapes who's allowed to use it.
