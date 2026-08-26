# Install ClickHouse using Docker

ClickHouse provides official Docker images available on Docker Hub. These images use the official ClickHouse deb packages and are a convenient way to get started quickly.

## Quick Start

Pull the latest ClickHouse server image:

```bash
docker pull clickhouse/clickhouse-server
```

## Versions

- `latest` - Points to the latest release of the latest stable branch
- Branch tags (e.g., `22.2`) - Point to the latest release of the corresponding branch
- Full version tags (e.g., `22.2.3`, `22.2.3.5`) - Point to specific releases
- `head` - Built from the latest commit to the default branch
- Optional `-alpine` suffix - Available for all tags to use Alpine Linux base image

## System Requirements

### Architecture Compatibility

**amd64 image:**
- Requires x86-64-v3 microarchitecture level support (AVX2, BMI1, BMI2, F16C, FMA, LZCNT, MOVBE, XSAVE)
- Virtually all x86 CPUs after 2015 are compatible

**arm64 image:**
- Requires ARMv8.2-A architecture support with Load-Acquire RCpc register
- Supported on: Graviton >=2, Azure instances, GCP instances
- Not supported on: Raspberry Pi 4, Jetson AGX Xavier/Orin

**Ubuntu 22.04 base (since ClickHouse 24.11):**
- Requires Docker version >= 20.10.10
- Workaround: Use `docker run --security-opt seccomp=unconfined` (note security implications)

## Basic Usage

### Start a Server Instance

```bash
docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 clickhouse/clickhouse-server
```

By default:
- ClickHouse is accessible only via the Docker network
- Runs as the `default` user without a password

### Connect with Native Client

Method 1 - Using container network:
```bash
docker run -it --rm --network=container:some-clickhouse-server --entrypoint clickhouse-client clickhouse/clickhouse-server
```

Method 2 - Using docker exec:
```bash
docker exec -it some-clickhouse-server clickhouse-client
```

### Connect with curl (HTTP Interface)

```bash
echo "SELECT 'Hello, ClickHouse!'" | docker run -i --rm --network=container:some-clickhouse-server buildpack-deps:curl curl 'http://localhost:8123/?query=' -s --data-binary @-
```

### Stop and Remove Container

```bash
docker stop some-clickhouse-server
docker rm some-clickhouse-server
```

## Networking

By default, the `default` user doesn't have network access unless a password is set.

### Expose Ports for External Access

Map container ports to host ports using port numbers:

```bash
docker run -d \
    -p 8123:8123 \
    -p 9000:9000 \
    -e CLICKHOUSE_PASSWORD=changeme \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    clickhouse/clickhouse-server
```

Access ClickHouse via port numbers:

**HTTP Interface (Port 8123):**
```bash
echo 'SELECT version()' | curl 'http://localhost:8123/?password=changeme' --data-binary @-
```

**Native Client (Port 9000):**
```bash
clickhouse-client --host localhost --port 9000 --password changeme
```

### Port Configuration

- `-p 8123:8123` - HTTP interface accessible on port 8123
- `-p 9000:9000` - Native client (TCP) accessible on port 9000

You can map to different host ports if needed:

```bash
docker run -d \
    -p 18123:8123 \
    -p 19000:9000 \
    -e CLICKHOUSE_PASSWORD=changeme \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    clickhouse/clickhouse-server

# Access on custom ports
echo 'SELECT version()' | curl 'http://localhost:18123/?password=changeme' --data-binary @-
```

## Persistent Storage

Mount volumes to persist data and logs:

```bash
docker run -d \
    -v "$PWD/ch_data:/var/lib/clickhouse/" \
    -v "$PWD/ch_logs:/var/log/clickhouse-server/" \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    clickhouse/clickhouse-server
```

### Volumes to Mount

**Essential for persistence:**
- `/var/lib/clickhouse/` - Main folder where ClickHouse stores data
- `/var/log/clickhouse-server/` - Server logs

**Optional for configuration:**
- `/etc/clickhouse-server/config.d/*.xml` - Server configuration adjustments
- `/etc/clickhouse-server/users.d/*.xml` - User settings adjustments
- `/docker-entrypoint-initdb.d/` - Database initialization scripts

## Configuration

ClickHouse uses a `config.xml` file for server configuration.

### Start with Custom Configuration

```bash
docker run -d \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    -v /path/to/your/config.xml:/etc/clickhouse-server/config.xml \
    clickhouse/clickhouse-server
```

