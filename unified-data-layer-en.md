# Unified Data Layer Architecture: PostgreSQL for Exploration, KV + Iceberg + DuckDB at Scale

> **Languages:** [English](unified-data-layer-en.md) (primary) · [中文](unified-data-layer.md)

**Conclusion**: the data stack is not one universal thing; it splits into two paths by **project characteristics** — **exploration / agile** uses PostgreSQL without thinking, **scale / predictable** converges to **KV + Iceberg + DuckDB**. The axis of the fork is "requirement predictability × time horizon × load weight (scale)"; load tiering (OLTP/OLAP) is only the internal organization of path B, not the single global answer.

## Criteria: the Fork Axis of the Two Paths

There is no "single correct stack" for a unified data layer, only the "stack that suits this project's state right now". The fork is decided by three orthogonal dimensions:

- **Requirement predictability**: you control the requirements and the queries can be fixed at design time → worth betting on KV; requirements change with the outside, unpredictable → not worth building the keyspace in advance.
- **Time horizon / lifecycle**: exploration and validation, undecided lifecycle → use the universal solution with the lowest mental burden; long-term evolution, wanting to eliminate middlemen entirely → converge to the minimal stack at the physical limit.
- **Load weight / scale**: light load, no need for multiple instances, can share a centralized storage pool (pooling dividend); heavy load, when the data volume is large enough that the shared pool becomes the bottleneck and even a dedicated instance hits its ceiling, **heavy load can ultimately only choose KV** (in-process zero RTT, zero parsing, a single instance carries the volume, and if volume grows further there is a smooth scaling exit).

When two dimensions both point to "temporary attachment" → **path A (PostgreSQL)**; both point to "long-term scale" → **path B (KV + Iceberg + DuckDB)**. The middle ground (requirements already predictable, but no scale demand yet) also belongs to path B — KV's zero operations lets it still hold when there is no scale scenario. The load dimension runs in the same direction as the other two: light load, the centralized pooling advantage holds and takes path A; heavy load, ultimately only path B's KV can be chosen.

## Path A: Exploration / Agile — PostgreSQL Without Thinking

The creed of contemporary data engineering: **if you don't know which database to use, just use Postgres without thinking and you'll be right.**

PostgreSQL's general-purpose capability makes it the optimal default for the exploration phase:

