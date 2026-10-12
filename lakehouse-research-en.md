# Lakehouse Research

> **Languages:** [English](lakehouse-research-en.md) (primary) · [中文](lakehouse-research.md)

## 1. The Landscape of Analytical Workloads

Analytical workloads are not the same as OLAP. OLAP is one category within them — specifically analytical databases that accept interactive user queries. There are other analytical workloads that do not follow the OLAP path.

| Type | Is a Database | Accepts User Queries | Representative Technologies |
|:---|:---|:---|:---|
| OLAP (MPP) | Yes | Yes (interactive) | ClickHouse, Snowflake |
| OLAP (Lakehouse) | Yes | Yes (interactive) | Delta Lake, Iceberg + DuckDB/Polars |
| Streaming Database | Yes | No (maintains materialized views) | RisingWave, Materialize |
| Stream Processing Framework | No | No (pipeline) | Flink, Spark Streaming |

**MPP** (ClickHouse, Snowflake): columnar storage + vectorized engine + distributed parallel queries. Extremely low latency (sub-second), but storage and compute are coupled, scaling means adding nodes, and the cost is high.

**Lakehouse** (Delta Lake / Iceberg + object storage): layers a table format on top of cheap object storage, with the query engine (DuckDB/Polars/Spark) started on demand. Storage and compute scale independently, and storage cost is one to two orders of magnitude lower. The price is slightly higher query latency and the need to manage compaction.

**Streaming databases** (RisingWave, Materialize): continuously maintain materialized views, compute in real time on write, and read the precomputed tables directly on query. Latency drops from the minute scale to the millisecond scale. Not OLAP.

**Stream processing frameworks** (Flink, Spark Streaming): general-purpose stream processing engines that require you to manage state and windows yourself. Not databases; they belong to the data pipeline.

**Apache Kylin (data cube/Cube)**: precomputes materialized views for every combination of dimensions. Elegant in theory, with obvious weaknesses in practice — Cube construction is slow and storage explodes, adding a dimension requires rebuilding the Cube, and scalability is poor. The technology no longer grows; it is in the withering phase of its technology lifecycle.

**⚠️ Common confusion**: OLAP is a subclass of databases, not a synonym for "analytical workload." RisingWave is an analytical database but not OLAP.

---

## 2. Why Choose Lakehouse

The advantages of object storage (S3/OSS) over file storage are overwhelming: one to two orders of magnitude lower cost, unlimited elastic scaling, native multi-replica disaster recovery, and no need to operate a storage cluster.

