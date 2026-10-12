# HelixDB vs LanceDB: Two Routes for an AI Data Stack on Object Storage

> **Languages:** [English](helixdb-vs-lancedb-en.md) (primary) · [中文](helixdb-vs-lancedb.md)

**Status:** Research analysis
**Date:** 2026-08-14
**Core question:** Both belong to the "object storage + Arrow" AI data stack, yet HelixDB and LanceDB have chosen almost opposite implementation routes — one takes the graph-vector service + proprietary DSL path, the other takes the embedded multimodal retrieval engine + vector/full-text/SQL path. This article reconciles the two dimension by dimension, and, against the positions established in existing documents (against proprietary syntax, against open-core feature gating, object-storage-native), gives a judgment on the trade-offs.

---

## 0. Three-Layer Stack: First Align the Subjects of Comparison

The direct competition between HelixDB and LanceDB is often blurred by the foundation "sandwiched in between". The fact is:

```
                            AI Data Stack (Object Storage + Arrow Ecosystem)                           
   ┌────────────────────────────────────────────────────┬─────────────────────────────────────────────┐
   │   HelixDB (graph + vector service layer)           │   LanceDB (embedded retrieval layer)        │
   │   Data model: graph + vector + KV                  │   Data model: tables + multimodal + vector  │
   │   Query: proprietary graph traversal DSL (no SQL)  │   Query: vector + full-text + SQL           │
   │   Deployment: service/container or Cloud           │   Deployment: embedded or Cloud             │
   ├────────────────────────────────────────────────────┴─────────────────────────────────────────────┤
   │   Storage foundation                               │                                             │
   │   SlateDB (object-store LSM, Helix's own fork)     │   Lance columnar format (in-house)          │
   └──────────────────────────────────────────────────────────────────────────────────────────────────┘
                        Common foundation: Apache Arrow / Parquet / object_store                       
```

Key layering judgment: **SlateDB is HelixDB's foundation, not LanceDB's rival**. The comparison of SlateDB and Fjall, "local compute-storage co-location vs S3 compute-storage separation", has already been completed in [KV Storage Engine](kv-storage-engine-en.md) and [Consensus Protocol](consensus-protocol-en.md). This article addresses only the topmost layer: **HelixDB vs LanceDB**.

---

## 1. Comparison of Core Dimensions

