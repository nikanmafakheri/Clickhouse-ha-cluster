# ClickHouse Distributed Cluster Architecture (High Availability)

[![ClickHouse](https://img.shields.io/badge/ClickHouse-26.10.3-yellow)](https://clickhouse.com/)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

A production-grade, highly available (HA), and fault-tolerant ClickHouse cluster deployed via Docker Compose. This architecture demonstrates distributed data processing, sharding, asynchronous replication, and consensus management using ClickHouse Keeper.

## 🏗️ Architecture Topology

This infrastructure provisions a 9-node distributed system consisting of two primary layers:

1. **Consensus Layer (ClickHouse Keeper):** 
   - 3 nodes (`keeper-01`, `keeper-02`, `keeper-03`)
   - Replaces ZooKeeper to manage cluster metadata, quorum, and replica synchronization using the Raft consensus algorithm.
2. **Storage & Compute Layer (ClickHouse Servers):** 
   - 6 nodes distributed across **2 Shards** (3 Replicas per shard).
   - `Shard 1`: clickhouse-01, clickhouse-02, clickhouse-03
   - `Shard 2`: clickhouse-04, clickhouse-05, clickhouse-06

## ✨ Key Features

- **Infrastructure as Code (IaC):** Entire 9-node cluster bootstrapped via a single `docker-compose.yml`.
- **Automatic Data Sharding:** Utilizes a `Distributed` table engine hashing by `PULocationID` for optimal data locality and query performance.
- **Asynchronous Replication:** `ReplicatedMergeTree` engine ensures zero data loss and eventual consistency across shard replicas.
- **Self-Healing & Fault Tolerance:** No single point of failure (SPOF). Nodes automatically sync missing data parts upon recovery.

---

## 🚀 Getting Started

### 1. Clone the Repository
```bash
git clone https://github.com/nikanmafakheri/clickhouse-distributed-cluster.git
cd clickhouse-distributed-cluster
```

### 2. Launch the Cluster
Deploy the infrastructure in detached mode. The internal `<secret>` token handles secure node-to-node authentication.
```bash
docker compose up -d
```
Verify all 9 containers are running successfully:
```bash
docker compose ps
```

### 3. Initialize the Database Schema
Connect to the primary node's client:
```bash
docker compose exec clickhouse-01 clickhouse-client
```

Create the underlying physical tables (`ReplicatedMergeTree`) across the cluster:
```sql
CREATE TABLE trip_data_local ON CLUSTER my_cluster (
    VendorID UInt8,
    tpep_pickup_datetime DateTime,
    tpep_dropoff_datetime DateTime,
    passenger_count UInt8,
    trip_distance Float32,
    PULocationID UInt16,
    DOLocationID UInt16,
    fare_amount Float32,
    total_amount Float32
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/trip_data',
    '{replica}'
)
PARTITION BY toYYYYMM(tpep_pickup_datetime)
ORDER BY (PULocationID, tpep_pickup_datetime);
```

Create the Distributed routing table (Load Balancer):
```sql
CREATE TABLE trip_data_distributed ON CLUSTER my_cluster AS trip_data_local
ENGINE = Distributed(
    my_cluster,
    default,
    trip_data_local,
    murmurHash3_64(PULocationID)
);
```

---

## 🧪 Load Testing (Synthetic Data Ingestion)

To benchmark the cluster without relying on external network I/O, we use the internal `generateRandom` engine to stream 1,000,000 records mimicking the NYC Taxi dataset directly into RAM and onto the disks.

```sql
INSERT INTO trip_data_distributed
SELECT * FROM generateRandom('
    VendorID UInt8,
    tpep_pickup_datetime DateTime,
    tpep_dropoff_datetime DateTime,
    passenger_count UInt8,
    trip_distance Float32,
    PULocationID UInt16,
    DOLocationID UInt16,
    fare_amount Float32,
    total_amount Float32
') LIMIT 1000000
SETTINGS max_partitions_per_insert_block = 0;
```

**Verify Data Distribution:**
Confirm that the rows are evenly sharded and perfectly replicated across the nodes:
```sql
SELECT
    hostName() AS node,
    count() AS rows_on_disk
FROM clusterAllReplicas('my_cluster', default, trip_data_local)
GROUP BY node
ORDER BY node;
```

---

## 💥 Chaos Engineering (Fault Tolerance Test)

Prove the high availability (HA) and self-healing nature of the architecture by intentionally crashing the primary node.

1. **Kill a Node:**
   ```bash
   docker compose stop clickhouse-01
   ```
2. **Execute a Distributed Query:**
   Connect to a surviving node (`clickhouse-02`) and run a cluster-wide count. The query will route around the dead node seamlessly.
   ```sql
   SELECT count() FROM trip_data_distributed;
   ```
3. **Trigger Auto-Recovery (Self-Healing):**
   ```bash
   docker compose start clickhouse-01
   ```
   *Node 1 will automatically consult ClickHouse Keeper, download missing data parts from its replicas, and rejoin the cluster in a synced state.*

---

## 👨‍💻 Author

**Nikan Mafakheri**
- GitHub: [@nikanmafakheri](https://github.com/nikanmafakheri)
- Instagram: [@devbynikan](https://instagram.com/devbynikan)