- **Relational + JSONB**: structured and semi-structured data caught by one engine, with a schema flexible enough to approach NoSQL.
- **Extension ecosystem**: graph (Apache AGE), vector (pgvector), full-text search, spatiotemporal — one process covers the vast majority of "don't make me decide now" workloads.
- **Lowest mental burden**: the most complete ecosystem, the most familiar operations, the most talent, and every off-the-shelf tool connects directly.
- **Centralized service form (unique to path A; a solid advantage over path B's embedded form)**: PostgreSQL is deployed as a standalone service, a single instance can serve multiple projects (resource pooling, reuse of connections and buffers), it can be hosted in the cloud (RDS/CloudSQL removing self-operations), and the application layer stays stateless (storage externalized as the sole durable point). These three dividends **come from the "centralized vs embedded" deployment form, not from a property of the SQL language** — if path B wants the same dividends, the correct answer is to switch to a **centralized KV server** — that is, the combination of a KV engine + an HTTP/WS network access layer (internally WS+Postcard, externally HTTP as a fallback, see [KV Storage Engine](kv-storage-engine-en.md) "The Composition of a KV Server"), rather than returning to SQL. They are recorded here as advantages of path A precisely because path A happens to land in the PostgreSQL = centralized form; the claim derives from the centralized property and merely lands on PostgreSQL (for attribution discrimination see [KV Storage Engine](kv-storage-engine-en.md) "Resource Pooling / Cloud Hosting / Stateless: A Topological Property, Not a Property of SQL").

**Applicable to**: MVPs with uncertain requirements, rapid validation, agile iteration by a single team, experimental projects with an undecided lifecycle. Betting early on a storage choice at this point is a negative return — PostgreSQL uses one engine to defer your choice until the requirements are clear.

**Upgrade signal (when to migrate to path B)**: when requirements solidify into predictability, the data volume starts to grow, and long-term archival and analytics demands appear, PostgreSQL's relational parser runtime becomes an unnecessary middle layer. Migration = remodelling along the keyspace: first enumerate the access patterns that have solidified, then work backwards to the composite key layout (see [KV Storage Engine](kv-storage-engine-en.md) §Access-Pattern-Driven).

## Path B: Scale / Predictable — KV + Iceberg + DuckDB

Teams that evolve long-term and pursue minimalism converge their data stack onto a two-piece set: embedded KV manages mutable state, Iceberg manages history and truth, DuckDB handles analytical queries. **Path B has no PostgreSQL** — KV is not merely listed alongside PostgreSQL, it replaces PostgreSQL's OLTP role.

```
┌────────────── OLTP Mutable State ──────────────┬──────────────────── OLAP Static Analytics ─────────────────────┐
│ Embedded KV (Fjall / SlateDB + S3)             │ DuckDB (Query Layer)                                           │
│ Predictable query patterns / large-scale state │ Embedded columnar, vectorized execution                        │
│ Replaces PostgreSQL's OLTP role                ├────────────────────────────────────────────────────────────────┤
│ In-process zero RTT · single binary            │ Iceberg (Storage Layer)                                        │
│                                                │ S3 as Truth + stateless Catalog                                │
│                                                │ Read only needed columns, high compression, aggregate pushdown │
└────────────────────────────────────────────────┴────────────────────────────────────────────────────────────────┘
```

- **OLTP: embedded KV** — predictable query patterns / large-scale state use KV instead of PostgreSQL. Whether it is predictable depends on the controllability of the requirements (the say you have), not on organizational identity. Zero parsing, zero optimizer, in-process zero RTT, a single binary; when the data volume is large, KV's physical advantage holds even if the patterns vary. For the full argument see [KV Storage Engine](kv-storage-engine-en.md).
- **OLAP: Iceberg + DuckDB** — history and analytical data goes to Iceberg landing on S3 (object storage as the sole Truth), DuckDB serves as the embedded query layer, reading only the needed columns, compressing homogeneous values, pushing down filters and aggregates. The Catalog can be made stateless: the local KV is only a `table → snapshot` cache, and if lost it is rebuilt by scanning S3. See [Lakehouse Research](lakehouse-research-en.md) for details.
- **Durability and distribution**: the Lakehouse simultaneously carries OLTP's **durable backup** and the external **OLAP service** — an asynchronous Flush lands the Fjall state into Iceberg, and DuckDB and third-party engines read the same S3 data through the standard table format. The distribution form is chosen between "Fjall + Lakehouse" and "SlateDB + S3", see the next section, "The Two Distribution Paths".
- **The convergence philosophy**: eliminate the middlemen — Redis's networked cache, the relational parser runtime, the operations of a separate warehouse are all absorbed by this two-piece set. The service is the database, the database is the service.

### The Two Distribution Paths: Fjall + Lakehouse vs SlateDB + S3

The watershed starts with **temporal locality**: if the hot data is recently written and can be caught by the memtable / local cache (things like stall messages, session state), the cold-read probability is low — SlateDB's S3 round-trip is triggered only by occasional point lookups, which is acceptable, and it can even **replace the "Fjall + Lakehouse" two-piece set** (single truth, no flush pipeline). Only when hot reads have a hard requirement for in-process ns-μs, and reads have no temporal locality, is Fjall necessary.

| Dimension | Fjall + Iceberg | SlateDB + S3 |
|:---|:---|:---|
| Location of truth | local disk (serving) + Iceberg landing in the lake, two places | S3 is the truth, one place |
| Hot-path serving latency | in-process zero RTT (ns-μs) | fast on a cache hit; cold reads go to S3 (ms) |
| Durability / disaster recovery | single-node local; lake-landing lag = crash data-loss window | S3 3-AZ native, no separate backup pipeline |
| Cross-instance sharing / read scaling | single-node in-process; sharing must go through Iceberg/consensus | multiple instances read shared S3, reads scale horizontally by nature |
| Write path | local WAL→SSTable (extremely fast) | memtable→asynchronous flush to S3 |
| Number of architectural pieces | two sets + a self-built flush pipeline | one set |
| Consistency | eventual between KV and Iceberg (flush lag) | single truth, coordinated by manifest |
| Analytics | DuckDB reads Iceberg (standard table format / time travel / evolution) | DataFusion queries directly; connecting DuckDB/BI requires exporting Parquet/Iceberg |
| Target scale | single-machine medium scale + separated archival analytics | large scale, many writers and readers, storage-compute separation |
| Maturity | Fjall is mature; the pipeline is self-built | SlateDB is relatively new, production unproven |

**Fjall + Iceberg wins on "hot"**: in-process direct `scan` serves requests with a deterministic response, while Iceberg covers durability and analytics; the cost is two sources of truth + flush lag (a node crashing before landing in the lake loses the most recent writes), and hot data cannot be shared across instances through consensus replication — Raft/Openraft does not shard, every node holds a full replica, storage and write amplification grow linearly with the number of nodes, and it is only fit for managing metadata-scale data.

**SlateDB wins on "single truth + scaling"**: S3 as the sole truth, no separate backup pipeline, multiple instances reading in common, collecting the full storage-compute separation dividend (replicas fixed at 3-AZ, independent of the node count); the cost is the S3 round-trip for cold reads, with the hot path relying on memory/local cache as a fallback, and point lookups not matching Fjall's embedded zero RTT.

**Decision**: strong temporal locality, or accepting occasional cold reads → SlateDB + S3 (single truth, scalable); a hard requirement for in-process ns-μs and reads with no temporal locality → Fjall + Iceberg. The two are paths with different optimization goals (single truth + scaling vs in-process hot latency), not like-for-like replacements.

**Do not self-build strongly consistent distribution**: if what you need is strongly consistent data replication + horizontal scaling (the "sharding + Raft" shape), do not bolt an embedded engine together into sharding + Raft (that amounts to rebuilding a distributed database); use mature FoundationDB / TiKV directly — for cross-shard ACID use FDB, for a sharded KV + intra-shard consistency use TiKV. For the comparison see [KV Storage Engine](kv-storage-engine-en.md) §Strongly Consistent Distribution.

## Why Path B Also Does Not Choose a Single Engine (SurrealDB)

Converging path B into a multi-model single engine (such as SurrealDB) is a route that has been **evaluated and not adopted**. The Matthew effect dictates that a challenger must form an overwhelming advantage in enough dimensions to overcome ecosystem inertia — SurrealDB crushes on language design, polymorphic relations, and compute pushdown, but is at a comprehensive disadvantage in ecosystem, toolchain, cloud support, and data consistency; **a mix of wins and losses = the challenge fails**.

SurrealDB's killer move, "compute pushdown", is itself not free — it splits execution into two worlds: basic queries go to declarative SurrealQL (the underlying core schedules them directly and goes through native indexes), while heavy algorithms go to precompiled WASM extensions (Surrealism, near-native machine code); every exchange of data between the two must pass through the Host Bridge for serialization and memory copying, and when traversing at scale the boundary-crossing overhead is significant. The embedded KV's "compute next to the data", by contrast, is a starting-point property: same process, same binary, `embed` is a single native call, zero serialization, zero boundary, naturally atomic (for the mechanism see [KV Storage Engine](kv-storage-engine-en.md) "The Case for Staying Pure KV"). This dimension degrades from "crushing" to "a mix of wins and losses", and the balance tilts further toward the two-piece set.

And the two-piece set splits OLTP from OLAP at a scale no single engine can reach: graph / multi-model requirements are absorbed by KV through multi-model encoding, and the analytical load goes to DuckDB · Lakehouse. For the full evaluation see [Query Language Design](query-language-design-en.md).

## The Boundary Between OLTP and OLAP

- **OLTP (mutable state)**: in path A carried by PostgreSQL, in path B by embedded KV (predictable query patterns, large-scale state). This layer handles real-time reads and writes.
- **OLAP (static analytics)**: DuckDB + Iceberg (path B) or ad-hoc analytics in path A. History data flows into the lake through CDC / batch Flush, separating mutable state from static analytics, see [Lakehouse Research](lakehouse-research-en.md) and [KV Storage Engine](kv-storage-engine-en.md) for details.

## Storage-Compute Separation and Layering Position

- **PostgreSQL (path A)**: monolithic extension (plugins/JSONB), storage-compute separation provided by the cloud vendor.
- **KV (path B)**: embedded (Fjall, single-writer metadata without consensus) or S3-backed (SlateDB), in-process zero RTT or object-storage durability.
- **DuckDB (path B)**: embedded columnar, storage-compute separation provided by the Lakehouse.
- **Iceberg (path B)**: S3 + Catalog, thoroughly decoupling storage from compute.

## Conclusion

There is no "one engine does everything", and no "one stack fits every project". **Exploration / agile opens with PostgreSQL without thinking, scale / predictable converges to KV + Iceberg + DuckDB**. The fork axis is requirement predictability, time horizon, and load weight (scale); load tiering is the internal organization of path B. SurrealDB's single-engine unified path was evaluated and not adopted — its query-language insights are retained in the evaluation dossier, but the core architecture does not depend on it.

## Cross-References

- **[KV Storage Engine](kv-storage-engine-en.md)**: embedded KV design (single-writer metadata without consensus; data-level strongly consistent replication handed to external TiKV-class solutions); the argument for KV replacing PostgreSQL under predictable query patterns and large-scale scenarios.
- **[Lakehouse Research](lakehouse-research-en.md)**: analytics-layer selection — Delta/Iceberg/Lance, object storage, landing patterns, Catalog statelessness.
- **[Query Language Design](query-language-design-en.md)**: a full evaluation of SurrealQL language philosophy, graph modelling, and challenger logic (why it did not enter the core).
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: the network RTT trap and cache tiering.
- **[Aura Architecture §5](aura-architecture-en.md)**: the lightweight solution when fully embedded KV and consensus are required.