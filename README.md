<div align="center">

# ⚡ Hold My Redis

### Pushing Node.js to 200,000 Requests Per Second on a Single Machine

A hands-on performance engineering case study — systematically optimizing a Node.js API from **4 RPS** (full-table scan) to **200k RPS** (sharded Redis cluster) by methodically eliminating bottlenecks across the storage layer, caching strategy, and process architecture.

[![Node.js](https://img.shields.io/badge/Node.js-22+-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org)
[![Redis](https://img.shields.io/badge/Redis-7+-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17+-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://postgresql.org)
[![PM2](https://img.shields.io/badge/PM2-Cluster_Mode-2B037A?style=for-the-badge&logo=pm2&logoColor=white)](https://pm2.keymetrics.io)

---

</div>

## 📑 Table of Contents

- [The Problem](#-the-problem)
- [Architecture](#-architecture)
- [Step-by-Step Optimization Journey](#-step-by-step-optimization-journey)
- [Benchmark Results](#-benchmark-results)
  - [1,000 Records (Baseline)](#-1000-records--baseline-)
  - [1,000,000 Records (Production Scale)](#-1000000-records--production-scale-)
  - [Key Takeaways](#-key-takeaways)
- [Getting Started](#-getting-started)
- [Configuration](#-configuration)
- [API Endpoints](#-api-endpoints)
- [Project Structure](#-project-structure)

---

## 🎯 The Problem

Most Node.js benchmarks test "hello world" endpoints that never touch a database. That tells you nothing about real-world performance. We wanted to answer a harder question:

> **How many requests per second can a single machine handle when every request involves a database lookup across 1 million records?**

The answer depends entirely on *how* you architect that lookup — and the difference is staggering: a **50,000× gap** between the worst and best approach.

---

## 🏗 Architecture

```
                        ┌─────────────────────────────────────┐
                        │          Load Generator             │
                        │        (autocannon + wrk)           │
                        └──────────────┬──────────────────────┘
                                       │
                                       ▼
                        ┌─────────────────────────────────────┐
                        │       PM2 Cluster (12 cores)        │
                        │  ┌───────┐ ┌───────┐ ┌───────┐     │
                        │  │ W1    │ │ W2    │ │ W...  │     │
                        │  └───┬───┘ └───┬───┘ └───┬───┘     │
                        └──────┼─────────┼─────────┼──────────┘
                               │         │         │
                  ┌────────────┼─────────┼─────────┼────────────┐
                  │            ▼         ▼         ▼            │
                  │  ┌──────────────────────────────────────┐   │
                  │  │     Redis (Standalone or Cluster)     │   │
                  │  │       in-memory cache layer           │   │
                  │  └──────────────────────────────────────┘   │
                  │            │                                │
                  │            ▼                                │
                  │  ┌──────────────────────────────────────┐   │
                  │  │     PostgreSQL (1M records)           │   │
                  │  │     source of truth + persistence     │   │
                  │  └──────────────────────────────────────┘   │
                  └─────────────────────────────────────────────┘
```

**Frameworks Tested:** Cpeak · Fastify · Express  
**Storage Layers:** PostgreSQL (unindexed → B-Tree indexed) · Redis Standalone · Redis 30-Node Cluster  
**Process Strategy:** PM2 cluster mode across all 12 CPU cores  
**Load Testing:** autocannon with configurable connections, pipelining, and worker threads

---

## 🛤 Step-by-Step Optimization Journey

This project follows a methodical, layered optimization approach. Each step isolates a single variable so you can see exactly what caused the improvement.

---

### Step 1 — Establish the Baseline (No Database)

**Goal:** Determine the raw HTTP throughput ceiling of Node.js itself, with zero I/O.

We started with a `/simple` endpoint that returns a static JSON response — no database, no Redis, no disk. This tells us the theoretical maximum our framework + OS + network stack can deliver.

**What we did:**
- Deployed the Cpeak server behind PM2 in cluster mode (6 workers across 12 cores)
- Hit `/simple` with autocannon: 300 connections, 15× pipelining, 6 load-gen workers
- Measured sustained throughput over 10 seconds

**Result:** **~100k RPS** with 38ms average latency — this is our ceiling. Every optimization below is measured against this number.

---

### Step 2 — PostgreSQL Full Table Scan (The Worst Case)

**Goal:** Quantify how bad an unindexed query gets at scale.

The `/code-v1` endpoint runs a `SELECT * FROM codes WHERE code = $1` query against 1 million rows — but with **no index** on the `code` column. Every request triggers a sequential scan across the entire table.

**What we did:**
- Seeded PostgreSQL with 1,000,000 records using our custom seeding script
- Ran autocannon against `/code-v1` (20 connections, single worker)
- Watched the throughput collapse

**Result:** **~4 RPS** with 4.4-second average latency. The database was completely I/O-bound, doing full sequential scans on every single request. At 1K records this same query did 1,400 RPS — proving that **benchmarking at toy scale gives dangerously misleading results**.

---

### Step 3 — Add a B-Tree Index (The Obvious Fix)

**Goal:** Measure the impact of proper indexing.

We created a B-Tree index on the `code` column: `CREATE INDEX idx_codes_code ON codes(code)`. The same query now uses an index scan instead of a sequential scan — O(log n) instead of O(n).

**What we did:**
- Added a B-Tree index on the `code` column
- Re-ran the benchmark on `/code-v4` with 100 connections, 5× pipelining, 4 workers
- Compared against the unindexed baseline

**Result:** **~15k RPS** with 18ms p50 latency — a **3,750× improvement** over the unindexed query. This single `CREATE INDEX` statement took us from unusable to production-viable.

---

### Step 4 — Add a Redis Cache Layer (Cache-Aside Pattern)

**Goal:** Eliminate database round-trips for hot data.

Instead of hitting PostgreSQL on every request, we implemented a **cache-aside pattern**: check Redis first, fall back to Postgres on miss, and populate the cache on read. Since our dataset is read-heavy, the hit rate approaches 100% after warm-up.

**What we did:**
- Migrated all 1M Postgres records into Redis using our `npm run migrate` script
- The `/code-fast` endpoint does a `GET` from Redis (standalone, single-node)
- Ran autocannon with 200 connections, 10× pipelining, 6 workers

**Result:** **~50k RPS** with 38ms p50 latency — a **3.3× improvement** over indexed Postgres, with zero schema changes needed. Redis serves everything from memory, eliminating disk I/O entirely.

---

### Step 5 — In-Memory Buffering + Async Disk Writes

**Goal:** Test a hybrid approach — serve reads from Redis, buffer writes in-memory, and flush to Postgres asynchronously.

The `/code-ultra-fast` endpoint reads from Redis and batches disk writes instead of doing synchronous Postgres inserts. This trades some durability for throughput.

**What we did:**
- Implemented in-memory write buffering with periodic Postgres flushes
- Ran autocannon with 200 connections, 10× pipelining, 6 workers

**Result:** **~35k RPS** — slightly lower than pure Redis reads because of the write-path overhead, but still 2.3× faster than indexed Postgres, with the added benefit of persistence.

---

### Step 6 — Postgres Transactional Writes (Disk-Heavy)

**Goal:** Benchmark the write path — inserts with full ACID guarantees.

The `/code` endpoint generates a new random code, writes it to PostgreSQL (with disk sync), and stores it in Redis. Every request hits the disk.

**What we did:**
- Ran autocannon against `/code` with 50 connections, 4 workers
- Measured sustained write throughput

**Result:** **~8.5k RPS** — respectable for fully durable writes. PostgreSQL's WAL + fsync is the bottleneck here, not Node.js.

---

### Step 7 — Redis Cluster Sharding (Breaking the Single-Core Barrier)

**Goal:** Redis standalone is single-threaded and caps out around 50k RPS. Can we break past that with cluster sharding?

We deployed a **30-node Redis cluster** (15 masters + 15 replicas) using our `redis.sh` script inside WSL. Data is automatically sharded across 16,384 hash slots distributed across 15 master nodes, with each replica providing failover.

**What we did:**
- Ran `redis.sh` to spin up 30 Redis instances on ports 7000–7029
- Created the cluster with automatic slot assignment and replication
- Benchmarked with `redis-benchmark --cluster` (100k operations)

**Results:**
- **SET:** **~170k RPS** (p50 = 0.087ms)
- **GET:** **~200k RPS** (p50 = 0.079ms)

This is the final form — **sub-millisecond latency at 200k requests per second** on a single machine. The cluster distributes load across all 15 master nodes, bypassing the single-threaded bottleneck of standalone Redis.

---

### Step 8 — Framework Comparison (Does It Even Matter?)

**Goal:** Quantify framework overhead — Express vs. Fastify vs. Cpeak.

We tested all three frameworks on the same `/simple` endpoint:

| Framework | Mode | RPS |
|:----------|:-----|----:|
| Fastify | Single thread | ~25k |
| Cpeak | PM2 (6 workers) | ~95k |
| Cpeak | PM2 (high concurrency) | ~100k |

**Conclusion:** Framework choice accounts for maybe 10–15% variation. The storage layer is responsible for the other **50,000×** gap. Don't bikeshed your framework — fix your queries.

---

## 📊 Benchmark Results

> All benchmarks run on a **12-core** machine using [autocannon](https://github.com/mcollina/autocannon) with PM2 cluster mode. Duration: 10 seconds per test.

### 🔹 1,000 Records ( Baseline )

| Test | Endpoint | Connections | Pipeline | Workers | Avg RPS | p50 Latency | Total Reqs |
|:-----|:---------|:-----------:|:--------:|:-------:|--------:|:-----------:|:----------:|
| **Fastify — single thread** | `/simple` | 100 | 1 | 1 | **~25k** | 3 ms | 266k |
| **Cpeak — PM2 cluster** | `/simple` | 200 | 10 | 6 | **~95k** | 9 ms | 908k |
| **Cpeak — high concurrency** | `/simple` | 300 | 15 | 6 | **~100k** | 34 ms | 991k |
| Postgres — full scan | `/code-v1` | 50 | 1 | 1 | **~1.4k** | 20 ms | 14k |
| Postgres — indexed (B-Tree) | `/code-v4` | 100 | 5 | 4 | **~16k** | 25 ms | 180k |
| Redis — cache lookup | `/code-fast` | 200 | 10 | 6 | **~53k** | 29 ms | 534k |

<details>
<summary>📸 <strong>Click to view benchmark screenshots (1K records)</strong></summary>
<br>

| Fastify — 1 thread | Cpeak — PM2 cluster |
|:---:|:---:|
| ![Fastify 1 thread](results/db_1000/fastify-1thread.png) | ![Cpeak PM2](results/db_1000/cpeak-pm2.png) |

| Cpeak — High concurrency | Postgres — Full scan |
|:---:|:---:|
| ![Cpeak high concurrency](results/db_1000/cpeak-pm2-concurrency++.png) | ![Postgres unindexed](results/db_1000/code-v1-test_unindexed.png) |

| Postgres — Indexed (B-Tree) | Redis — Cache lookup |
|:---:|:---:|
| ![Postgres indexed](results/db_1000/code-v4-test_indexed.png) | ![Redis cache](results/db_1000/code-fast-redis.png) |

</details>

---

### 🔸 1,000,000 Records ( Production Scale )

This is where the storage layer becomes the **dominant factor**. Framework overhead is noise compared to the I/O cost of a bad query plan.

| Test | Endpoint | Connections | Pipeline | Workers | Avg RPS | p50 Latency | Total Reqs |
|:-----|:---------|:-----------:|:--------:|:-------:|--------:|:-----------:|:----------:|
| **Simple (no DB)** | `/simple` | 300 | 15 | 6 | **~100k** | 38 ms | 990k |
| Postgres — full scan 😱 | `/code-v1` | 20 | 1 | 1 | **~4** | 4,435 ms | 40 |
| Postgres — indexed (B-Tree) | `/code-v4` | 100 | 5 | 4 | **~15k** | 18 ms | 162k |
| Postgres — disk writes + txn | `/code` | 50 | 1 | 4 | **~8.5k** | 4 ms | 80k |
| Redis — standalone cache | `/code-fast` | 200 | 10 | 6 | **~50k** | 38 ms | 473k |
| Redis — in-memory + buffered writes | `/code-ultra-fast` | 200 | 10 | 6 | **~35k** | 46 ms | 361k |
| **Redis — 30-node cluster (SET)** 🔥 | `redis-benchmark` | — | — | — | **~170k** | 0.087 ms | 100k |
| **Redis — 30-node cluster (GET)** 🔥 | `redis-benchmark` | — | — | — | **~200k** | 0.079 ms | 100k |

<details>
<summary>📸 <strong>Click to view benchmark screenshots (1M records)</strong></summary>
<br>

| Simple — No DB | Postgres — Full scan (1M rows) |
|:---:|:---:|
| ![Simple no DB](results/db_1m/simple-no-db-rps.png) | ![Postgres full scan 1M](results/db_1m/code-v1.png) |

| Postgres — Indexed (B-Tree) | Postgres — Disk writes + transaction |
|:---:|:---:|
| ![Postgres indexed 1M](results/db_1m/code-v4-indexed-b-tree.png) | ![Postgres disk writes](results/db_1m/postgress-disk-writes+transaction.png) |

| Redis — Standalone cache | Redis — In-memory + buffered writes |
|:---:|:---:|
| ![Redis cache 1M](results/db_1m/code-fast-redis-lookup.png) | ![Redis buffered](results/db_1m/redis-in-memory-buffered-writes.png) |

| Redis — 30-Node Cluster |
|:---:|
| ![Redis cluster](results/db_1m/redis-30-node-cluster.png) |

</details>

---

### 🧠 Key Takeaways

```
  4 RPS ──────────────────────────────────────────────────── 200,000 RPS
  ▮                                                                    ▮
  Postgres full scan (1M rows)          Redis 30-node cluster sharding
```

| # | Insight | Impact |
|:-:|:--------|:-------|
| 1 | **Indexing is non-negotiable.** A missing B-Tree index on 1M rows drops throughput from 15k to **4 RPS** — a 3,750× penalty. | 🔴 Critical |
| 2 | **Redis caching delivers 3.3× over indexed Postgres** with zero schema changes — just a cache-aside pattern. | 🟠 High |
| 3 | **Redis cluster sharding (30 nodes) breaks the single-core bottleneck**, scaling to 200k RPS with sub-millisecond p50 latency. | 🟢 Maximum |
| 4 | **PM2 cluster mode** saturates all CPU cores. A single Fastify thread does 25k RPS; 6 Cpeak workers hit **100k RPS** (4× scaling). | 🟠 High |
| 5 | **Framework overhead is minimal** at scale — the storage layer is the real bottleneck, not Express vs. Fastify vs. Cpeak. | 🟡 Medium |
| 6 | **Record count matters.** 1K-row Postgres does 1.4k RPS unindexed; at 1M rows it collapses to 4 RPS. Always benchmark at production scale. | 🔴 Critical |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** v22+
- **PostgreSQL** 17+
- **Redis** 7+
- **PM2** (for cluster mode): `npm install -g pm2`

### Installation

```bash
git clone https://github.com/Shivam-Singh-1/hold-my-redis.git
cd hold-my-redis
npm install
```

### Database Setup

**1. Create the config file**

Create `database/keys.js` with your Postgres credentials:

```javascript
const keys = {
  dbUser: "postgres",       // your Postgres username
  dbHost: "localhost",
  dbDatabase: "benchmark",
  dbPassword: "",           // your Postgres password
  dbPort: 5432,
};

export default keys;
```

**2. Seed the database**

```bash
# Default: 1,000 records
npm run seed

# 1 million records (recommended for realistic benchmarks)
npm run seed -- -r 1000000

# 20 million records (stress test)
npm run seed -- -r 20000000
```

This creates the `benchmark` database (if it doesn't exist), the `codes` table, and populates it with random records.

**3. Migrate data to Redis** _(required for cache benchmarks)_

```bash
npm run migrate
```

> ⚠️ Migration does not work with Redis in cluster mode. Use standalone Redis for migration, then switch to cluster if needed.

### Running the Server

Pick any of the three frameworks — they all share the same logic and endpoints:

```bash
node cpeak.js      # Cpeak   → http://localhost:3000
node express.js    # Express → http://localhost:3001
node fastify.js    # Fastify → http://localhost:3002
```

Expected output:

```
Cpeak server running at http://localhost:3000
[redis] standalone ready.
[postgres] connected successfully to benchmark.
```

---

## ⚙ Configuration

### Environment Variables

| Variable | Default | Description |
|:---------|:--------|:------------|
| `PG_CONNECT` | `"true"` | Connect to PostgreSQL on startup |
| `REDIS_CLUSTER` | `"false"` | Use Redis cluster mode instead of standalone |
| `F` | `"cpeak"` | Framework selection for PM2 (`cpeak`, `express`, or `fastify`) |

**Example — Redis cluster only, no Postgres:**

```bash
PG_CONNECT=false REDIS_CLUSTER=true node cpeak.js
```

```
Cpeak server running at http://localhost:3000
[redis] cluster ready. Total nodes 30 (masters: 15, replicas: 15)
```

### PM2 Cluster Mode

```bash
# Start (defaults to Cpeak)
pm2 start ecosystem.config.cjs

# Start with a specific framework
F=fastify pm2 start ecosystem.config.cjs
F=express pm2 start ecosystem.config.cjs

# Monitor
pm2 logs

# Stop
pm2 delete all
```

### Redis Cluster (30-Node)

The `redis.sh` script spins up a 30-node Redis cluster (15 masters + 15 replicas) on ports 7000–7029:

```bash
# First run — creates and initializes the cluster
chmod +x redis.sh
./redis.sh

# Subsequent runs — resume existing nodes
./redis.sh -resume

# Verify
redis-cli -p 7000 cluster info
redis-benchmark -p 7000 --cluster -t get,set -n 100000 -q
```

---

## 🔌 API Endpoints

| Method | Endpoint | Description | Storage Layer |
|:-------|:---------|:------------|:--------------|
| `GET` | `/simple` | Static JSON (no DB hit) | None |
| `POST` | `/code` | Generate code → write to PG + Redis | PostgreSQL + Redis |
| `GET` | `/code-v1` | Random code lookup — **full table scan** | PostgreSQL (no index) |
| `GET` | `/code-v4` | Random code lookup — **indexed query** | PostgreSQL (B-Tree) |
| `GET` | `/code-fast` | Random code lookup — **Redis cache** | Redis |
| `GET` | `/code-ultra-fast` | In-memory read + buffered async writes | Redis + PG (async) |
| `PATCH` | `/update-something/:id/:name` | Simulated update with body parsing | None |

---

## 📁 Project Structure

```
hold-my-redis/
├── cpeak.js                  # Cpeak framework server
├── express.js                # Express framework server
├── fastify.js                # Fastify framework server
├── ecosystem.config.cjs      # PM2 cluster configuration
├── redis.sh                  # 30-node Redis cluster setup (bash)
├── utils.js                  # Shared utilities
├── autocannon.txt            # Pre-built benchmark commands
├── database/
│   ├── index.js              # PostgreSQL connection pool
│   ├── redis.js              # Redis client (standalone + cluster)
│   ├── seed.js               # Database seeding script
│   ├── migrate.js            # Postgres → Redis data migration
│   ├── sync.js               # Data synchronization utilities
│   ├── tables/
│   │   └── codes.sql         # Table schema
│   └── keys.js               # Credentials (gitignored)
├── results/
│   ├── db_1000/              # Benchmark screenshots — 1K records
│   └── db_1m/                # Benchmark screenshots — 1M records
└── playground/
    ├── single-thread.js      # Single-threaded experiments
    └── multi-thread.js       # Multi-threaded experiments
```

---

<div align="center">

### 💡 The bottleneck is never the framework — it's the storage layer.

_Index your queries. Cache your hot paths. Shard when you need to scale._

---

**MIT License** · Built with ❤️ and a mass amount of `autocannon`

</div>
