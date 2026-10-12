# Redis: The "Network RAM" Trap

> **Languages:** [English](redis-critique-en.md) (primary) · [中文](redis-critique.md)

**Conclusion**: Rejected for use in new architectures.
**Status**: A "patch-style" solution to the fundamental design flaws of legacy languages (PHP/Java) and the limitations of hardware.

## Executive Summary

The classic "Redis + MySQL" stack is not an engineering ideal but a historical compromise. Redis solved specific problems (the statelessness of PHP/Java, microsecond-level latency) in an era when better alternatives (async runtimes, local KV, event sourcing) did not exist or were immature. Insisting on Redis in a modern high-performance environment (Rust/Go, event sourcing) introduces unnecessary latency and complexity.

*For the Nginx critique, see [`nginx-critique-en.md`](./nginx-critique-en.md).*

## 0. First Principles: The Memory Hierarchy and Data Sinking


### The Memory Hierarchy Pyramid
To understand why Redis is wrong, one must first look at the most fundamental physical constraint of computer architecture — the **Memory Hierarchy**. It is not a choice some designer made on a whim, but a direct response to the physical fact that no single storage technology can be "fast, large, and cheap" at the same time.

The pyramid runs from fast to slow: L1/L2/L3 cache → main memory → NVMe SSD → HDD. Each layer's latency and unit cost differ by an order of magnitude. The correct approach is to keep hot data resident in the upper layers and let cold data sink automatically to the lower layers.

**It is more fundamental than the von Neumann architecture**: von Neumann answers "how does a program run" (stored program, sequential instruction fetch); it is a **control model**, a design choice — the Harvard architecture, dataflow machines, and the GPU's SIMD all prove that "how execution is organized" has alternatives; but **the memory hierarchy does not disappear with the choice of control model**. The Harvard architecture and GPUs likewise have deep hierarchies (registers → shared memory → L2 → HBM/DRAM), because it is a direct response to a physical constraint, not an optional design. The hierarchy precedes and is independent of the von Neumann architecture, and has been so since the dawn of computing.