The shortcomings of object storage are latency and metadata — a single request has higher latency than a local disk, and directory listing degrades in performance with massive numbers of files. But **analytical workloads are precisely the kind insensitive to latency**: waiting seconds to minutes after submitting a query is perfectly acceptable. The lakehouse only needs to solve the metadata problem (through the table format's metadata layer or a REST Catalog) to inherit all the advantages of object storage.

Dedicated cluster storage for MPP cannot compete with it on cost or scalability. **The lakehouse may be the endgame of analytical architectures.**

---

## 3. Core Mechanism: Concurrency Control

> The iron law of object storage determines the design space of every lakehouse table format.

### The Nature of Object Storage

Object storage was designed from the outset for: **high concurrency, high throughput, immutability**. It has no row-level locks, no file locks, no atomic overwrite. Doing concurrency control on the data plane violates that original intent.

### Why latest.json Does Not Work

Suppose you place a `latest.json` in OSS and overwrite it with a new pointer on every update:

- **Lost Update**: two write scripts overwrite at the same time; object storage cannot provide row-level locks, the old content may arrive later and win, and the newly written file becomes a permanent orphan.
- **Cache pollution**: overwriting the same file name at high frequency causes eventual-consistency lag in CDN/gateway cache synchronization, so the query engine gets a garbled version.

### Two Solutions

| Dimension | Delta Lake | Iceberg + Catalog |
|:---|:---|:---|
| Concurrency control | PutIfAbsent atomic operation | Catalog in-memory lock / database transaction |
| File strategy | One-way append, no overwrite | No duplicate names, no overwrite + outer pointer switch |
| Query performance | Slow (scans a long log chain) | Fast (reads the latest snapshot directly) |
| High-frequency updates | CoW, severe write amplification | MoR, near-zero write amplification |
| Complexity | Medium (no Catalog needed) | Requires a Catalog service (or built into S3 Tables) |

**Delta**: never overwrites any file, using the PutIfAbsent atomic operation to do one-way appends. The cost is that a long log chain slows queries.

**Iceberg**: delegates pointer overwriting to the outer Catalog. The Catalog completes the pointer switch instantly in memory or in a database transaction, and queries read the latest snapshot directly.

**The value of OSS Tables / S3 Tables**: making the Catalog a native capability of object storage — the bucket itself becomes a metadata service center, and users need not deploy any additional service. S3 Tables changes the selection weights: previously Iceberg's biggest pain point was "you have to build your own Catalog"; now the Catalog is internalized as a native storage capability.

### The Nature of the Catalog: System Tables, Which Can Be Non-Relational (Even Stateless)

The metadata backend of a REST Catalog (Lakekeeper/Polaris) commonly uses PostgreSQL, but this is merely implementation inertia, not a structural necessity — what the Catalog stores is essentially a set of "system tables":

- **Table metadata**: schema, partitioning, sorting
- **Current valid snapshot**: the manifest file list
- **Transaction log**: pointer switching for concurrent commits

What these three layers share is **an extremely small data volume** (a few KB to a few MB per table), a **fixed access pattern** (point lookup `table → snapshot` + range scan `snapshot → file list`), and **strong consistency** — landing squarely in the predictable query domain of the [KV Storage Engine](kv-storage-engine-en.md). An embedded KV (Fjall) is viable, and it avoids the whole parser/optimizer/authorization runtime that PostgreSQL carries.

One more radical step is **making the Catalog stateless**:

- **Iceberg's philosophy**: "metadata is data" — `metadata/*.json` itself lives on S3, and the Catalog only serves as a fast index for `table → current_snapshot`, **not the source of the data**.
- **The KV is only a cache**: if it breaks, is lost, or restarts, rebuild it by scanning the S3 prefix (read `version-hint.text`, or sort `v{N}.metadata.json` and take the latest); rebuilding hundreds or thousands of tables takes **on the order of hundreds of milliseconds**.
- **Downgraded to a stateless index**: the Catalog drops from a "stateful service" to a "stateless index," living and dying with the application process — if the application crashes, one scan brings it back.

This converges in the same direction as the earlier internalization of the Catalog into a native storage capability by OSS/S3 Tables: one lands in the cloud bucket, the other inside the application process.

The permission boundary thereby shifts to object storage: **the Catalog is undefended**, and data security is carried by the storage layer — the application's IAM Role has only write (`PutObject`), and the DuckDB/BI engine's Role has only read (`GetObject`). Controlling the S3 write policy is controlling data security, replacing the ACL/RBAC maintained inside the Catalog.

#### Lock-Free Concurrent Commits: The Atomicity of S3 Conditional Writes

Iceberg's pointer switching can also be pushed down to object storage conditional writes, avoiding an external lock service (DynamoDB/Zookeeper): write `v{N}.metadata.json` with `If-None-Match: *`; success means you have won the right to commit, while `PreconditionFailed` means a conflict — pull the latest metadata, merge the manifests, and retry as `v{N+1}`. **Note**: this header is AWS S3 / MinIO semantics; Alibaba Cloud OSS does not support it (see §7 — the very same `If-None-Match` is what Delta gets stuck on with OSS), so it holds only on standard S3-compatible stacks.

---

## 4. Table Format Selection

### Core Comparison

| Feature | Delta Lake | Iceberg | Lance | Vortex |
|:---|:---|:---|:---|:---|
| **Storage mechanism** | Append-only log + Parquet | Snapshot + Manifest + Parquet | Self-contained directory (Fragments + Manifest) | Monolithic file (configurable internal layout) |
| **Concurrency control** | PutIfAbsent (storage layer) | Catalog pointer switching | Built-in MVCC | None (relies on the table format layer) |
| **Write mode** | CoW (write amplification) / MoR | MoR (append-dominant) | Append-only | No native writes |
| **Random reads** | Slow (log chain) | Fast (snapshot direct) | Extremely fast (row-level addressing, 10-100x faster than Parquet) | Fast (cascading adaptive compression) |
| **Full scan** | Fast (Parquet compression) | Fast (Parquet compression) | Medium | Extremely fast (an order of magnitude faster than Parquet) |
| **AI/Vector** | None | None | Built-in (IVF_PQ, HNSW, BM25) | None |
| **Metadata format** | JSON log | Protocol Buffers | Protocol Buffers | FlatBuffers (O(1) tail loading) |
| **GPU** | Via Arrow | Via Arrow | Via Arrow | Native GPU decompression + zero-copy |
| **Catalog dependency** | Optional | Required (or built into S3 Tables) | None (self-contained) | None |
| **Ecosystem maturity** | High (Spark/Databricks) | High (Spark/Trino/DuckDB) | Medium (good Python/Rust ecosystem, early support in big-data engines) | Low (LF AI & Data project) |

### Selection Decision

**Decision reasoning (ADR) for choosing Lance**:

This decision is scoped to **modern teams using a lightweight Python/Polars/DuckDB stack and running in container environments**. For traditional BI teams using Spark/Trino/Tableau, or PB-scale data platforms, the selection conclusion here does not directly apply.

For data assets of **tens of thousands to millions of rows**, Lance's current disadvantages in pure OLAP scenarios (compression ratio or scan throughput versus Parquet) can be ignored. Whether it is 50 thousand rows or 2 million rows, aggregation with Polars and DuckDB takes **<50ms** in both cases, and the physical file-read difference is imperceptible to users.

Three-stage reasoning:

- **Today (Day 1): avoid the OSS compatibility quagmire and the operational burden** — Delta Lake `delta-rs` 1.6 has an `If-None-Match` header bug on Alibaba Cloud OSS, and Delta depends heavily on a Catalog service. Lance gives short-lived containers a purely stateless, zero-service way of running, and its Append-only mechanism achieves lock-free concurrency.
- **Tomorrow (Day 2): inherit performance dividends at zero cost** — 6-12 months later, as Lance pushes low-level optimizations (such as **BtrBlocks, adaptive compression protocols**) into the Python ecosystem, the system will automatically gain significant OLAP performance improvements without changing any Skill code or DDL.
- **The day after (Day 3): native AI/vector readiness** — if the finance Skill introduces **vector auditing (GraphRAG / Semantic Search)** next quarter, the underlying storage already has mature vector retrieval capability, with no storage-layer refactoring needed.

**Iceberg (primary, 50%)**: the right answer for high-frequency update / delete scenarios. MoR cures write amplification at the root, and the REST Catalog eliminates the metadata bottleneck. Requires a Catalog service (Lakekeeper, Polaris, or built into S3 Tables).

**Lance (30%)**: first choice for AI/ML-native scenarios. Run SQL filtering with one hand and vector retrieval with the other on the same table. Zero Catalog, zero service, suited to containerized lightweight stacks. Weaknesses: pure full-table-scan throughput is inferior to Parquet, and big-data engine (Spark/Trino) support is early.

**Delta Lake (20%)**: append-only batch write scenarios. Writing is simple, no Catalog needed. In high-frequency update scenarios, CoW write amplification is fatal, so it does not apply.

**Vortex**: the next-generation Parquet replacement format. A pure storage format, targeting the Iceberg File Format API. The ecosystem is still early.

**Design philosophy**:
- **Lance**: row-level addressing + a self-contained system (MVCC, vector indexes). Positioned as a lightweight embedded database.
- **Vortex**: "the LLVM of file formats" (adaptive encoding, SIMD alignment). A purely storage format, targeting the Iceberg File Format API.

### Lance's Detailed Trade-offs

Lance exhibits a pronounced asymmetric character in analytical business:

**Strengths:**

1. **Random reads of sparse columns are markedly better than Parquet**: Parquet's minimum read unit is a Row Group (tens of thousands of rows), and querying a specific few rows requires downloading the whole row group. Lance supports row-level random access, so in sparse filter queries its I/O speed is more than an order of magnitude faster than Parquet.

2. **Zero operations in the mixed "analytics + updates" scenario**: after high-frequency updates, Iceberg/Delta must run COMPACT late at night to merge the fragmented files, or daytime queries are severely delayed. Lance's metadata version chain is Rust-driven and embedded inside the files, so read performance is naturally stable under daytime high-frequency updates, with no extra scheduled tasks.

3. **The ceiling of multimodal analytics**: the same table supports SQL filtering + vector similarity retrieval + full-text search, avoiding the movement of data between the data warehouse and the vector database.

**Weaknesses:**

1. **Full-table-scan throughput is slightly inferior to Parquet**: after 10+ years of refinement, Parquet's columnar compression ratio (Bit-Packing, dictionary encoding, ZSTD) remains the industry ceiling. Lance's file layout is somewhat looser in order to accommodate high-frequency updates and vector retrieval.

2. **A maturity gap in compute-engine optimizers**: DuckDB/Polars predicate pushdown and multithreaded prefetch optimizations for Parquet are far more mature than for Lance. The gap is obvious in complex multi-table nested Join aggregations.

**Conclusion**:
- Pure T+1 report analytics → stay with Parquet (Lance has no advantage)
- High-frequency updates + point queries → boldly use Lance
- AI/vector scenarios → Lance is the only choice

### Scenario Mapping

| Dimension | Choose LanceDB | Choose Delta/Iceberg + Parquet |
|:---|:---|:---|
| **Data type** | AI-driven (Embedding, long text, multimodal) | Purely structured (numbers, dates, strings) |
| **Team tech stack** | Python / Polars / DuckDB | Spark / Trino / Tableau |
| **Data scale** | Hundreds of GB to several TB | Multiple TB to PB |
| **Operations preference** | Zero service, no Catalog | Enterprise-grade Catalog + ACID governance |
| **BI tool integration** | Custom dashboards only | Tableau, Power BI, traditional BI |

---

## 5. Object Storage Selection

For containerized / self-hosted lakehouse scenarios:

| Dimension | RustFS | MinIO | Garage |
|:---|:---|:---|:---|
| **Language/Architecture** | Rust, decentralized peer-to-peer architecture | Go, Master/Agent architecture | Rust, leaderless architecture (CRDT) |
| **Consistency model** | Strong consistency (read-after-write) | Strong consistency (CP) | Eventual consistency (AP/CRDT) |
| **Delta CAS support** | ✅ Native | ✅ Mature | 🔴 Probabilistic (LWW risk) |
| **S3 Tables / REST Catalog** | ✅ Native (2026+) | ❌ Not supported | ❌ Not supported |
| **Resource footprint** | Low (<100MB binary, Rust has no GC) | High (Go runtime, >2GB memory) | Low (<100MB binary) |
| **Hybrid deployment** | ✅ Tiered storage (local NVMe hot tier + cloud cold tier, automatic tiering) | ❌ Local only | ❌ Local only |
| **Deployment convenience** | Single-command startup, Linux/Docker/K8s/macOS/Windows | Single-command startup, Linux/Docker/K8s | Relatively complex |
| **MCP Server** | ✅ Native support | ❌ | ❌ |
| **Protocol** | Apache 2.0 | AGPL v3 (commercial restrictions) | AGPL v3 |

- **RustFS (first choice)**: written in Rust, combining performance with memory safety. Decentralized peer-to-peer architecture with no Master single point. Inline erasure coding protects data. Native S3 Tables protocol — the bucket itself can serve as an Iceberg REST Catalog, with built-in Auto-Compaction. Local deployment is minimal, a single binary + single-command startup. Apache 2.0 license, commercially friendly.
  - **Tiered storage (the core differentiator)**: local NVMe SSD serves as the hot data tier (Rust + io_uring squeezes disk I/O dry), and external object storage (S3/MinIO/COS) as the cold data tier. Automatic tiering through lifecycle rules — cold data migrates automatically to the cloud, hot data stays local. Essentially a hybrid model of "blazing local writes + low-cost cloud storage."
  - **MCP Server**: native support, good AI ecosystem integration.
- **MinIO (alternative)**: battle-tested, the enterprise standard. The AGPL license carries commercial restrictions. The Go runtime has a relatively high resource footprint. Does not support S3 Tables.
- **Garage (not recommended)**: leaderless architecture, CRDT + Dynamo-style arbitration (AP-leaning). The storage layer has no atomic CAS, so "Last Write Wins" silently overwrites the transaction log, and Delta's MVCC state splits physically → data corruption.

**Recommendation**: RustFS first (it comprehensively surpasses MinIO), MinIO for safety, not Garage.

---

## 6. Deployment Patterns

### Pattern One: DuckDB Querying Parquet Directly (Simplest)

The lightest analytical pattern — the same idea as SQLite reading local files. Put Parquet in object storage, spin up DuckDB on demand to query it, zero service, zero operations.

```
┌─────────────────────┐        ┌─────────────────────┐
│  Incremental Write  │        │  Analytical Query   │
│  (Python / Polars)  │        │  (DuckDB + httpfs)  │
└────────┬────────────┘        └────────┬────────────┘
         │                              │
         ▼                              ▼
  ┌──────────────────────────────────────────┐
  │  s3://bucket/finance_table/              │
  │    append_20260101_120000.parquet        │
  │    append_20260102_093000.parquet        │
  │    ...                                   │
  └──────────────────────────────────────────┘
```

**Write side**: Polars writes timestamped `.parquet` files directly to OSS. Lock-free, no directory service — pure append.
**Query side**: DuckDB scans with the wildcard `*.parquet`, automatically merging all files. Projection pushdown and predicate pushdown minimize I/O.

```python
# Write
from datetime import datetime
import polars as pl

def sync_incremental(new_rows: pl.DataFrame):
    ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
    path = f"s3://your-bucket/finance_table/append_{ts}.parquet"
    new_rows.write_parquet(path, storage_options=oss_options)

# Query
import duckdb

def run_summary():
    con = duckdb.connect()
    con.execute("""
        SET s3_endpoint='oss-cn-hangzhou.aliyuncs.com';
        SET s3_access_key_id='YOUR_ACCESS_KEY';
        SET s3_secret_access_key='YOUR_SECRET_KEY';
        SET s3_url_style='path';
    """)
    return con.execute("""
        SELECT month_col, employee_id, SUM(amount) AS total
        FROM 's3://your-bucket/finance_table/*.parquet'
        GROUP BY month_col, employee_id
    """).pl()
```

**Note**: as time goes on, the accumulation of small files increases HTTP round trips. Periodically run a compaction script to merge large files.

**Compaction script and safety notes:**

```python
# Compaction sketch
df_all = con.execute("SELECT * FROM 's3://your-bucket/finance_table/*.parquet'").pl()
df_all.write_parquet(
    "s3://your-bucket/finance_table/combined_v1.parquet",
    storage_options=oss_options
)
# Then safely remove historical append_*.parquet files
```

**Safety note**: when deleting fragments, make sure no concurrent writes are in progress (for example, merge only files from before the current day, or use a temporary-rename pattern).

### Pattern Two: LanceDB (AI/Vector Ready)

Zero Catalog, zero service, suited to containerized lightweight stacks. The same table supports SQL filtering + vector retrieval.

**Schema definition strategy:**

| Data type | Use case | Recommended |
|:---|:---|:---|
| Financial amount | Amounts, balances | `pa.float64()` (not float32, to avoid precision loss) |
| Time/audit | Timestamps, dates | `pa.date32()`, `pa.timestamp('us', tz='Asia/Shanghai')` |
| Fixed nesting | Invoices, detail rows | `pa.struct([...])` (supports projection pushdown) |
| Dynamic tags | Unpredictable JSON | `pa.map_(pa.string(), pa.string())` (prevents schema drift) |

**Complete code template (four functions):**

```python
import os
from datetime import datetime

OSS_OPTIONS = {
    "aws_access_key_id": os.environ.get("OSS_ACCESS_KEY_ID"),
    "aws_secret_access_key": os.environ.get("OSS_SECRET_ACCESS_KEY"),
    "aws_endpoint": "https://oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou"
}
DB_URI = "s3://your-finance-bucket/lancedb_core_lake/"

# 1. Schema initialization (explicit definition to prevent schema drift)
def initialize_blank_financial_lakehouse():
    import pyarrow as pa
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    if "financial_records" not in db.table_names():
        explicit_schema = pa.schema([
            pa.field("tx_id", pa.string(), nullable=False),
            pa.field("amount", pa.float64(), nullable=False),
            pa.field("year", pa.int32(), nullable=False),
            pa.field("month", pa.int32(), nullable=False),
            pa.field("invoice_detail", pa.struct([
                pa.field("invoice_no", pa.string()),
                pa.field("tax_rate", pa.float64())
            ])),
            pa.field("custom_tags", pa.map_(pa.string(), pa.string()))
        ])
        db.create_table("financial_records", schema=explicit_schema, storage_version="2.0")

# 2. Incremental sync (lazy import, cold start <10ms)
def skill_data_sync_lancedb(incoming_pl_df):
    import polars as pl
    import lancedb
    initialize_blank_financial_lakehouse()
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    payload = incoming_pl_df.with_columns(pl.lit(datetime.now()).alias("ingested_at"))
    table.add(payload)
    return {"status": "success", "version": table.version}

# 3. Analytical query (zero-copy LazyFrame)
def skill_analytical_query_lancedb(sql_filter: str):
    import polars as pl
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    lazy_lance = table.to_lance().to_lazy()
    result = (
        lazy_lance.filter(pl.sql(sql_filter))
        .group_by(["year", "month"])
        .agg(pl.col("amount").sum())
        .collect()
    )
    return {"status": "success", "data": result}

# 4. Weekly maintenance (async CronJob; MVCC ensures foreground queries are unaffected)
def skill_weekly_maintenance_compact():
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    table.compact_files()
    table.cleanup_old_versions(older_than_days=7)
```

**Key points**:
- **Lazy Import**: do not import `polars`/`lancedb` at the top level; cold start drops from ~500ms to <10ms
- **Zero-Copy Query**: `table.to_lance().to_lazy()` creates a Polars LazyFrame without downloading data
- **Blind Append**: concurrent writes create independent physical fragments, avoiding lock contention
- **Maintenance**: run compaction asynchronously (CronJob); LanceDB's MVCC ensures foreground queries are unaffected

### Pattern Three: Iceberg + Catalog (High-Frequency Updates)

The right answer for high-frequency update/delete scenarios. MoR appends a few-KB Delete File, completing writes in milliseconds. The REST Catalog provides an efficient snapshot index.

**Deployment roadmap:**
1. Get access to the OSS Tables preview → create a test table bucket → simulate high-frequency updates in Python, writing MoR-format Iceberg
2. Read via REST Catalog with DuckDB → verify whether the full-table scan goes from 50s → sub-second after frequent updates
3. Rebuild the production write path as Iceberg appends + enable cloud auto-compaction

**Key insights:**
- MoR is "cheap on write, slightly more expensive on read but smoothed out by background governance" — a huge net gain for high-frequency update scenarios
- The REST Catalog is the standardization direction for Iceberg — no longer dependent on Hive Metastore / Glue / a self-built lock service
- OSS Tables ≈ S3 Tables — the object storage bucket has a built-in Iceberg REST Catalog + Auto-Compaction

---

## 7. Alibaba Cloud Compatibility Issues

### The Root Cause of OSS Not Supporting Delta Lake

`deltalake` 1.6 hardcodes the `If-None-Match` HTTP header when writing the transaction log. Alibaba Cloud OSS does not support this header (it returns 400 `NotImplemented`), and Rust's `object_store` depends on the full S3 API, with no fallback path.

OSS has its own conditional-write header, `x-oss-forbid-overwrite`, but `object_store` does not recognize it.

**In essence**: Alibaba Cloud has a complete lakehouse solution (DLF + MaxCompute + Hologres), and making OSS perfectly compatible with Delta Lake would mean users could freely switch compute engines and Alibaba Cloud would be reduced to "selling hard drives." Maintaining edge-API differences is an invisible vendor lock-in.

**Workaround** (single-writer scenario):

```python
storage_options = {
    "aws_access_key_id": "YOUR_ACCESS_KEY",
    "aws_secret_access_key": "YOUR_SECRET_KEY",
    "aws_endpoint": "oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou",
    "aws_unsafe_disable_conditional_write": "true",
    "aws_virtual_hosted_style_tables": "false",
    "aws_skip_signature_v4_cluster": "true",
}
```

After disabling conditional writes, concurrent writes to the same Delta table may conflict. For multi-writer scenarios, use native Parquet + DuckDB or switch to Iceberg.

### Alternatives

1. **Self-hosted MinIO**: 100% S3 compatible, Delta Lake works out of the box
2. **Native Parquet + DuckDB**: bypass the Delta transaction log and manage data with filesystem semantics
3. **Iceberg**: support via the DLF Catalog is somewhat better, but the Rust implementation may face the same `object_store` problem

---

## 8. Migration Path: Delta → Iceberg

> In the high-frequency point-update scenario of the order system, the root cause of the 50s query latency caused by Delta Lake's CoW mode, and the migration plan.

### Root Cause: The Two Fatal Flaws of Delta CoW

- **Write amplification**: update 1 row → read in the entire 200MB file → rewrite a new 200MB file
- **Fragment file explosion**: frequent rewrites leave OSS with a backlog of large numbers of unordered small files + an extremely long metadata log. A DuckDB full-table scan is forced to issue 60+ serial HTTP request downloads, and the vast majority of the 50-second latency is spent on establishing network handshakes and parsing massive amounts of metadata

### Migration Plan

- **MoR cures write amplification at the root**: updates do not touch the original file, appending only a few-KB Positional Delete File + one new record
- **REST Catalog eliminates the metadata bottleneck**: the query engine requests the current snapshot directly from the REST API
- **Auto-Compaction delegated to the cloud**: merges are scheduled automatically in the background, so queries always hit clean large files

**Note on the cost of MoR**: reads must merge the original file + the Delete patch, adding query overhead. But Auto-Compaction continuously merges the patches back into the main file in the background, ensuring queries always hit clean large files. MoR is "cheap on write, slightly more expensive on read but smoothed out by background governance" — a huge net gain for high-frequency update scenarios.

**Migration roadmap:**
1. **Test validation**: get access to the OSS Tables preview → create a test table bucket → simulate high-frequency order updates in Python, writing MoR-format Iceberg
2. **Benchmark confrontation**: read the table bucket via REST Catalog with DuckDB → verify whether the full-table scan aggregation goes from 50s → sub-second after frequent updates
3. **Formal cutover**: rebuild the production write path as Iceberg appends + enable cloud auto-compaction

### When Not to Migrate

| Scenario | Choose Delta | Choose Iceberg |
|:---|:---|:---|
| Time-series logs / event streams (append-only, no updates) | ✅ Good enough | Over-engineering |
| Batch ETL, loaded once and then read-only | ✅ Simple | Little difference |
| IoT sensor data (high-frequency writes, rare updates) | ✅ Well suited | Optional but unnecessary |
| Order/user state (high-frequency updates + deletes) | ❌ CoW is fatal | ✅ The right answer |
| Audit scenarios with time travel + row-level deletes | ❌ Slow log chain | ✅ Snapshot isolation |

**Judgment principle**: if data is basically never modified after being written → Delta is simpler. If updates/deletes are the norm → only then is Iceberg worth it.

---

## Cross-References

- **[Unified Data Layer](unified-data-layer-en.md)**: SurrealDB's compute-pushdown principle and the consensus protocol of object storage are fundamentally of the same origin — reduce unnecessary transfers and let logic execute where the data lives.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: an architectural-critique methodology — examining technology choices from the perspective of "what does the architecture encourage," rather than "the tool is fine, the developer used it wrong."
- **[Flux Architecture](flux-architecture-en.md)**: The lakehouse's Catalog layer manages data versions and snapshots, consistent with the version-number interface idea of reactive architecture — elevating version governance from the storage layer to the control plane.
