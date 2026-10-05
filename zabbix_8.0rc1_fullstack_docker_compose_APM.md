Here is the **Complete, Ultimate Guide to Deploying Zabbix 8.0 APM with Docker**, incorporating every fix and workaround we discovered.
Because APM is a brand-new feature, there are several "gotchas" in the current RC1 build (like missing compilation flags and missing ClickHouse auto-schemas).

---

### Folder Structure
Create a new directory for your stack and create these three files:
```text
zabbix-8-apm/
├── docker-compose.yml
├── Dockerfile-proxy
└── otel_schema.sql
```

---

### File 1: `Dockerfile-proxy`
*The Problem:* The official Zabbix 8.0 RC1 Docker images are currently built *without* the `--with-apm` flag. 
*The Solution:* We use this custom Dockerfile to download the source, install gRPC/Protobuf dependencies, compile the proxy with APM enabled, and replace the binary.

```dockerfile
# Use the official Ubuntu-based proxy as the base
FROM zabbix/zabbix-proxy-mysql:ubuntu-trunk

USER root

# Install dependencies required for compiling Zabbix with APM (OpenTelemetry)
RUN apt-get update && apt-get install -y \
    wget build-essential pkg-config libmysqlclient-dev libevent-dev \
    libpcre2-dev zlib1g-dev libcurl4-openssl-dev libssl-dev \
    libgrpc++-dev protobuf-compiler-grpc libprotobuf-dev

# Download Zabbix 8.0.0rc1 source code, compile with APM, and overwrite binary
RUN wget https://cdn.zabbix.com/zabbix/sources/release_candidates/8.0/zabbix-8.0.0rc1.tar.gz && \
    tar -zxvf zabbix-8.0.0rc1.tar.gz && \
    cd zabbix-8.0.0rc1 && \
    ./configure --enable-proxy --with-mysql --with-apm --with-openssl --enable-ipv6 --with-libcurl && \
    make && \
    cp src/zabbix_proxy/zabbix_proxy /usr/sbin/zabbix_proxy && \
    cd .. && rm -rf zabbix-8.0.0rc1* && apt-get clean

# Switch back to the default zabbix user
USER 1997
```

---

### File 2: `docker-compose.yml`
*Key Fixes Included:* 
- Uses `ZBX_TELEMETRYPROVIDER_0` (indexed variable required by Docker entrypoint).
- Builds the custom proxy image.
- Uses one MySQL container for both Server and Proxy (Proxy auto-creates the `zabbix_proxy` DB using `MYSQL_ROOT_PASSWORD`).

```yaml
version: '3.8'

services:
  zabbix-db:
    image: mysql:9.7
    container_name: zabbix-mysql-server
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_bin
    volumes:
      - zabbix-db-storage:/var/lib/mysql
    environment:
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_password
      - MYSQL_ROOT_PASSWORD=root_secure_password
    networks:
      - zabbix-net
    restart: always

  zabbix-clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: zabbix-clickhouse-server
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-storage:/var/lib/clickhouse
    environment:
      - CLICKHOUSE_DB=zabbix
      - CLICKHOUSE_USER=zabbix
      - CLICKHOUSE_PASSWORD=zabbix_password
      - CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1
    networks:
      - zabbix-net
    restart: always

  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-trunk
    container_name: zabbix-server-mysql
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=zabbix-db
      - DB_SERVER_PORT=3306
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_password
      - MYSQL_ROOT_PASSWORD=root_secure_password
      - ZBX_STARTCONNECTORS=1
      - ZBX_TELEMETRYPROVIDER_0=clickhouse;url=http://zabbix-clickhouse:8123,db=zabbix,username=zabbix,password=zabbix_password
    networks:
      - zabbix-net
    depends_on:
      - zabbix-db
      - zabbix-clickhouse
    restart: always

  zabbix-proxy:
    build:
      context: .
      dockerfile: Dockerfile-proxy
    image: custom-zabbix-proxy-apm:latest
    container_name: zabbix-proxy-mysql
    ports:
      - "10061:10051"
      - "4317:4317" # OpenTelemetry gRPC Port
    environment:
      - DB_SERVER_HOST=zabbix-db
      - DB_SERVER_PORT=3306
      - MYSQL_DATABASE=zabbix_proxy
      - MYSQL_USER=zabbix_proxy
      - MYSQL_PASSWORD=zabbix_proxy_password
      - MYSQL_ROOT_PASSWORD=root_secure_password
      - ZBX_SERVER_HOST=zabbix-server
      - ZBX_SERVER_PORT=10051
      - ZBX_HOSTNAME=zabbix-proxy-apm
      - ZBX_STARTAPMCOLLECTORS=1
      - ZBX_APMLISTENIP=0.0.0.0
      - ZBX_APMLISTENPORT=4317
      - ZBX_TELEMETRYPROVIDER_0=clickhouse;url=http://zabbix-clickhouse:8123,db=zabbix,username=zabbix,password=zabbix_password
    networks:
      - zabbix-net
    depends_on:
      - zabbix-server
      - zabbix-db
      - zabbix-clickhouse
    restart: always

  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:ubuntu-trunk
    container_name: zabbix-web-apache
    ports:
      - "8086:8080"
    environment:
      - DB_SERVER_HOST=zabbix-db
      - DB_SERVER_PORT=3306
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_password
      - ZBX_SERVER_HOST=zabbix-server
      - ZBX_SERVER_PORT=10051
      - PHP_TZ=UTC
    networks:
      - zabbix-net
    depends_on:
      - zabbix-db
      - zabbix-server
    restart: always

networks:
  zabbix-net:
    driver: bridge

volumes:
  zabbix-db-storage:
  clickhouse-storage:
```