### Network Ports

- **8123** - HTTP interface
- **9000** - Native client interface (TCP)

## Running as Custom User

```bash
# Ensure $PWD/data/clickhouse and $PWD/logs/clickhouse exist and are owned by your user
docker run --rm \
    --user "${UID}:${GID}" \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    -v "$PWD/logs/clickhouse:/var/log/clickhouse-server" \
    -v "$PWD/data/clickhouse:/var/lib/clickhouse" \
    clickhouse/clickhouse-server
```

## Running as Root

```bash
docker run --rm \
    -e CLICKHOUSE_RUN_AS_ROOT=1 \
    --name clickhouse-server-userns \
    -v "$PWD/logs/clickhouse:/var/log/clickhouse-server" \
    -v "$PWD/data/clickhouse:/var/lib/clickhouse" \
    clickhouse/clickhouse-server
```

## Linux Capabilities

ClickHouse has advanced functionality that requires specific Linux capabilities:

```bash
docker run -d \
    --cap-add=SYS_NICE \
    --cap-add=NET_ADMIN \
    --cap-add=IPC_LOCK \
    --name some-clickhouse-server \
    --ulimit nofile=262144:262144 \
    clickhouse/clickhouse-server
```

These capabilities are optional but enable advanced features. See the ClickHouse documentation for more details on configuring CAP_IPC_LOCK and CAP_SYS_NICE capabilities.

## Creating Users and Databases on Startup

Use environment variables to set up default database and user:

```bash
docker run --rm \
    -e CLICKHOUSE_DB=my_database \
    -e CLICKHOUSE_USER=username \
    -e CLICKHOUSE_PASSWORD=password \
    -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 \
    -p 9000:9000/tcp \
    clickhouse/clickhouse-server
```

### Managing the Default User

**Disable network access for default user:**
- By default, if `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`, or `CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT` are not set, the `default` user has no network access

**Allow insecure access for default user:**
```bash
docker run --rm \
    -e CLICKHOUSE_SKIP_USER_SETUP=1 \
    -p 9000:9000/tcp \
    clickhouse/clickhouse-server
```

## Custom Initialization Scripts

You can extend the image by adding initialization scripts to `/docker-entrypoint-initdb.d/`:

Supported file types:
- `*.sql` - SQL scripts
- `*.sql.gz` - Gzipped SQL scripts
- `*.sh` - Shell scripts (executable or non-executable)

Scripts are executed in **alphabetical order by filename**.

### Example Initialization Script

Create `/docker-entrypoint-initdb.d/init-db.sh`:

```bash
#!/bin/bash
set -e

clickhouse-client -n <<-EOSQL
    CREATE DATABASE docker;
    CREATE TABLE docker.docker (x Int32) ENGINE = MergeTree
    ORDER BY ();
EOSQL
```

You can pass environment variables to the client during initialization:

```bash
docker run --rm \
    -e CLICKHOUSE_USER=myuser \
    -e CLICKHOUSE_PASSWORD=mypassword \
    -v /path/to/init-db.sh:/docker-entrypoint-initdb.d/init-db.sh \
    clickhouse/clickhouse-server
```

## Complete Example

A production-ready setup with persistent storage, custom user, and initialization:

```bash
# Create data directories
mkdir -p ch_data ch_logs

# Run container
docker run -d \
    --name clickhouse-server \
    --ulimit nofile=262144:262144 \
    -p 8123:8123 \
    -p 9000:9000 \
    -e CLICKHOUSE_DB=production_db \
    -e CLICKHOUSE_USER=admin \
    -e CLICKHOUSE_PASSWORD=your_secure_password \
    -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 \
    -v "$PWD/ch_data:/var/lib/clickhouse/" \
    -v "$PWD/ch_logs:/var/log/clickhouse-server/" \
    clickhouse/clickhouse-server

# Verify installation
echo 'SELECT version()' | curl 'http://localhost:8123/?user=admin&password=your_secure_password' --data-binary @-
```

## References

- [ClickHouse Client Documentation](/docs/concepts/features/interfaces/cli)
- [ClickHouse HTTP Interface](/docs/concepts/features/interfaces/http)
- [ClickHouse Configuration Files](/docs/concepts/features/configuration/server-config/configuration-files)
- [Docker Hub - ClickHouse Server](https://hub.docker.com/r/clickhouse/clickhouse-server/)