All discussion of caching, Redis, and data storage in this document is essentially **a scheduling problem over this pyramid** — which layer the data is placed in, and when it moves between layers. Every defect of Redis can be traced to its violation of this structure. See [5. Cache Layer Taxonomy](#5-cache-layer-taxonomy-rediss-position-and-why-its-wrong).


### Data Sinking: Compute-Near-Data



The memory hierarchy answers "which layer the data is placed in"; **compute sinking** answers "which process the compute is placed in" — these are two parallel physical constraints: the distance between data and compute determines command latency, and bringing compute near the data is the essence of efficiency. Redis violates the latter (data in Redis, compute in the application, ferried across the network) and it springs from the same source as violating the former.

#### The Golden Rule: Compute Near the Data

In computer science there is an eternal golden rule: **always move the computation to where the data is**, rather than hauling massive amounts of data over the wire to where the computation is. Redis's design violates this principle completely.

#### The "Patch-Style" Architecture

When combined with other legacy tools, **Redis + MySQL** creates a fragile "distributed monolith":
1. **The application queries Redis** (network hop) to check the session/cache.
2. **The application queries MySQL** (network hop) to fetch data.
3. **The application updates Redis** (network hop) to invalidate the cache.

**Result**: a single simple user request triggers **2-3 network hops**. Every hop is a potential failure point and a latency multiplier.

See [`mysql-critique-en.md`](./mysql-critique-en.md) for a detailed critique of MySQL's SQL anti-patterns and the sharding illusion.

#### The Disaster of Application-Side Joins

Redis supports only Key-Value and basic collections (Set/List/Hash). It **cannot handle the relationships between data (Relations)**.

If your business objects are even slightly complex (for example: fetch a user, his orders, the items within the orders, and the items' current stock), Redis forces you into **N+1 network queries** in code:

1. You first have to look up the user's Key and get back a pile of order IDs.
2. Write a loop in code to fetch those orders from Redis one by one, or in a batch.
3. Once you have the orders, parse out the item IDs in code, then issue network requests to fetch the items and their stock.

Even if you use Pipeline to batch the requests, your application server is still doing inefficient work — repeatedly deserializing responses in various formats in memory, then reassembling them into business objects with the CPU. **You have implemented a "pseudo query engine" lacking an optimizer on the application side.**

In the SQL world, by contrast, an elegant declarative JOIN or `JSON_AGG` lets the database engine orchestrate the relationships directly in **shared memory hugging bare metal** at top speed, spitting out through the wire just one clean final JSON result.

#### The Database as Application Server: The Dimensionality Reduction of Compute Sinking

**PostgreSQL's embedded high-level languages**: Postgres long ago ceased to be a rigid storage disk; it is an **all-purpose compute platform**. By allowing an unrestricted Python runtime (`plpython3u`), the V8 engine (`plv8`), or native PL/pgSQL to be embedded inside the database, developers can execute extremely complex business logic, data cleaning, regex matching, and even AI inference directly inside the database. **Data and code interact within the same memory space, driving network latency and serialization overhead down to an absolute 0.**

**SurrealDB's multi-model query (SurrealQL)**: SurrealDB natively fuses the graph database (Graph), document database (Document), and relational database (Relational). Its query language supports traversing deeply nested relationships within a single declarative query directly through graph semantics like `->like->product`. **You don't need multiple queries at all, and you need some external cache even less.** → See [SurrealDB: The Architectural Philosophy of a Unified Data Layer](./unified-data-layer-en.md)

#### The Collective Ebb of the 2010s NoSQL Fever

The 2010s "NoSQL/Polyglot Persistence" boom has receded, and the technical community is re-examining the architectural choices made back then.

The vision of the day — "use five databases in one project: Redis for cache, Neo4j for graph, Mongo for documents, Elastic for search, MySQL for relations" — evolved in practice into:
- Distributed transactions failing frequently
- Data synchronization latency that is hard to eliminate
- Operational complexity growing exponentially
- Infrastructure costs climbing continuously

By firmly embracing PostgreSQL's formidable JSONB aggregation capabilities/embedded compute ecosystem, or supplementing SurrealDB's multi-model capabilities in critical data scenarios, you achieve an "Occam's razor" architectural minimalism:

- No multi-round network latency
- No serialization churn
- No Lua landmines
- No multi-database synchronization

**Single request, single return, straight to the essence.**

### The Database's Two Core Capabilities: Storage-Architecture Utilization and Query Optimization

A database system has only two core capabilities: **efficient utilization of the existing storage architecture**, and **query optimization**.

**Storage-architecture utilization** = the scheduling of the memory hierarchy (which layer data is placed in, when it sinks) compounded with **access-pattern adaptation** — matching the read/write shape to the physical characteristics of the medium.

Here let us first puncture an assumption taken as common sense: **"memory inevitably crushes disk."** Most people's default belief is that memory and disk differ by several orders of magnitude (casually "a thousand times faster"), but this belief is the product of "not digging deeper," not a fact. The value of disk was never speed — DDIA's own words are that disk is "awkward to deal with"; the reason it is tolerated is precisely the two points of **durability** and **low unit cost**; and as for the concrete comparison of an in-memory store (Redis) against a disk-based store (Kafka/LSM), the real gap is often only **around one order of magnitude** (perhaps tenfold, perhaps less), not three orders of magnitude.

The gap is exaggerated because people assume by default that "not reading from disk = fast," while overlooking that any disk engine with a buffer pool keeps the data resident in the OS page cache / buffer pool whenever the working set fits, and simply does not read from disk at all — "avoiding disk reads" is not an advantage unique to in-memory databases. So what real advantage does an in-memory database have left? DDIA puts it plainly — the performance advantage of in-memory databases is not due to the fact that they don't need to read from disk... Rather, they can be faster because they can avoid the overheads of encoding in-memory data structures in a form that can be written to disk. That is, there are only two real advantages: **avoiding the encode/decode overhead between the in-memory representation and the on-disk representation**; and **being able to provide complex data structures that on-disk indexes find hard to implement (priority queues, sets)** — neither of which is "memory being fast."

Crossing the network, this advantage is dominated by millisecond-level RTT first (§1). Kafka's read/write pattern offers the same insight at this layer: sequential append log + page cache + zero-copy lets cheap media serve at memory-grade throughput without hauling data into expensive memory.

**Query optimization** = Parser + Optimizer: compiling declarative intent into the optimal physical execution path (stacking these two layers on top of KV yields a database, see [KV Storage Engine](kv-storage-engine-en.md)).

Redis possesses neither core capability:

- **Storage-architecture utilization**: it rejects the memory hierarchy, pressing data of all temperatures onto the same layer of expensive media (§5). And the two real advantages DDIA attributes to in-memory databases are both precisely falsified: encode/decode overhead — an embedded KV reads directly in-process, with zero serialization, saving even more thoroughly (§8); complex data structures — under composite key encoding the moat is zero (§7). All-in-memory thus leaves only cost, having bought no exclusive speed; it also locks naturally sequential data (queues, logs) into the most expensive layer (§7).
- **Query optimization**: it has no optimizer. "Data-structure commands" are fixed primitives rather than a query capability — under composite key encoding the moat is zero (§7); capabilities that truly require an optimizer to orchestrate, such as JOIN / aggregation / subqueries, it simply does not have.

The result of both cores being empty is that Redis can only degenerate into **"a remote service that does a few operations in memory"** — which is exactly the structural root of what §1.5 "Beyond the Process as Well" argues, namely that "a self-built single-responsibility service is superior to Redis": since it does none of the things a database should (hierarchy scheduling, access-pattern design, query optimization), building one's own "engine + network access layer" to cover its entire function, and being comprehensively superior on multicore, persistence, and protocol overhead, is necessity rather than coincidence.

## 1. Network Latency: The Achilles' Heel

**If the data is remote, "a fast in-memory database" is a semantic contradiction.**

Memory access latency is about 100 nanoseconds. Network RTT (a typical same-datacenter value of 0.5-2ms) is 5,000-20,000 times slower than memory access. Even if Redis completes a query in 0.01ms, the network round-trip (1ms) dominates the total latency. Using Redis to "speed up" data access is usually slower than querying a well-indexed local SQLite/Redb file or using a local in-process cache.

**The relationship between RTT and compute sinking**: this chapter focuses on the **symptom** that "remote access itself is expensive" (RTT dominates latency); one of the **means** of reducing round-trips is compute sinking — moving the logic to the data layer to execute. As the same means, why it fails on Redis (Lua) (logic falls back to the application layer, RTT cannot be reduced), see the RTT dimension table in [Query Language Design](query-language-design-en.md) — comparing the different fates of the same means under KV (driven to zero), SurrealQL (~1), and Redis Lua (failure).

### RTT Arithmetic: The Overlooked Order of Magnitude

This is the most common blind spot in architecture design — developers focus on Redis's internal microsecond-level latency while ignoring the limit imposed by the speed of light in the wire.

```
Redis internal execution time:      ~0.1 ms (pure in-memory lookup)
PostgreSQL shared buffer execution: ~0.5-1.0 ms (shared memory)
Real RTT on the datacenter LAN:     ~1.0-3.0 ms (network physical overhead, TCP handshake, switch routing)

Real total latency = network RTT + database internal processing time
```

**Head-to-head comparison**:
- **Redis total latency**: 2.0 ms (RTT) + 0.1 ms = **2.1 ms**
- **PostgreSQL total latency**: 2.0 ms (RTT) + 0.5 ms = **2.5 ms**

The absolute gap between the two is only 0.4 ms. In the vast majority of commercial web service and API backend scenarios, a 0.4 ms gap is imperceptible to end users. Introducing an independent middleware into the architecture for this — one that is high-risk, prone to loss, expensive in memory, and must also guard against cache breakdown and stampede — lacks engineering justification.

This is not performance optimization but cargo cult architecture imitation.

**The number of round-trips per operation** — the other side of RTT is not just how high or low the latency is, but how many round-trips it takes. For the same logic, different storage layers produce wildly different network round-trip counts:

| Storage | RTT | Reason |
|:--|:--|:--|
| KV (embedded) | **0** | Direct in-process connection, no network |
| SurrealDB | **1** | Compute pushdown, a single request completes graph traversal + logic, one round-trip |
| PostgreSQL | **≈1** | Multi-table/complex operations also approach 1 — CTE/single-transaction batch modification of multiple tables, plpython3u carrying complex logic |
| Redis | **N (Lua can compress to ~1)** | Multi-command N round-trips; Lua (EVAL) can batch, but far less capable than plpython3u, complex logic still constrained |

Redis's pain point is not just that each round-trip is slow (latency), but that **the same logic often has to round-trip N times** — the N+1 associations of §0's data sinking, read-modify-write, and multi-command orchestration all push the round-trip count up. Lua scripts (EVAL) can indeed pack them into a single round-trip, but **they are far less usable than plpython3u**: PG's plpython3u is full Python, able to call external libraries and write complex logic, making multi-table/complex operations naturally approach 1 RTT (plpython3u is only used when external libraries or complex operations are involved; ordinary queries go through native SQL and likewise take 1 round-trip); Redis's Lua is functionally limited, has no external libraries, and has awkward syntax — carrying complex logic with it is more painful than not using it. In a layered architecture KV is 0 and PG is a stable ≈1; Redis's "fast memory" loses on both round-trip count and built-in logic capability.

### Beyond the Process as Well: Even Forced Out of the Process, Redis Is Not the Optimal Solution

The "prefer in-process memory" argued in the rest of this chapter is an **independent** principle. But more strictly — **even if this principle is discarded, or the scenario forces the state to live outside the process (a shared service)**, Redis is still not the optimal solution for that scenario, only the "good enough" default. The following argument stands entirely on the premise of "outside the process" and does not depend on the in-process principle holding:

- **RTT is still absurd**: in-process saves the network, but conversely — since the state has been decided to live outside the process, it should be designed to be **precisely accounted for**. Redis's RTT is **absurd** (the ping/pong protocol round-trips, multi-command serial N round-trips, already described in §1; and the linear protocol parsing it retains for generality further amplifies the overhead). In an equally absurd out-of-process scenario, a self-built, single-purpose local service also eliminates the "absurd margin," and is naturally superior.
- **Single-machine multicore: what Redis deliberately cedes.** Redis's consistency relies on a single process (lock-free), but "single-process serial" is not its patent — a self-built single-responsibility service is likewise single-threaded, and also does away with RESP parsing and the general-purpose command set. More thoroughly: a self-built one can freely use multiple processes/threads to attain true parallelism on modern multicore; Redis, even after 7.x introduced I/O threads, still keeps its core execution in a single-threaded closed loop. Multiple processes incur consistency control, but that is precisely "the performance bought in exchange" (§2 multicore throughput).
- **Compact serialization: RESP's text overhead is pure waste** (detailed in §8). A single-purpose service can use Bincode/CBOR/Postcard-style binary, eliminating all of Redis's byte-by-byte `\r\n` scanning and ASCII-to-integer conversion.

**Conclusion**: when neither path works, Redis is that "good enough but mediocre" middleware. **It is not that "out-of-process justifies Redis," but that "out-of-process + pursuing the extreme argues all the more for abandoning Redis and switching to a self-built single-responsibility service."** Composite structure: this section argues "out-of-process Redis is not optimal," §6 argues "multi-machine Redis has no strong consistency" — the two combined leave Redis with no foothold in any form (in-process/out-of-process/multi-machine).

## 2. Design Philosophy: The Optimal Solution of 2009

### Historical Background

Redis was born in 2009 — an era dominated by **mechanical hard disks (HDD)** and **single-core CPUs**. Its single-threaded, all-in-memory architecture was the optimal solution for the hardware reality of the time: when disk I/O takes milliseconds, putting everything in RAM and processing it sequentially is the only way to achieve microsecond latency.

Redis's explosive popularity stems from the fact that **PHP-FPM is stateless**. Every request starts from scratch, forcing the application to offload *everything* (sessions, counters, caches) to external storage. In languages with persistent runtimes (Rust, Go, Node.js, Elixir), the application process can hold state efficiently in local memory (HashMap, B-Tree), with no network overhead. Redis is often the crutch of "Share Nothing" architectures — sharing *too much* over the network.

This origin also determined the performance-cognition baseline of Redis's user base: it was long bound to the PHP ecosystem, and this group's "memory vs disk" judgments mostly stay at the slogan level of "memory is a thousand times faster," without digging into buffer pools, access patterns, and encode/decode overhead (§0). Redis's performance myth circulates most widely in the Chinese-language technical community, and the soil is exactly here.

### The Failure of Modern Hardware

We are no longer in 2009. Modern servers have:
- **Multicore CPUs** (16-128 cores): Redis's single-threaded event loop squanders more than 95% of the available compute
- **NVMe SSD** (sub-millisecond I/O): the "RAM vs disk" gap has shrunk from 1000x to 10x
- **Cloud-native architecture**: network RTT (0.5-2ms same-datacenter) dominates Redis's microsecond-level processing

Redis's single-threaded event loop means:
- **Throughput ceiling**: ~100,000 QPS per instance (limited by single-core performance)
- **Cannot exploit multicore**: a 128-core server wastes 99% of its compute
- **Scaling requires sharding**: Redis Cluster is client-side sharding, with no strong consistency, manual slot management, and painful resharding

This is not a "fixable limitation" — it is a **fundamental architectural choice**, reasonable in 2009 but untenable in 2026. Modern systems (Rust/Tokio, Go, Elixir) achieve **millions of QPS** by exploiting multicore parallelism. Redis is trapped in the single-core era.

### Multi-Instance Redis: The Complexity Trap

**Workaround**: run 64 Redis instances on a 128-core machine (2 cores each) to exploit multicore.

**Problem**: this introduces enormous complexity without solving the fundamental issue:

**1. Client-side sharding complexity**
```python
# PostgreSQL: simple and direct
db.execute("SELECT * FROM users WHERE id IN (1, 1001, 2001)")

# Multi-instance Redis: the client must implement routing
def get_users(user_ids):
    results = {}
    for uid in user_ids:
        instance = hash(uid) % 64  # route to the correct instance
        results[uid] = get_redis_client(instance).get(f"user:{uid}")
    return results
# 64 network requests instead of 1 query
```

**2. Cross-instance operations are impossible**
- `MGET user:1 user:1001 user:2001` → `CROSSSLOT` error (the keys are on different instances)
- Cross-instance transactions require 2PC or the Saga pattern (application-layer compensation)
- PostgreSQL: a single instance, ACID transactions, no sharding logic

**3. Memory waste persists**
- A 100GB dataset spread across 64 instances = 1.56GB per instance
- Each instance still stores all data in memory (no tiered caching)
- Cold data (90% of the dataset) occupies expensive RAM but is rarely accessed
- PostgreSQL: 20GB buffer pool (hot data) + 100GB SSD (all data) = 80% memory saved

**4. Operational complexity explodes**
- 64 configuration files, 64 monitoring endpoints, 64 backup strategies
- Resharding requires data migration (service degraded during migration)
- PostgreSQL: 1 instance, 1 configuration, 1 monitoring endpoint

**Performance comparison** (128-core machine, 100GB dataset):

| Metric | 64 Redis instances | 1 PostgreSQL instance |
|------|-----------------|---------------------|
| CPU utilization | 64/128 (50%) | 4-8/128 (3-6%) |
| Memory usage | 100GB (all data) | 20GB (hot data cache) |
| Throughput | 6.4M QPS | 3-6M QPS |
| Cross-instance operations | ❌ Not supported | ✅ Supported |
| Transactions | ❌ Single instance only | ✅ ACID |
| Operational complexity | 🔴 Extreme (64 instances) | 🟢 Low (1 instance) |
| Scaling cost | 🔴 High (resharding + migration) | 🟢 Low (add disks) |

### PostgreSQL's Multicore Throughput

Modern PostgreSQL can fully and significantly surpass single-threaded Redis in throughput.

**PostgreSQL's modern heavy weapons**:
- **Multi-process/multi-thread concurrent execution**: Parallel Query capability, scaling linearly across cores
- **Highly optimized shared-memory buffers**: mapped directly onto operating-system shared memory (Shared Buffers)
- **Unlogged Tables**: skip the WAL log in exchange for maximum write throughput (when durability is not critical)
- **Heavy JSONB caching**: cache JSON documents directly in shared memory, queried with GIN indexes

**Benchmark data** (128-core machine, all-NVMe array, 200-500 concurrent test threads):

| Metric | Ordinary table (Logged) | UNLOGGED table | Redis single-threaded |
|------|-----------------|-------------|-------------|
| **Pure write TPS** | 50-150K (limited by fsync/WAL) | 300-600K+ (limited by LWLock contention) | ~100K (single-core bottleneck) |
| **Read-only TPS (all-memory hit)** | 1.5-2.5M | 1.5-2.5M (read performance is independent of WAL) | ~100K (single-core bottleneck) |
| **Bulk import** | 300-500K rows/sec | 600-1000K+ rows/sec | Not applicable |

Key finding: **in read-only scenarios, even without UNLOGGED, PG's throughput is 15-25 times that of Redis** (1.5-2.5M vs ~100K). The reason is simple — PG can use all 128 cores, while Redis is locked to a single core. Redis's "memory speed advantage" is meaningless in the face of multicore parallelism.

**The irony of the architecture**: Redis was designed to avoid the "overhead" of disk I/O and locks. But modern PostgreSQL's shared buffer pool is essentially a highly optimized, multicore-safe, ACID-compliant in-memory cache — plus an extra benefit: cold data automatically spills to cheap SSD instead of occupying expensive RAM 24/7.

### The Scalability Chasm

When you need to scale beyond a single Redis instance, cost and complexity explode:
- **Cache**: stream computing (RisingWave/Flink) + CDN is **900 times cheaper**, and can scale horizontally without limit. Redis Cluster requires manual slot management, cross-slot operations fail, and resharding is agonizing.
- **Sessions**: this is a **distributed coordination problem** disguised as a caching problem. The logic is simple (store/retrieve user state), but Redis Cluster introduces single-point-of-failure risk. Modern distributed KV stores (etcd, Consul) with consensus leases provide correct coordination guarantees; aura's stance is metadata single-write with no consensus (user data bound to its owning node).
- **Leaderboards**: at large scale (100M+ entries), specialized algorithms (Count-Min Sketch, T-Digest) or stream processing engines handle aggregation more efficiently. At medium scale (1M entries), PostgreSQL with appropriate indexes can achieve **1 million concurrent queries** — Redis's throughput advantage disappears, because it is bottlenecked by single-core performance (~100K QPS).

**Co-location probability**: in a cluster of N machines, the probability that the application and Redis are on the same machine is 1/N. Even with co-location, the expected latency is still ~1ms.

DHH's insight: modern hardware is so good that we should **re-examine every architectural assumption from before the 2010s**. Things that were "impossible" without a cache layer in the past are now easily achievable through direct database queries. Redis's existence is a **historical constraint**, from an era when hardware was 100 times slower.

## 3. Cost Analysis: Memory vs Disk

### Hardware Resource Comparison

**Scenario**: 100GB dataset, 100,000 QPS target

| Resource | Redis | PostgreSQL | Cost difference |
|------|-------|------------|---------|
| **CPU** | 2-4 cores (single-threaded) | 16-32 cores (multicore) | PG 4-8x more |
| **Memory** | **100GB** (all data) | **10-20GB** (hot data cache) | Redis 5-10x more |
| **Disk** | 0GB (pure memory) | 100GB SSD (all data) | PG more (but SSD is cheap) |

**Memory vs disk pricing** (AWS reference prices, 2026):
- Memory: $0.023/GB/hour = **$16.56/GB/month**
- SSD: $0.08/GB/month
- **Memory is two orders of magnitude more expensive than SSD** (the exact multiple shifts with cloud-vendor pricing, but the order-of-magnitude gap is constant)

### Memory Waste

```
Access pattern of a 100GB dataset:
- 10% hot data (1000 accesses/day) = 10GB
- 30% warm data (10 accesses/day) = 30GB
- 60% cold data (0-1 accesses/day) = 60GB

Redis: must store all 100GB in memory
- Memory cost: 100GB × $16.56/GB/month = $1,656/month
- Waste: 60GB of cold data occupies 60% of memory but is rarely accessed

PostgreSQL: tiered caching
- Buffer pool (memory): 20GB (hot data)
- Disk (SSD): 100GB (all data)
- Memory cost: 20GB × $16.56/GB/month = $331/month
- Disk cost: 100GB × $0.08/GB/month = $8/month
- Total: $339/month

Savings: 60GB of cold data placed on disk ($0.08/GB/month vs $16.56/GB/month)
```

### Total Cost of Ownership

(3 years, 100GB dataset, 100,000 QPS):

| Cost item | Redis | PostgreSQL | Difference |
|--------|-------|------------|------|
| **Hardware** | $166,320 | $21,024 | Redis 8x more |
| **Operations** | $54,000 (3 person-years) | $18,000 (1 person-year) | Redis 3x more |
| **Scaling** | $138,600 (growth to 1TB) | $21,600 (growth to 1TB) | Redis 6x more |
| **Total** | **$358,920** | **$60,624** | **Redis 6x more** |

**Scaling cost comparison** (100GB → 1TB):
- **Redis**: cost rises 10x (memory must be added for all data)
- **PostgreSQL**: cost rises 2x (mainly adding disk, not memory)

Redis's "low cost" is a myth. In real-world medium-scale scenarios, PostgreSQL's TCO is **1/6 that of Redis**. Redis's memory waste and CPU waste make it the **most expensive caching solution**.

## 4. The Flux Architecture: From Passive Pull to Active Push

The legacy passive cache (Redis, request-driven, TTL invalidation) is a placeholder substitute for a local cache. Data characteristics determine the caching strategy: **static data** (product listings, category pages, blog posts) has a freshness requirement of hours to days, suited to CDN edge caching — a CDN is about 900 times cheaper than Redis, puts zero load on the origin, and delivers sub-10ms latency globally; **dynamic data** (user profiles, shopping carts, real-time inventory) has a second-level freshness requirement and should go through streaming precomputation + active push, rather than pull-from-origin. Cache invalidation is solved with an **index version-number interface**: `if (remote_version > local_version) { download_from_cdn(); }`, lifting version governance from the storage layer to the control plane (the same source as the Lakehouse Catalog; see [Lakehouse Research](lakehouse-research-en.md)).

**The complete architectural argument for this CDN / stream computing / index version-number approach has been written up separately, see [Flux Architecture](flux-architecture-en.md)** — the three-tier push strategy (static layer CDN, dynamic layer WS/WebTransport, hybrid strategy), stream computing and message queues moving from auxiliary to central, the Aura in-process event-processing alternative path, and the architectural comparison with the passive mode. This document retains only the conclusion: the Flux architecture replaces Redis at the pattern level, a structural replacement rather than a patch.

## 5. Cache Layer Taxonomy: Redis's Position (and Why It's Wrong)

Modern caching is not a binary choice between "Redis" and "no cache." There is a **spectrum of caching strategies**, each with different trade-offs. Redis occupies an increasingly obsolete and awkward middle ground.

The essence of caching is not "using memory makes it fast," but the **tiered scheduling of the memory pyramid** (see §0). Core conclusion: Redis's all-in-memory strategy **rejects this tiering** — pressing data of all temperatures onto the same layer of expensive media. This is not "fast caching," it is ignoring the first principles of the memory pyramid. If "memory = fast" were the truth, the optimal solution would be to cram all data into L1 cache or weld it directly onto the CPU. Redis did not go to that extreme; it merely went to the other extreme: replacing the entire pyramid with a single layer of memory.

### L0: In-Process Memory (Fastest, Smallest Scope)

**Examples**: Caffeine (Java), cachepot (Rust), `HashMap` with TTL, LRU caches.

- **Latency**: ~10-100 nanoseconds (direct memory access, no serialization).
- **Scope**: a single application instance. Not shared across processes/nodes.
- **Invalidation**: simple (process-local state, no distributed consensus).
- **Use cases**: hot data suited to RAM, per-request caching, session data in stateful applications.

**Why Redis loses here**: even if Redis executes in 0.01ms, the network RTT (20-500µs) dominates. In-process is **200-50,000 times faster**. For languages with persistent runtimes (Rust, Go, Elixir), there is no reason to externalize the cache to a network service.

### L1: Database Transparent Cache (Automatic, Query-Level)

**Examples**: **ReadySet** (based on the Noria paper), **ProxySQL query cache** (deprecated), **MySQL Query Cache** (removed in 8.0).

- **Mechanism**: sits between the application and the database. Intercepts SQL queries, returns cached results, and auto-invalidates when data changes (via binlog replication or trigger-based tracking).
- **Latency**: ~100-500 microseconds (still a network hop, but fewer than Redis + DB).
- **Scope**: database query results. Transparent to the application (no code changes).
- **Invalidation**: automatic (data-flow driven, no manual TTL or cache-aside logic).

**ReadySet** is the most sophisticated: it maintains **incremental materialized views** of SQL queries. When base tables change, ReadySet incrementally updates the cache (not a full recomputation). This is architecturally superior to Redis cache-aside:
- **No cache breakdown**: ReadySet gracefully handles concurrent cache misses (coalescing).
- **No stale data**: invalidation is automatic and immediate (binlog-driven).
- **No application changes**: a plug-and-play MySQL/PostgreSQL proxy.

**Why Redis loses here**: Redis requires manual cache management (TTL, cache-aside pattern, invalidation logic). This is error-prone (cache breakdown, stale data, thundering herd). ReadySet automates everything Redis makes you do by hand.

**Historical failures**: MySQL Query Cache (removed in 8.0) failed because it used a global mutex (a single lock for the entire cache). ProxySQL query cache was deprecated because it could not handle complex queries (joins, subqueries) and had no automatic invalidation. ReadySet solves these problems with a **dataflow graph** (the Noria paper) — each query is an operator DAG, and updates flow incrementally.

### L2: CDN Edge Cache (Cheapest, Largest Scale)

**Examples**: CloudFront, Cloudflare, Fastly, Akamai.

- **Latency**: ~10-50 milliseconds (geographically distributed, but with network hops to the edge).
- **Scope**: static or semi-static data (HTML, JSON, images, API responses).
- **Invalidation**: TTL-based (eventual consistency) or manual purge API.
- **Cost**: ~$0.08/GB/month (1000 times cheaper per GB than Redis).

**Why Redis loses here**: for static data, a CDN is **900 times cheaper** and **faster** (edge nodes are closer to users than a Redis cluster). Redis forces every request to hit your origin + a Redis cluster, paying for both.

### L3: Distributed KV / Coordination (Specialized, Not a Cache)

**Examples**: etcd, Consul, Raft-based stores.

- **Purpose**: distributed coordination, service discovery, configuration management.
- **Not a cache**: these are the **source of truth** for distributed state, not a cache of other data.
- **Why Redis loses here**: Redis Cluster is not strongly consistent (no Raft/Paxos). Using Redis for distributed locks or coordination is a single point of failure. etcd/Consul provide correct consensus guarantees.

### The Redis Trap: Stuck Between L0 and L2

| Tier | Latency | Cost | Automation | Redis comparison |
|------|------|------|--------|-----------|
| L0 (in-process) | 10-100 ns | Free (RAM you already have) | Manual (but simple) | Redis is **200-50,000 times slower** |
| L1 (ReadySet) | 100-500 µs | Moderate (proxy overhead) | **Automatic** (binlog-driven) | Redis requires **manual cache-aside** |
| L2 (CDN) | 10-50 ms | $0.08/GB/month | TTL-based | Redis is **900 times more expensive** |
| L3 (etcd/Consul) | 1-10 ms | Moderate | N/A (source of truth) | Redis has **no consensus** |

**Verdict**: Redis is slower than L0, less automated than L1, more expensive than L2, and less reliable than L3. It is a **jack of all trades, master of none** — the same class of anti-pattern as MySQL's pluggable storage engines or Nginx's Lua scripting.

**Scenarios where Redis *might* make sense** (rare):
- **Legacy migration**: you have a PHP/Java application with thousands of cache-aside calls. Rewriting it to use ReadySet or an in-process cache is a months-long project. Redis is the "least bad" incremental step.
- **Polyglot microservices**: multiple services (Python, Go, Node.js) need a shared cache. In-process caches cannot work across language boundaries. But: ReadySet is language-agnostic (an SQL proxy), and CDNs are universal.
- **Pub/Sub toy**: you need simple publish/subscribe and don't want to deploy NATS/Kafka. But: Redis pub/sub has no persistence, no consumer groups, no backpressure. NATS is lighter and better.

**For new projects**: skip Redis entirely. Use:
- **L0** (in-process) for hot data.
- **L1** (ReadySet) for database query caching (if you need it).
- **L2** (CDN) for static/semi-static data.
- **L3** (etcd/Consul) for distributed coordination.
- **SurrealDB/PostgreSQL** for complex data structures (Redis's sorted sets, GEO, HLL all have better alternatives).

## 6. Redis Is Not Distributed

A common defense of Redis is: "even if it is not ideal as a cache, it still has value as a distributed coordination/shared-state layer." This defense collapses under scrutiny.

### The Core Problem: Single Point of Failure

Redis is essentially a **single-node process**. Redis Cluster is not true distributed computing — it is client-side sharding, with no strong-consistency guarantee during partitions. When you use Redis as a "distributed coordination layer," Redis itself becomes the **weakest link** in your distributed system.

**Client-side caching (Redis 7.0)**: the ultimate irony. Redis introduced client-side caching because it finally admitted: *remote queries are too slow, and you need local data regardless.* The result: you end up managing *two* caches (client + server), handling invalidation logic, adding complexity for a marginal gain.

### Distributed Locks

**Why Redis happens to be "popular" in this scenario**: a distributed lock is the most typical coordination-class primitive that can be implemented with a **monolithic component** and "locked" with a very low barrier to entry — it does not require true distribution or strong consistency, and a single-point process is enough to reach it. Precisely because Redis can reach it and the barrier is exceedingly low, it became the first-choice vehicle for distributed locks — but this is exactly what exposes the problem: **a low barrier to entry does not mean it is correct.**

The complete argument for this anti-pattern (the file-lock boundary, single replica vs true distribution, watermark substitution, distributed transactions needing no lock, the amplification effect of locks) is in the separate document **[Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md)**. Core conclusion: **distributed locks are an anti-pattern in the vast majority of scenarios**.

- **Claim**: "Redis is very well suited to distributed locks (Redlock)."
- **Reality**: Redis is single-node, so the lock service is itself a single point of failure. If Redis goes down, all locks are lost.
- **Alternative**: a simple single-threaded Rust service using `Arc<Mutex<HashMap>>` — data in memory, no async, no concurrency — achieves the same effect. A lock service is simple logic; Redis is over-engineering in this scenario.

#### The Redlock Debate: Kleppmann vs Antirez (2016)

Redlock is a distributed lock algorithm proposed by Redis's author Antirez — achieving mutual exclusion based on multiple Redis nodes. At the time, many regarded it as the standard solution for distributed locks in production.

Martin Kleppmann (author of *Designing Data-Intensive Applications*) published **"How to do distributed locking"** raising a systematic challenge:

**Core argument: distributed locks have two purposes — efficiency and correctness.**

| Purpose | Meaning | Correct solution |
|------|------|---------|
| **Efficiency** | Avoid duplicated work; a lost lock merely means doing it once more | Single-node Redis suffices, Redlock is unnecessary |
| **Correctness** | Protect shared resources from concurrent corruption; a lost lock causes data corruption | Redlock is unsafe, a consensus algorithm is needed |

**The attack scenarios Kleppmann raises:**

1. **The GC pause problem**: client A acquires the lock and then a GC pause occurs; the lock expires during the pause and client B acquires the same lock — two clients hold the lock simultaneously, and mutual exclusion fails
2. **Clock jumps**: Redlock relies on clock synchronization between nodes; if a node's clock jumps (NTP correction, etc.), the safety of the lock collapses
3. **Network delay**: the response to a client's lock request is delayed for a long time in transit, but the lock has already expired — the client thinks it acquired the lock, but it has in fact lapsed

**Kleppmann's alternative: the fencing token**

Each time a lock is acquired, the lock service returns a monotonically increasing token. The client carries the token when accessing the resource, and the resource side rejects stale tokens. This moves safety from the lock service to the resource side, without depending on assumptions about clocks and network timing.

**Antirez's response: "Is Redlock safe?"**

- Concedes that the GC pause is indeed a problem, but argues that the probability of it occurring in practice is extremely low
- Rebuts the clock-jump scenario
- Argues that Redlock is practically pragmatic in engineering

**The subsequent consensus:**

This debate had no clear winner, but the community formed a clear conclusion:

> **Redlock must not be used in scenarios that demand correctness.**

- Need strict mutual exclusion (to protect shared resources) → **ZooKeeper / etcd** (based on the Raft/Zab consensus algorithms)
- Optimizing only for efficiency → single-node Redis suffices, Redlock is unnecessary
- Redlock sits in an awkward middle ground — **heavier than single-node Redis, yet not safe enough to serve as a correctness guarantee**

**Essence**: this debate reveals not just Redlock's flaws, but the typical pattern of Redis as a "placeholder" — many people use Redlock without actually thinking through whether they need efficiency or correctness. Just like adding Redis blindly in an architecture document, behind it is a problem you have not thought through.

### Redis Cluster: Cluster Mode Still Has No Coordination Capability

Redis's cluster mode is its only "multi-machine" form, but taken apart, it **possesses no real distributed coordination capability whatsoever** — which is precisely the proof that single-node defects cannot be made up for by "adding machines."

| Capability (that a distributed system should have) | The reality of Redis Cluster |
|:--|:--|
| **Online migration during rebalancing** | **None**. reshard / reslot is **manual + blocking**, sacrificing availability during migration; it is not "smooth migration during coordination" |
| **Strong consistency of failover** | **None**. Master-replica replication is **asynchronous**; a failover switch **loses the most recent writes** (data acknowledged but not yet replicated), and loss is never prevented |
| **Cross-node consistency** | **None**. **No distributed transactions**; multi-key atomicity can only be forced into the same slot via hash tags, with no consistency guarantee across slots |
| **Cross-node strong consistency** | **None**. Redis Cluster is at most "sharding + best-effort availability," not a strongly consistent distributed KV |

**Key corollary**: what truly possesses "rebalancing migration + strongly consistent failover + cross-node transactions" is a distributed KV at the tier of TiKV (Raft) / FDB (global ACID) — **which is not Redis's positioning**. Therefore:

- **Single-machine scenarios**: a self-built in-process/out-of-process service >> Redis (§1, §8).
- **Multi-machine scenarios**: the shallow sharding Redis provides (static + asynchronous loss), a self-built system can also achieve, and achieve unambiguously; and the strong consistency Redis cannot provide, **Redis does not have either** — one should not use "distributed coordination is hard" to retort against self-built solutions, because the reference point itself lacks these capabilities.

This rebuttal applies equally to the defense of "using Redis as a distributed coordination layer": the coordination layer is precisely where consistency is most needed, yet in cluster mode Redis does not even provide the most basic strongly consistent failover.

## 7. The False Moat of Redis's Data-Structure API

### API Level Classification

Data-operation APIs come in three levels. Redis stops at the middle level — it has more structured primitives than bare KV, but less relational expressiveness than a true query language. This middle layer is **extremely thin** — it introduces no new abstraction mechanism whatsoever; it merely renames KV's existing capabilities (key space, atomic batches) into a handful of commands:

| Level | Representative | Core capability | Missing |
|:---|:---|:---|:---|
| **Low** | Bare KV (get/put/scan) | Byte read/write, no semantics | All higher-level abstractions |
| **Middle** | Redis (Hash/ZSET/Set) | Fixed data structures, no relational queries | JOIN, declarative logic, transaction rollback |
| **High** | SQL/SurrealQL (JOIN/aggregation/subqueries) | Declarative relational queries, Turing-complete | None (complete) |

Redis's data structures are often regarded as a core competency — "Redis's API is more advanced than bare KV." Under the scrutiny of Composite Key Encoding, this argument does not hold. How thin this "thin layer" really is can be shown with two primitives:

**Hash is not a new structure; it is a sub-level key space.** `hash:user:101:name` from `HSET user:101 name "Alice"` is essentially concatenating two key names into one — KV's composite key encoding is its default usage anyway. The "Hash" primitive introduces no capability KV cannot provide; it merely replaces the key concatenation a developer already has to do when writing KV with a command name. It does not even pretend to be a new level of abstraction; at a glance it is still a key-space operation.

**What ZSET makes atomic is the entire "read → compute → write" sequence in KV.** To update a leaderboard score, in KV you read the old score, compute the new score at the application layer, then write it back — the intermediate computation happens at the application layer; Redis does all of this atomically with the single command `ZINCRBY`, absorbing the computation into the server side. So-called atomicity is essentially merging the "read-modify-write" sequence the application layer would have to do into the server, locking the concurrency window with a single command.

But this built-in compute capability is extremely limited — only increment/decrement of numeric values (things like `INCR`/`ZINCRBY`); most of the time the application layer still has to fetch the data out and compute. Even with an interpreted language (such as Python), a single application-layer operation is microsecond-level, negligible compared to network RTT (0.1~2ms), and more flexible — any logic can be written, unconstrained by Redis's command set. Redis's server-side compute brings no order-of-magnitude advantage; it merely moves the concurrency window from the application layer into the server.

### Composite Key Encoding Dissolves the Middle-Level Abstraction

Once developers master key-space design, Redis's middle-level abstraction can be fully simulated with pure KV + encoding (see [KV Composite Key Encoding](kv-storage-engine-en.md) for details):

| Redis primitive | KV simulation scheme | Key operations |
|:---|:---|:---|
| Hash | `hash:{id}:{field}` prefix encoding | Prefix scan for batch read/write; Tombstone batch delete O(1) |
| ZSET | `zset:{id}:{score_big_endian}:{member}` Score prefix | LSM-Tree byte-order auto-sorting, Seek iterator for Top-N |
| Set | `set:{id}:{member}` → `""` | Existence check, prefix scan enumeration |
| List | `list:{id}:{seq}` monotonically increasing sequence number | Range scan, Raft monotonic counter guarantees a unique sequence |

After simulation you get an API with the same semantic capability, but running underneath on a multi-threaded, disk-first KV engine that supports atomic Batch. The value of the middle-level abstraction — "not designing data structures yourself" — vanishes once the encoding scheme is settled.

### Case-by-Case Takedown: The Substance of the Data-Structure Claims

#### Sorted Sets / Leaderboards
- **Claim**: "Redis sorted sets are very well suited to leaderboards."
- **Reality**: Redis stores everything in memory. A leaderboard of 1 million users permanently consumes several GB of RAM.
- **Alternative**: PostgreSQL (or any true database) automatically manages a **tiered cache**. The top 1000 are hot (in memory via the buffer pool). The rest are on disk, rarely accessed. Cost difference: ~1000 times less memory for the same data.

#### Geospatial
- **Claim**: "Redis GEO commands are convenient."
- **Reality**: geospatial data is large (coordinates + metadata + spatial index). Storing all of it in memory is too costly.
- **Alternative**: PostGIS uses R-Tree indexes and disk storage. Query performance is comparable; memory footprint is orders of magnitude smaller. For large geospatial datasets, Redis is effectively unusable due to memory cost.

#### HyperLogLog (HLL)
- **Claim**: "Redis has built-in HLL for cardinality estimation."
- **Reality**: HLL is not a general-purpose feature. It is rarely used, far less well-known than Bloom filters, and not needed by most applications.
- **Alternative**: HLL is better suited as a **library** (for example Rust's `hyperloglog` crate) or an internal database feature (PostgreSQL uses HLL internally in `pg_stats` for column statistics). It does not justify a dedicated network service.

#### Counters / Rate Limiting: A Deterministic Pseudo-Requirement

**Claim**: "Redis `INCR` / token buckets can do distributed counting and rate limiting."

**Reality**: counters and rate limiting are, among these "shared-state primitives," the ones least suited to Redis — their correct implementation hinges on a key dividing line: **determinism vs non-determinism**.

- **Exact counting** (total transaction value, total item count) — high volume, Redis cannot cope; the correct solution is **stream computing** (e.g. Flink) aggregating over the data stream.
- **Approximate statistics** (UV, deduplicated counts) — cardinality estimation (HLL / Count-Min Sketch) is inherently non-deterministic and need not be carried by an exact structure.
- **Rate limiting** — usually does not require exactness (lossy approximation is fine), adapts in production to load / ML models; and being integrated with the gateway, with local caching + real-time synchronization, it does not itself pursue global exactness.

**The key dividing line**: counters / semaphores / locks are **not three parallel primitives**, but a single **counting substrate** differentiated by use — doing **admission control** (mutex with N=1, semaphore with N, rate limiting by time window) requires **exactness**, since miscounting by one can admit two machines at once; doing **state statistics** further splits by requirement into **exact** (transaction value must balance the books) and **approximate** (UV/dedup inherently need no exactness). Although rate limiting belongs to the admission-control family and **could achieve an exact token bucket**, because it **tolerates loss** (admitting one extra request is not fatal) and lives in the low-latency scenario of a high-performance gateway, it **chooses** endpoint-side local approximation — the same family, yet whether the cost of determinism is worth it is completely different.

**Gateway corroboration**: Redis-style gateways (Kong) are migrating toward etcd-style ones (APISIX) — coordination state injected across gateways, such as rate limiting, should land in a strongly consistent configuration layer, not on Redis's best-effort single point.

#### Pub/Sub, Streams, Queues
- **Claim**: "Redis can do messaging."
- **Reality**: Redis pub/sub has no persistence, no consumer groups, no backpressure. Streams is a poor imitation of a real message queue. A queue is essentially a **sequential write** — the sequential throughput of sequential media such as mechanical disks does not lose much to memory (Kafka is the evidence, see §0 "The Database's Two Core Capabilities"); carrying a sequential-write structure on Redis's memory is a waste in the memory sense.
- **Alternative**: NATS (pub/sub), Kafka (streams), RabbitMQ (queues). These are purpose-built systems for correct durability, ordering guarantees, and consumer management.

#### Transactions (MULTI/EXEC)
- **Claim**: "Redis transactions guarantee atomicity."
- **Reality**: Redis's MULTI/EXEC is a **pseudo-transaction that does not support rollback**. Commands execute in order at EXEC; if a command in the middle fails (such as a type error or OOM), the commands that already succeeded **do not roll back**, leaving permanent dirty data at the application layer. This is fundamentally different from a relational database's ACID transactions.
- **The Hash Slot constraint of Lua scripts**: in Redis Cluster, a Lua script requires all Keys in the transaction to route to the same physical Hash Slot (usually forced via `{tag}`), and cross-node operations error out directly. For distributed ZSET double-writes (updating a score involves the main data table + the ranking index, and the two Keys may land in different Slots), this constraint makes cross-Key atomic operations infeasible in cluster mode.
- **Alternative**: the `Batch` API of an embedded KV engine (Fjall/SurrealKV) provides true atomic writes — all operations commit atomically at once in the WAL, and physically roll back on failure. Combined with Raft consensus, Batch's atomicity extends from a single machine to a cluster majority. Fjall also supports MVCC and optimistic-lock retries, providing deterministic control over concurrency conflicts.

#### Lua Scripts
- **Claim**: "Redis Lua scripts provide atomic multi-step operations, very convenient."
- **Reality**: Lua as an embedded scripting language has significant engineering friction:
  - **1-based indexing**: contrary to the convention of mainstream languages such as C, Rust, Go, Python, Java, and JavaScript, every array operation requires a mental translation, increasing the chance of error.
  - **Scant standard library**: no built-in JSON, regex, or date/time handling. Any non-trivial logic requires external C library bindings.
  - **Ecosystem disconnect**: Lua's type system, toolchain, and testing frameworks are completely detached from the main technology stack (Rust/Go/Python), forfeiting strong typing protection, linter static checks, mature unit-testing capability, and IDE intelligence.
  - **Single-thread blocking risk**: Redis's core is single-threaded. If a Lua script contains complex logic or a long loop, the entire Redis server blocks all other clients' requests until the script finishes. The price of this engineering compromise is a significant productivity loss.
  - **Hash Slot lock-in**: in Redis Cluster, all Keys within a Lua script must fall in the same Hash Slot. Cross-Slot atomic operations are forcibly prohibited — this is not a design flaw but the physical constraint of Redis's sharding architecture. Business logic that needs cross-Key linkage is forced to degenerate into application-layer compensation.

**The SurrealDB contrast**: SurrealDB took the opposite approach — instead of embedding a poor general-purpose scripting language (Lua), it designed a **purpose-built query language (SurrealQL)** that handles data modeling, queries, permissions, and business logic in a single coherent syntax. In the AI era, a DSL's learning cost is negligible — an AI assistant can instantly write flawless SurrealQL. A well-designed DSL eliminates the "embedded-language tax" — no context switching between application code and database scripts.

This is the same first-principles thinking that makes SurrealDB superior to Redis: **do not bolt together a poor embedded language; design a correct interface from scratch.**

### The Genuine High-Level Capabilities Redis Lacks

What key encoding can simulate are only the primitives Redis itself provides. What Redis cannot provide, and cannot make up for through encoding, are the genuine high-level query capabilities:

- **Relational queries**: JOIN, subqueries, conditional aggregation — Redis degenerates into application-layer N+1 network round-trips (detailed in §0, Data Sinking)
- **Declarative logic**: SQL/SurrealQL executes business logic directly at the storage layer; Redis is forced to rely on Lua scripts (single-thread blocking + Hash Slot constraint, §7 Lua Scripts)
- **Transaction rollback**: MULTI/EXEC does not roll back on failure; the KV Batch API provides true ACID atomicity (§7 Transactions)

### Verdict

Redis's data-structure API is a **middle-level illusion** — it looks more advanced than KV, but under Composite Key Encoding its moat is zero; it looks close to a database, but lacks the genuine high-level capabilities.

**The conditionality of middle-level convenience**: Redis's data-structure convenience holds only under a specific combination — Redis as cache, the database storing the real data, and functionality that Redis happens to cover (leaderboards, sessions, etc.). Once pure Redis takes on business logic, developers must nonetheless design composite keys (prefix separation, static/dynamic paths, precise matching), and the supposed convenience of "not designing a key space" does not exist in practice.

**OpenResty + Redis Gateway: Composite Key Design from Immature to Mature**

A typical immature design: one Key per route path, no distinction between static/dynamic, cache and rate limiting sharing prefixes, and Keys scattered across different slots in Cluster mode. A mature scheme requires refined key orchestration in five dimensions:

| Domain | Key encoding | Data structure | Atomic operations |
|:--|:--|:--|:--|
| **Rate limiting** | `rl:{client_id}:{window_ts}` | String (i64) | Lua: GET → judge → INCR → EXPIRE |
| **Response cache** | `{cache}:{method}:{path_hash}` | String (serialized response body) | SET + TTL, invalidation via a separate index Key |
| **Session** | `ss:{user_id}` | Hash (all fields within one Key) | HGET/HSET single-field read/write |
| **Circuit breaking** | `cb:{upstream_id}` | Hash `{state, failures, last_ts}` | Lua: check state → INCR failures → judge threshold → flip state |
| **Routing table** | `rt:{method}` | String (JSON array, sorted by priority) | SET whole rewrite, Lua not needed |

**Key optimization techniques**:

**Hash Tag same-slot**: Redis Cluster assigns slots by the CRC16 of the Key. The string inside `{}` determines the slot. `{cache}:GET:/api/v1/users` and `{cache}:GET:/api/v1/orders` land in the same slot — batch invalidation can then be done with an atomic Lua script without triggering a CROSSSLOT error. An immature scheme does not use Hash Tags, so Keys with the same prefix are scattered across different nodes and batch operations are impossible.

**Hash instead of multiple Keys**: session data uses a single Hash Key `ss:1001` to store all fields (`{name, token, role, last_active}`), rather than one Key per field. When the field count is < 128, Redis Hash uses ziplist encoding (contiguous memory, zero pointer overhead), saving 60-70% of the memory of an equivalent set of String Keys (eliminating each Key's SDS header + DictEntry).

**Lua scripts for compound judgment**: the three steps of rate limiting — "read-judge-write" — must be atomic — two requests simultaneously read count=99, both pass the judgment, both INCR, and the result is count=101, exceeding the limit but already admitted. A Lua script executes atomically within Redis's single thread:
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")
if current >= limit then return 0 end
redis.call('INCR', key)
redis.call('EXPIRE', key, ARGV[2])
return 1
```

**Cache invalidation via an index Key rather than SCAN**: `SCAN` cursor traversal degrades in performance when the Key count is large (each SCAN returns a fixed number, and the cursor state is stateful). The mature approach is to maintain an index Key `ci:{method}:{prefix_hash}` → Set holding the hashes of all cache Keys matching that prefix. On invalidation, `SMEMBERS` retrieves all Keys and DELs them one by one. When writing a cache, `SADD` to the index at the same time. The cost is one extra Redis operation per cache write, but invalidation drops from O(N) SCAN to O(M) precise deletion (M = the number of matches, far smaller than N).

**The physical limits that remain even in this mature scheme**:

| Dimension | Mature Redis scheme | Physical ceiling |
|:--|:--|:--|
| Rate-limiting Lua | Atomic but serialized single-threaded | 100K QPS ceiling (single core); multi-instance sharding adds complexity |
| Cache index | Avoids SCAN | The index Key itself has memory overhead; the Set may bloat under large prefixes |
| Session Hash | A single Key saves memory | Pure memory; RAM cost grows linearly as data grows |
| Circuit-breaking Lua | Atomic state flip | Single-thread bottleneck; Lua queues up under highly concurrent upstream traffic |

The mature scheme squeezes every last drop of capability out of Redis — Hash Tags, Hash ziplist, atomic Lua scripts, index Keys to avoid SCAN. But it cannot physically break through the hard limits of single-threading and pure memory. → The KV Sidecar approach keeps the same key-orchestration thinking, with the underlying engine changed from single-threaded network RAM to a multi-threaded disk-first LSM-Tree, eliminating all the above ceilings. → For the complete design see [Use Case: OpenResty + KV Gateway](kv-storage-engine-en.md#use-case-openresty--kv-gateway)

**Verdict**: the reasons to choose Redis over an embedded KV reduce to two — (1) using it as a cache layer alongside a database, without needing transaction rollback; (2) needing an out-of-the-box leaderboard (ZSET) without wanting to write encoding logic. Both reasons become increasingly untenable in a modern architecture: the cache can be replaced by a CDN/L1 ready-made set (§5), and the leaderboard can be implemented with 200 lines of KV encoding, with disk persistence and multi-threaded concurrency.

### The Pattern: Redis's Claims vs Reality

Every "advantage" of Redis dissolves under scrutiny:

| Use case | Redis "advantage" | Why it is false |
|------|-------------|-------------|
| Cache | Fast memory | Network overhead dominates; local KV is faster |
| Distributed locks | Simple API | Redis is a single point of failure; building your own is simple |
| Leaderboards | Sorted sets | All-in-memory waste; a database's tiered cache is 1000 times cheaper |
| Geospatial | GEO commands | Data too large to fit in memory; PostGIS is superior |
| HLL | Built-in | Rarely used; better as a library |
| Messaging | Pub/sub, streams | No persistence, no consumer groups; NATS/Kafka are the right tools |
| Lua scripts | Atomic multi-step operations | Lua is unfriendly to developers (1-based, no standard library, blocking single thread); Hash Slot restricts cross-node atomic operations |
| Transactions | MULTI/EXEC atomicity | No rollback, failed commands are not undone; the embedded KV Batch API provides true ACID atomic batching |

Redis's "richness of data structures" is not an advantage — it is a **design smell**. It shows that Redis tries to be everything (cache, lock service, message queue, database) without doing any of it well. A modern architecture should **use the right tool for the job**: in-process memory or a local KV (Redb/RocksDB) for caching, a Raft-based store (etcd, Consul) for distributed coordination, PostgreSQL/SurrealDB for complex data structures, NATS/Kafka for messaging, and PostGIS for geospatial.

## 8. RESP Protocol vs Binary Serialization: The Physical Advantage of an Embedded Architecture

Redis's RESP (Redis Serialization Protocol) is a purely text-based protocol split by line, using `+`, `-`, `:`, and `\r\n` as data boundaries. In high-throughput scenarios, this text protocol introduces three layers of overhead:

- **String-scanning overhead**: RESP is line-delimited, so the CPU must scan byte by byte for the `\r\n` separator, then convert ASCII strings into integers. CPU clock cycles are spent on text parsing.
- **Bandwidth waste**: in binary a 64-bit unsigned integer needs only 8 bytes, whereas in RESP's text protocol `18446744073709551615` occupies 20 bytes, plus the type symbol and terminator.
- **Cross-process memory copies**: Redis is a separate process, so data is copied from the kernel Buffer into the Redis process, serialized, transmitted over the network, and copied again into the application process's memory. These multiple copies across processes and the network are overhead that an embedded architecture does not have.

### Bincode: Serialization with Zero Metadata Overhead

Bincode is a Rust-native pure-binary serialization format. A struct containing `i32` and `bool` serializes to exactly 5 bytes — no field names, no separators, no type tags. Deserialization needs no parsing: the CPU reads the bytes from disk (Fjall) or the network (inter-node frame protocol) and, within a few clock cycles, casts them directly into a Rust in-memory struct.

### CBOR: Balancing Dynamic Extension and Hardware Throughput

CBOR is a binary format that retains JSON's dynamic KV extension capability, but encodes field names and types as compact binary headers. The binary header contains the field length, so the decoder can skip fields it does not care about directly, with no byte-by-byte separator scanning.

### Zero-Copy Deserialization

An embedded architecture (Fjall + multi-language engines in the same memory address space of the same Rust process) achieves runtime zero-copy deserialization. After Fjall reads data from NVMe into a shared Buffer, leveraging Rust's lifetime system combined with Bincode/CBOR, the script engine's pointers can point directly at the memory of Fjall's underlying disk buffer, with no extra memory allocation and copying.

```rust
#[derive(Deserialize)]
struct DistributedAgentContext<'a> {
    id: u64,           // fixed 8-byte physical mapping
    status: &'a str,   // zero-copy: the pointer points directly at the Fjall disk buffer
    raw_payload: &'a [u8], // zero-copy: no RAM copy
}

fn execute_zero_copy_pipeline(raw_bytes_from_fjall: &[u8]) {
    let context: DistributedAgentContext = bincode::deserialize(raw_bytes_from_fjall).unwrap();
    // context.status's memory pointer is passed directly to the embedded Steel Lisp / PyO3 for execution
}
```

External Redis, isolated across processes and the network, must put data through multiple copies and RESP wrapping, and physically cannot achieve zero-copy.

## 9. Alternatives

### Persistence Is a False Premise: Redis Undertakes Work That Was Never Its Own

Redis's persistence (snapshots + AOF) is not something it naturally needs, but something forced by "wanting to be used as a database" — **it is essentially a cache, and can be lost**. This creates a structure that pleases no one:

- **Want "fast"** → it can only hold everything in memory, with no guarantee against loss (AOF's default everysec loses one second).
- **Want "no loss"** → back to disk speed (appendfsync always, an fsync on every write), which is then essentially no different from a disk-based database, and slower still.

**The real tiers of need**:

- If you truly require "not a single byte lost" → **write to disk immediately; store in a database / KV (LSM-Tree)**, not a cache. LSM's sequential write + in-memory buffer + batch commit crushes Redis's per-row random fsync in throughput. Batch writes are easier to control in business code (accumulate a batch and commit at once), and off-the-shelf libraries exist for differential writes. Redis's "immediate flush to disk" is not fast, because an independent fsync per command is a random write.
- A lossable cache → needs no persistence at all; Redis carrying this persistence machinery is just a burden.

**Corollary**: Redis's core selling point, "memory-fast + can persist," is a false composite — if it is truly fast, it does not persist; if it must persist, it is no faster than disk. Rather than letting Redis wobble awkwardly between the two, choose one as needed: **want persistence → DB/LSM KV; want speed → an in-process/single-responsibility memory service (§1.5/§8)**. This is exactly the rationale for the alternatives below.

The only remaining justification for Redis is **inertia** — the historical constraint inherited from the PHP era.

1. **For cache/state**: **Fjall** (embedded LSM-Tree KV, Rust-native) — for the complete architecture see [Aura Architecture](aura-architecture-en.md) §5.
2. **For distributed state**: **Fjall (local) + single-write metadata + federated authentication** — the constructive counterpart of this critique. Replaces Redis at L0 (in-process KV); L3 (strongly consistent distributed coordination) is delegated as needed to an etcd-class consensus service or an external TiKV.

---

## Cross-References

- **[Aura Architecture](aura-architecture-en.md)**: the complete architecture design of Fjall + independent metadata (single-write, no consensus) — a modern distributed stall engine integrating storage and compute.
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: this critique's counterpart at the scripting layer.
- **[Fractal](Fractal.md)**: a first-principles engineering-decision closed loop — the "Occam's razor" methodology shared by Redis, Koto, and Helix.
- **[agentmemory](agent-memory-en.md)**: the persistence scheme for the Agent memory layer. agentmemory chose SQLite + local embedding over Redis, confirming the core argument of this critique — network latency dominates Redis's microsecond-level processing advantage, and embedded local storage is the better solution.
- **[Lakehouse Research](lakehouse-research-en.md)**: the index version-number interface shares its source with the Lakehouse Catalog — both lift version governance from the storage layer to the control plane, replacing cache-invalidation mechanisms with the simplest implementation.
- **[Flux Architecture](flux-architecture-en.md)**: the complete standalone exposition of §4 — from passive pull to active push, Fluxora = Flux + Aura.
- **[Long-Lived Connection Engineering](long-lived-connection-engineering-en.md)**: the physical limits (backpressure, fragmentation, connection skew) and solutions for WS/WT long-lived connections under a compute-storage-integrated architecture.