| Dimension | LanceDB (reference) | HelixDB |
|:--|:--|:--|
| **Positioning** | Multimodal AI embedded retrieval library ("Search More, Manage Less") | OLTP graph-vector database (AI memory / knowledge graph) |
| **Data model** | Tables + multimodal columns + vectors | Graph (nodes/edges) + vectors + KV + documents |
| **Query interface** | **Vector similarity + full-text search (BM25) + SQL (DuckDB)** | **Proprietary graph traversal DSL only (no SQL)** |
| **Deployment form** | Embedded (Python/TS/Rust SDK) + Cloud | Standalone service/container + Cloud |
| **Storage foundation** | In-house Lance columnar format | SlateDB (Helix's own forked materialization, object-store LSM) |
| **Concurrency model** | Multiple readers and writers + automatic versioning | Single writer + multiple readers (Writer/Reader) |
| **Transactions** | Atomic writes / versioning (relies on object storage atomicity) | Full ACID (Cloud; for local, topology-dependent) |
| **Maturity** | ~11k★, mature ecosystem | ~5.7k★, v3.x, evolving rapidly |
| **Licensing** | Open source + commercial cloud | Apache-2.0 core + closed-source hyperscale/gateway |

---

## 2. The Decisive Factor in the Query Interface: SQL vs Proprietary DSL

This is the core dividing line between the two, and it relates directly to your consistent technology-selection position (against proprietary syntax IDL).

**LanceDB: vector + full-text + SQL, all three.**
- Vector similarity goes through the SDK: `table.search(vec).limit(20)`
- Full-text search supports BM25
- **SQL queries** (DuckDB integration) for structured filtering/aggregation
- The three query types are complementary on the same data, and ecosystem tools (Pandas, Polars, DuckDB, LangChain, LlamaIndex) can plug in directly

**HelixDB: only a proprietary graph traversal DSL.**
- Queries are written as `#[query]` functions (Rust DSL) or TS/Go/Python builders → compiled into a **JSON AST** → `POST /v2/query`
- **No SQL, no GQL, no Cypher**
- Worse still: HelixDB once had a complete query language, **HelixQL v1** (type-safe, Gremlin/Cypher-like), which was **deprecated in v2**, with the Rust DSL pushed instead

**The unbalanced demerits** (HelixDB has all three, LanceDB none):
1. **Learning curve**: the graph traversal DSL is proprietary syntax, and every developer has to learn it from scratch
2. **Ecosystem fragmentation**: the entire SQL ecosystem toolchain (BI, analytics, existing integrations) is unusable; the JSON AST transport is not a standard protocol either
3. **A history of self-abandoned language**: HelixQL v1 (a complete declarative query language) was abandoned, suggesting that the query layer's direction is unsettled and carries rewrite risk

This is precisely the empirical evidence for the "resist proprietary syntax IDL" position: when a database does not even provide a standard query language, it locks developers into its own custom worldview.

---

## 3. Scenarios Each Excels At

| Scenario | LanceDB | HelixDB |
|:--|:--|:--|
| Multimodal lakehouse retrieval (text/image/point cloud + vector) | ✅ Main battlefield | Partial (vector support) |
| RAG hybrid vector + full-text retrieval | ✅ Strong | Strong (vector + full-text) |
| Knowledge graph: node/edge traversal, graph analytics | ❌ None | ✅ Main battlefield |
| AI memory: relational chained context | Weak (table model) | ✅ Strong (graph model) |
| Structured SQL analysis (BI, aggregation) | ✅ Strong (DuckDB SQL) | ❌ Missing (DSL only) |

**Verdict**: LanceDB wins on full-spectrum coverage of "retrieval + analytics" (vector + full-text + SQL) — stable, mature, embedded-friendly. HelixDB wins in the narrow scenario of "graph model + transactions" (knowledge graph, relational memory), but at the cost of fixed syntax and ecosystem fragmentation.

---

## 4. Object Storage Fit with S3 (Shared Foundation)

Both HelixDB and LanceDB depend on being object-storage-native, which contrasts with Fjall's "local compute-storage":

| Dimension | LanceDB | HelixDB |
|:--|:--|:--|
| Storage medium | S3/OSS/local (object_store abstraction) | S3/local MinIO/in-memory (SlateDB ObjectStore) |
| Atomic writes | Relies on object storage `If-None-Match` conditional writes | SlateDB object-store LSM |
| Foundation relationship | In-house Lance format | SlateDB (same origin as the underlying layer compared in your KV document) |

**The argument that "Fjall + consensus replication is unsuitable for S3"** has already been fully developed in the "What Raft Is Not Suitable For" section of [Consensus Protocol](consensus-protocol-en.md) (write amplification, Raft replication cost vs S3's fixed three replicas, compute-storage separation), and is not repeated here — the two follow-up options (LanceDB, HelixDB) are both positive solutions that are object-storage-native.

Multi-vendor object storage adaptation (the `If-None-Match` compatibility matrix for Alibaba Cloud OSS/Huawei Cloud OBS/Tencent Cloud COS/Cloudflare R2) is covered in the appendix.

---

## 5. Licensing and Maturity: Two Hidden Risks

### Licensing Layers (HelixDB)
- **Open-source core**: `HelixDB/helix-db` = Apache-2.0; `crates/db` (the database kernel), planner, server, and graph-algorithms are all open source and self-hostable, with no "slow query log = enterprise-only" style feature gating. The forked SlateDB is also Apache-2.0.
- **Closed-source platform**: `helix-hyperscale` (the core of the hyperscale multi-tenant gateway) and `helix-gateway` are not public. That is, Cloud's full topological capability is opaque and cannot be self-hosted.

Compared with your frame of reference: **this differs from SurrealDB locking core features (such as slow query logs) into the enterprise edition** — HelixDB's core can be self-hosted. But the "open-source core + closed-source hyperscale platform" hybrid remains an opaque item for large deployments.

### Maturity (HelixDB, the main risk)
- v3.x (2026-08), **rewritten from scratch, evolving rapidly**
- Doing **OLTP graph transactions + single-writer multi-reader** on object storage — the credibility of this combination itself remains to be verified
- The query language underwent a **self-abandoning rewrite** from HelixQL v1 → Rust DSL, with the direction unsettled

Combining the two: HelixDB is an early-stage choice whose "capability surface may align but whose ecosystem is unsettled + platform is closed-source"; LanceDB is the steady choice with "full capability-spectrum coverage + mature ecosystem".

---

## 6. Verdict

- **Choose LanceDB by default**: multimodal lakehouse retrieval, RAG, hybrid vector/full-text, SQL analytics — it covers them all, and its embedded SDK is simple to deploy with a mature ecosystem. The vast majority of AI data requirements fall within its capability spectrum.
- **Consider HelixDB only when "graph model + transactions" are needed**: knowledge graph traversal, relational chained AI memory. But you must accept: no SQL, proprietary DSL, closed-source platform, extremely early stage.
- Whichever you choose, **both SlateDB / Lance belong to the object-storage-native route** — the comparison with Fjall's local compute-storage has already been settled in the KV/consensus documents, and what is here is the upstream application-layer implementation of that judgment.

---

## Appendix: Multi-Vendor Object Storage Adaptation Matrix (Provider Quirks)

The atomic writes of LanceDB and Delta Lake rely on the standard AWS S3 `If-None-Match: *` conditional request header. Non-AWS vendors (Alibaba Cloud OSS, Huawei Cloud OBS, etc.) implement the protocol with differences, commonly returning 400/412. The Arrow ecosystem's `object_store` abstraction performs client-side adaptation through the `aws_copy_if_not_exists` parameter in `storage_options`:

| Cloud vendor | Status | `aws_copy_if_not_exists` | Conflict mechanism |
|:--|:--|:--|:--|
| Alibaba Cloud OSS | Does not support standard S3 headers | `header-with-status:x-oss-forbid-overwrite:true:409` | Hardware atomic lock; returns 409 if already present |
| Huawei Cloud OBS | Does not support standard S3 headers | `header-with-status:x-obs-forbid-overwrite:true:409` | Hardware atomic lock |
| Tencent Cloud COS | Supported for new buckets; occasional 400 for old/overseas buckets | None | Software-level transactional degradation |
| Google GCS | S3-compatible conditional writes unstable | Switch to the `gs://` protocol | GCS-native generation ID |
| Cloudflare R2 | Natively supported | No configuration needed | Natively aligned |
| MinIO / RustFS | Natively supported | No configuration needed | Natively aligned |

Key parameters force path-style routing: `aws_s3_addressing_style: "path"` + `aws_virtual_hosted_style_request: "false"`.

---

## Cross-References

- **[KV Storage Engine](kv-storage-engine-en.md)**: the primitive-layer comparison of Fjall's local compute-storage co-location vs SlateDB's object storage, which is the underlying basis for this article's application-layer decision.
- **[Consensus Protocol](consensus-protocol-en.md)**: "What Raft Is Not Suitable For" and "Fjall + Raft vs SlateDB + S3" — why S3 is the correct object-storage-native foundation.
- **[Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md)**: HelixDB's single writer/multiple readers and object storage atomic writes are precisely the architectural embodiment of avoiding distributed locks.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: why the caching/coordination layer rules out Redis-style single-point memory.
- **[Serialization Protocol Comparison](serialization-protocol-comparison-en.md)**: the position against proprietary syntax IDL is the basis for this article's critique of HelixDB's proprietary DSL.

**The unified first principle**: do not engage in technology worship, do not swallow the pie that open source paints, look only at real physical limits, licensing transparency, and team productivity.