---

### File 3: `otel_schema.sql`
*The Problem:* The Zabbix Proxy auto-creates MySQL schemas, but it does *not* auto-create the ClickHouse OpenTelemetry tables, resulting in `UNKNOWN_TABLE` errors.
*The Solution:* We reverse-engineered the required schema (including the missing `ScopeName` and `ScopeVersion` columns).

```sql
CREATE DATABASE IF NOT EXISTS zabbix;

CREATE TABLE IF NOT EXISTS zabbix.otel_traces (
     Timestamp DateTime64(9) CODEC(Delta, ZSTD(1)),
     TraceId String CODEC(ZSTD(1)),
     SpanId String CODEC(ZSTD(1)),
     ParentSpanId String CODEC(ZSTD(1)),
     TraceState String CODEC(ZSTD(1)),
     SpanName LowCardinality(String) CODEC(ZSTD(1)),
     SpanKind LowCardinality(String) CODEC(ZSTD(1)),
     ServiceName LowCardinality(String) CODEC(ZSTD(1)),
     ResourceAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
     ScopeName String CODEC(ZSTD(1)),
     ScopeVersion String CODEC(ZSTD(1)),
     SpanAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
     Duration Int64 CODEC(ZSTD(1)),
     StatusCode LowCardinality(String) CODEC(ZSTD(1)),
     StatusMessage String CODEC(ZSTD(1)),
     Events Nested (Timestamp DateTime64(9), Name LowCardinality(String), Attributes Map(LowCardinality(String), String)) CODEC(ZSTD(1)),
     Links Nested (TraceId String, SpanId String, TraceState String, Attributes Map(LowCardinality(String), String)) CODEC(ZSTD(1)),
     INDEX idx_trace_id TraceId TYPE bloom_filter(0.001) GRANULARITY 1,
     INDEX idx_duration Duration TYPE minmax GRANULARITY 1
) ENGINE = MergeTree() PARTITION BY toDate(Timestamp) ORDER BY (ServiceName, SpanName, toUnixTimestamp(Timestamp), TraceId);

CREATE TABLE IF NOT EXISTS zabbix.otel_logs (
    Timestamp DateTime64(9) CODEC(Delta, ZSTD(1)),
    TraceId String CODEC(ZSTD(1)),
    SpanId String CODEC(ZSTD(1)),
    TraceFlags UInt32 CODEC(ZSTD(1)),
    SeverityText LowCardinality(String) CODEC(ZSTD(1)),
    SeverityNumber Int32 CODEC(ZSTD(1)),
    ServiceName LowCardinality(String) CODEC(ZSTD(1)),
    Body String CODEC(ZSTD(1)),
    ResourceAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    LogAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    INDEX idx_trace_id TraceId TYPE bloom_filter(0.001) GRANULARITY 1
) ENGINE = MergeTree() PARTITION BY toDate(Timestamp) ORDER BY (ServiceName, SeverityText, toUnixTimestamp(Timestamp), TraceId);

CREATE TABLE IF NOT EXISTS zabbix.otel_metrics_sum (
    ResourceAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    ResourceSchemaUrl String CODEC(ZSTD(1)),
    ScopeName String CODEC(ZSTD(1)),
    ScopeVersion String CODEC(ZSTD(1)),
    ScopeAttributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    ScopeDroppedAttrCount UInt32 CODEC(ZSTD(1)),
    ScopeSchemaUrl String CODEC(ZSTD(1)),
    MetricName String CODEC(ZSTD(1)),
    MetricDescription String CODEC(ZSTD(1)),
    MetricUnit String CODEC(ZSTD(1)),
    Attributes Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    StartTimeUnix DateTime64(9) CODEC(Delta, ZSTD(1)),
    TimeUnix DateTime64(9) CODEC(Delta, ZSTD(1)),
    Value Float64 CODEC(ZSTD(1)),
    Flags UInt32 CODEC(ZSTD(1)),
    Exemplars Nested (FilteredAttributes Map(LowCardinality(String), String), TimeUnix DateTime64(9), Value Float64, SpanId String, TraceId String) CODEC(ZSTD(1)),
    AggregationTemporality Int32 CODEC(ZSTD(1)),
    IsMonotonic Boolean CODEC(Delta, ZSTD(1))
) ENGINE = MergeTree() PARTITION BY toDate(TimeUnix) ORDER BY (MetricName, Attributes, toUnixTimestamp(TimeUnix));

CREATE TABLE IF NOT EXISTS zabbix.otel_metrics_gauge AS zabbix.otel_metrics_sum ENGINE = MergeTree() PARTITION BY toDate(TimeUnix) ORDER BY (MetricName, Attributes, toUnixTimestamp(TimeUnix));
CREATE TABLE IF NOT EXISTS zabbix.otel_metrics_histogram AS zabbix.otel_metrics_sum ENGINE = MergeTree() PARTITION BY toDate(TimeUnix) ORDER BY (MetricName, Attributes, toUnixTimestamp(TimeUnix));
CREATE TABLE IF NOT EXISTS zabbix.otel_metrics_exponential_histogram AS zabbix.otel_metrics_sum ENGINE = MergeTree() PARTITION BY toDate(TimeUnix) ORDER BY (MetricName, Attributes, toUnixTimestamp(TimeUnix));
```

---

### The Deployment Playbook

**Step 1: Build and Start the Stack**
Run this in your directory to build the custom proxy and start all services:
```bash
docker compose up -d --build
```

**Step 2: Inject the ClickHouse Schema**
Wait 30 seconds for ClickHouse to fully initialize, then run:
```bash
cat otel_schema.sql | docker exec -i zabbix-clickhouse-server clickhouse-client --user=zabbix --password=zabbix_password
```

**Step 3: Configure Zabbix Web UI**
1. Log in to `http://<YOUR_IP>:8086` (Admin / zabbix).
2. Go to **Administration -> Proxies**. Click **Create proxy**.
   - Proxy name: `zabbix-proxy-apm`
   - Proxy mode: `Active`
   - APM Tab: Check **Data collection enabled**. Click **Add**.
3. Go to **Administration -> Data source -> APM** (or General -> APM).
   - Type: `ClickHouse`
   - URL: `http://zabbix-clickhouse:8123`
   - Database / User / Password: `zabbix` / `zabbix` / `zabbix_password`
   - Click **Update/Save**.

**Step 4: Restart the Proxy (The Chicken & Egg Fix)**
Because the proxy refuses to open port 4317 until it receives the "APM Enabled" flag from the Server *and* verifies the ClickHouse tables exist, you must restart it now:
```bash
docker restart zabbix-proxy-mysql
```

---

### Testing the Setup

**1. Generate Dummy Data:**
Find your docker network name (`docker network ls`), then run the official OpenTelemetry generator:
```bash
docker run --rm --network zabbix-8_zabbix-net \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  traces --otlp-insecure --otlp-endpoint="zabbix-proxy-mysql:4317" --rate=5 --duration=60s
```

**2. View the Data in Zabbix:**
- Go to **Monitoring -> APM** (or **Traces**).
- **CRITICAL:** Ensure the time filter in the top right is set to **"Last 1 hour"**. (Timezone differences often hide live data).
- Click on the `telemetrygen` service to view your trace waterfalls!
