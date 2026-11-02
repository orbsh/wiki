# KV Storage Engine: Architecture, Composite Key Encoding, and Use Cases

> **Languages:** [English](kv-storage-engine-en.md) (primary) · [中文](kv-storage-engine.md)
> **Implementation:** [OKM](https://github.com/orbsh/okm) — Rust KV crate with declarative derives (`KeyEncode` / `RowEncode` / `EdgeEncode`)

**Status:** Continuously evolving
**Engines covered:** Fjall (local NVMe), SlateDB (S3 cloud-native), redb (local B-tree), SQLite comparison
**Architecture:** [Aura Architecture §5](aura-architecture-en.md) — dual-engine mode (Fjall / SlateDB+S3)
**Cross-ref:** [Redis Critique](redis-critique-en.md) — why Redis is being replaced by KV

## Core Thesis

KV covers the vast majority of storage needs with a minimal substrate, resting on three independently valid judgments:

1. **Zero network overhead in-process**: Embedded KV (Fjall et al.) runs directly inside the application process; local reads involve no network hops — it makes "network calls to access data" an avoidable architectural burden:
   - **Co-located reads with the process**: no network hops, no remote round-trips.
   - **Eliminates RTT and serialization overhead**: local get/put go straight through, skipping remote transport and encode/decode.
   - **Contrast with Redis's "networked RAM"**: data lives remotely, latency dominated by RTT (see [Redis Critique](redis-critique-en.md)).
2. **A minimal substrate covering many models**: KV only has get/put/scan, yet can express Hash, ZSET, graphs, vectors, full-text, and every other structure on top of it — **one substrate covering many models**:
   - **Complexity converges in the encoding layer** rather than the engine layer; the engine stays minimal get/put/scan (see [Composite Key Encoding](#composite-key-encoding-redis-data-structures--kv)).
   - **Structure is a byte-order projection of composite keys**: Hash fields, ZSET score prefixes, graph relations, vectors, and full-text inverted entries are all encoded into the key (see [Multi-Model Capability on Top of KV](#multi-model-capability-on-top-of-kv-vector-search-full-text-graph-queries)).
3. **From expert-only to AI-usable**: KV encapsulates storage complexity, turning storage modeling from an "expert-only" discipline into something "AI can participate in":
   - **AI handles the glue code**: key encoding, serialization, and other mechanical parts are well within AI's competence.
   - **Humans own the judgment**: architecture design, review, and environment-specific production operations.

## The Basic KV API

The KV storage engine's interface is minimal — only four operations:

```
put(key, value)              ← create/update (create if key absent, overwrite if present)
get(key)                     ← read (point lookup, O(log N))
delete(key)                  ← delete (tombstone; space reclaimed at Compaction)
scan(prefix) / range(a, b)   ← prefix scan / range scan (ordered iteration)
```

**The value is an opaque byte blob**. The engine doesn't care what format it is, what fields it contains, or how it's serialized. All operations inside the value (reading fields, modifying fields, appending data) happen at the application layer.

Correspondence with SQL:

| SQL | KV equivalent |
|:---|:---|
| `INSERT INTO t VALUES (...)` | `put(key, value)` |
| `SELECT col FROM t WHERE id=1` | `get(key)` → application-layer deserialization → extract field |
| `DELETE FROM t WHERE id=1` | `delete(key)` |
| `UPDATE t SET col=col+1 WHERE id=1` | `get` → application-layer modification → `put` (**not atomic**) |
| `SELECT * FROM t WHERE name='alice'` | No equivalent (requires secondary-index keys) |
| `JSON_SET(col, '$.field', val)` | No equivalent (value is bytes; no awareness of internal structure) |

**KV has no value-internal operations**. SQL has `UPDATE SET col=col+1`, `JSON_SET`, `ARRAY_APPEND` — KV has none of them. This is KV's minimalist philosophy: the engine only stores and retrieves bytes; all value operations live in your code.

**The only atomicity guarantees**: some engines provide key-level atomic operations:

| Engine | Atomic operation | Granularity |
|:---|:---|:---|
| **Fjall** | Atomic batch writes + transactions (MVCC optimistic locking) | Multiple keys committed at once |
| **RocksDB** | `merge` (custom merge function, e.g. counter increments) | Single key |
| **etcd** | `compare_and_swap` (CAS, conditional update) | Single key |

These are **key-level atomicity**, not value-internal operations. "Add a field", "remove a field", "add an index" are all application-layer logic — KV only stores and retrieves bytes.

### Complexity and Physical I/O

#### get(key): O(log N), same order as B-Tree, different physical path

| | B-Tree | LSM-Tree |
|:---|:---|:---|
| Lookup path | root → leaf, fixed 3-4 levels (B=256, N=1 billion) | MemTable → L0 → L1 → ... → Ln |
| Per-level work | Binary search for the key within a page | Binary search of the SSTable index + Bloom Filter filtering |
| Disk I/O | 3-4 reads (one page read per level) | 1-3 reads (Bloom Filter skips most SSTables) |
| Determinism | High (fixed path) | Low (depends on Bloom Filter hit rate) |

B-Tree's O(log_B N) is exact — the root-to-leaf path length is fixed. LSM-Tree's O(log N) is approximate — it depends on the number of levels and Bloom Filter effectiveness. For positive lookups (key exists), LSM-Tree is usually faster; for negative lookups (key absent), B-Tree is more deterministic. Both are O(log N), but B-Tree's constant is more predictable, while LSM-Tree's constant is usually smaller (for read workloads).

#### scan(prefix): O(log N + K), locate first, then iterate sequentially

```
scan("inv:hello:")
│
├── 1. Locate the starting position: O(log N)
│   Find the first key ≥ "inv:hello:" in the MemTable + SSTables of each level
│   (binary search within each SSTable's index)
│
└── 2. Sequential iteration: O(K)
    K = number of keys matching the prefix
    The MergeIterator merges the ordered streams of multiple SSTables, taking the smallest key on each advance
    until keys no longer match the prefix
```

Iteration is **sequential I/O**, not random I/O. SSTables are ordered; a scan just moves a pointer forward over an ordered stream. Sequential reads are 10-100x faster than random reads (a huge gap on HDDs, several times on SSDs).

#### Operation complexity summary

| Operation | Complexity | Physical I/O |
|:---|:---|:---|
| `get(key)` | O(log N) | 1-3 random reads |
| `scan(prefix)` locate | O(log N) | 1-3 random reads |
| `scan(prefix)` iterate | O(K) | K sequential reads |
| `put(key, value)` | O(log N) | Write to MemTable (memory); sequential SSTable writes at flush |
| `delete(key)` | O(log N) | Write a tombstone marker; space reclaimed at Compaction |

### SQL Translation Layer vs KV Pipeline Chain: A Decisive Advantage Under Predictable Query Patterns

For framework platforms (API gateways, agent executors, 3D pipelines), the data access path is fixed at design time. Dropping SQL is not for convenience — it eliminates the database optimizer (Query Planner) as a runtime black box, yielding 100% physical performance determinism.

#### Concrete comparison: fetching the 10 most recent conversation memories

**SQL approach** (PostgreSQL / SurrealDB):

```sql
SELECT message_id, content FROM agent_memories
WHERE session_id = 'session_456'
ORDER BY timestamp DESC LIMIT 10;
```

Underlying physical cost: lexical and syntactic parsing → AST generation → logical execution plan → the optimizer guesses index scan vs. full table scan → frequent jumps between B-Tree nodes.

**KV approach** (Rust + embedded KV):

At write time the key is already arranged in reverse physical order: `m:{session_id}:{u64_max - timestamp}:{message_id}`. The query takes just one Rust pipeline chain:

```rust
// Prefix locate → walk back 10 steps, done
let prefix = format!("m:session_456:").into_bytes();
let top_10 = kv_engine
    .scan(&prefix)         // locate the starting range (O(log N) in-memory binary search)
    .take(10)              // sequentially read 10 records (O(1) disk/memory sequential I/O)
    .collect::<Vec<_>>();
```

**Why KV wins**: the Rust method chain is more concise than SQL's declarative boilerplate (SELECT/FROM/WHERE/ORDER BY/LIMIT). No black-box optimizer being clever — the code *is* the execution path. From 10MB to 10TB of data, execution efficiency is unchanged and sub-millisecond response is unshakable.

#### Two often-overlooked costs: result-set caching and permission checks

Beyond parsing/optimization, SQL carries two "hidden costs" that fall exactly outside KV's blind spots:

- **Temporary result-set caching**: a relational query must first materialize results into a result set on the engine side (intermediate table + return buffer), then serialize and return it over the protocol. KV is an in-process `scan` delivered directly — the bytes read never become a transient copy of a SQL result set; they go straight to the deserializer, reusing business memory (the in-process zero-serialization philosophy). At low concurrency this is just a spike of a few KB, but on the agent hot path assembling tens of thousands of tokens of context per round, repeated materialization + copying is a real cost.
- **Per-query permission checks**: after parsing, a relational database also performs an authorization decision per SQL statement (roles / row-level security / ACL table lookups). Embedded KV has no separate authentication layer — who can access which prefix is a policy of the application process embedding it; the engine doesn't check permissions. What's saved is not just a table lookup but the relocation of "who can read what" from the database runtime back to the application layer: permissions become ordinary code you can review, rather than an implicit black box inside the engine.

**Composability**: SQL's compute space is an island — inserting an ML model prediction or a graph algorithm in the middle of a computation is impossible at the language level; you can only pull the data back to the application layer. KV pipeline chains are ordinary code in the host language, with control flow and data flow interwoven: `if/else` orchestrates different scan chains, intermediate results bind to variables, breakpoints can be inserted at any time. Data never has to leave the process to connect with any algorithm library in the ecosystem.

#### Production-grade components: atomic dual-write + timeline index

Demonstrating how a single atomic batch writes the primary data while automatically building a reverse-chronological secondary index, ensuring no data drift between the main table and the index table:

```rust
use bytes::Bytes;
use std::sync::Arc;

/// Atomic batch (equivalent to Fjall WriteBatch / SlateDB's batch API)
pub struct TransactionBatch {
    pub actions: Vec<(Vec<u8>, Option<Bytes>)>,
}

impl TransactionBatch {
    pub fn put(&mut self, k: &[u8], v: &[u8]) {
        self.actions.push((k.to_vec(), Some(Bytes::copy_from_slice(v))));
    }
    pub fn delete(&mut self, k: &[u8]) {
        self.actions.push((k.to_vec(), None));
    }
}

pub struct FrameworkStorage {
    /// Global embedded storage engine shared across Tokio's multithreaded runtime
    /// In production, replace with Arc<fjall::Keyspace> or Arc<slatedb::Db>
    pub engine: Arc<String>,
}

impl FrameworkStorage {
    /// Atomic dual-write: save agent memory + automatically build the reverse-chronological index
    pub fn save_agent_memory(
        &self,
        session_id: &str,
        message_id: &str,
        timestamp: u64,
        payload: &[u8],
    ) -> TransactionBatch {
        let mut batch = TransactionBatch { actions: Vec::new() };

        // 1. Primary table: full data
        let data_key = format!("data:session:{}:msg:{}", session_id, message_id).into_bytes();
        batch.put(&data_key, payload);

        // 2. Timeline index: reverse order (u64::MAX - timestamp)
        let inverted_time = u64::MAX - timestamp;
        let mut index_key = Vec::new();
        index_key.extend_from_slice(format!("idx:time:session:{}:", session_id).as_bytes());
        index_key.extend_from_slice(&inverted_time.to_be_bytes());
        index_key.extend_from_slice(format!(":{}", message_id).as_bytes());

        // Value is empty — the entity ID is already encoded in the key's lexicographic order
        batch.put(&index_key, &[]);

        batch
    }
}
```

The primary table stores the full data; the index table stores only an empty value (the entity ID is already encoded in the key via its binary-sortable order). One atomic commit — both keys succeed or fail together. SQL's `INSERT INTO` cannot write to two tables in a single statement with atomicity guaranteed — that requires an additional transaction wrapper.

### Prefix Scan vs SQL: The Promise and Its Price

A prefix scan is less flexible than SQL and less abstract. That is simply true, and it is worth laying out what it costs and what it buys.

SQL sits at a higher level of abstraction: write the predicate, and which index serves it, whether a covering index is used, how joins are ordered — all the optimizer's job. The trouble is that this promise is often unkept, and in the worst possible way: not with an error, but by quietly making physical layout part of the query. Every implementation whose index depends on key order (MySQL's composite indexes being the canonical case) runs into the left-prefix rule: whether `INDEX(a, b, c)` can serve `WHERE b = 1` depends on the field order chosen at create-table time. The developer is thus forced to maintain a mental picture of a B+ tree while writing declarative queries, and field order ends up decided by query usability rather than semantics. That is an abstraction leak, not abstraction (see [MySQL Critique](mysql-critique-en.md), "Index Rigidity").

A prefix scan puts the same fact on the table: key layout IS the access path, read order IS storage order, and the probe IS the WHERE clause. It never promises to choose a path for you, so no promise can be broken.

The costs are three real constraints:

*   **Flexibility**: predicates cannot be combined freely. Adding a `WHERE` clause in SQL costs one edit; changing an access dimension here costs a declaration change, a rebuild of the face, and a data migration.
*   **Abstraction**: the developer has to know byte order, field widths, and the limits on variable-length segments — has to know that `[org][user][device]` serves "all users of an org" well and "which org does this device belong to" not at all.
*   **Evolution**: query patterns are settled at modeling time; a dimension that arrives late is served by a new face, or by re-laying-out the key.

The payoff is three items, one per cost: the read path is predictable (no black-box optimizer, no stale-statistics surprises at runtime); performance is reasoned about rather than hoped for (a prefix hit is a range hit, and a scan's cost is the range size times the per-entry cost); and the semantics are honest (there is no "reads nicely, scans the whole table" gap).

In one sentence: SQL uses abstraction to cover the optimizer's competence boundary; a prefix scan publishes that boundary and hands the decision to the developer. Choosing the second road means admitting that automatic optimization is genuinely hard — and only after admitting it can a model deliver value where it actually excels (settling access patterns at modeling time) instead of promising what it cannot do.

### Composite Key Encoding: Redis Data Structures → KV

A pure KV engine has no Hash, ZSET, or other primitives. Every data structure the frontend needs is uniformly simulated via key encoding + prefix/range scans (the table below gives the generic mapping using Redis data structures as reference):

| Redis | KV Pattern | Read | Write |
|-------|--------------|------|-------|
| `STRING` | `str:<key>` | `get` | `put` |
| `HASH` | `hash:<key>:<field>` | `get` / `prefix` | `put` / `remove` |
| `LIST` | `list:<key>:<seq>` (8-digit zero-padded) | `range` | `put` + monotonic seq |
| `SET` | `set:<key>:<member>` → `""` | existence check | `put` / `remove` |
| `ZSET` | `zset:<key>:<score>:<member>` | `range` (score interval) | `put` / `remove` |

Key details: LIST requires atomic sequence number generation → a monotonic counter inside the engine. ZSET score encoding uses a zero-padded fixed-width format to guarantee correct lexicographic order.

A pure KV engine has no Hash, ZSET, or other primitives. Every data structure is simulated on byte order via key-space encoding (Composite Key Encoding) — the LSM-Tree's iterator is naturally byte-ordered, which means that as long as the key encoding is designed correctly, range scans and sorting are achieved at zero cost in the storage layer. Redis's abstraction at this layer is **extremely thin**: Hash is just a second-level key space (key concatenation), and ZSET merely atomizes the "read → compute → write" sequence on the server side — and atomizing read-modify-write is exactly the existing capability of KV `Batch`/CAS; see [Redis Critique](redis-critique-en.md).

#### Physical encoding of Hash

The Redis Hash `HSET user:101 name "Alice"` splits into two independent KV pairs:

```
Key: "hash:user:101:name"  →  Value: "Alice"       (string)
Key: "hash:user:101:age"   →  Value: [0x00000019]   (i64 big-endian, 8 bytes)
```

**Prefix-scan bulk deletion**: `DEL user:101` doesn't need to read first and delete field by field. Use `prefix("hash:user:101:")` directly to locate all field keys and issue Delete commands in bulk. Deletion in an LSM-Tree is fundamentally writing a tombstone marker — an O(1) operation. Redis's large-Hash deletion (at the million-field scale) blocks the single-threaded event loop for tens of milliseconds; LSM-Tree engines have no such problem — multithreaded background compaction reclaims tombstones asynchronously without blocking foreground requests.

#### Physical encoding of ZSET: score as a reverse index in the key prefix

ZSET's core constraint is "sorted by score". Leveraging the LSM-Tree's byte-order sorting property, embed the score in the key prefix:

```
Player 99 scores 100, player 88 scores 99

Primary data table:
  Key: "data:player:88"  →  Value: {...}  (player struct)
  Key: "data:player:99"  →  Value: {...}

Ranking index (dual-write):
  Key: "rank:scores:[0x0000000000000064]:99"  →  Value: ""  (score=100, big-endian)
  Key: "rank:scores:[0x0000000000000063]:88"  →  Value: ""  (score=99,  big-endian)
```

The score is converted to a fixed 8-byte big-endian byte array (`i64::to_be_bytes()`), ensuring strict consistency between numeric magnitude and lexicographic byte order. Fetching the top 10: the iterator `Seek`s to the `"rank:scores:"` prefix and then traverses backward 10 times. No in-memory sorting needed; the storage layer guarantees the order automatically.

#### Dual-write atomicity: updating a ZSET score

Updating a ZSET score involves two steps: deleting the old score key + writing the new score key. These are two distinct KV operations that must be completed within the same atomic batch:

```rust
let mut batch = keyspace.batch();
batch.delete(old_score_key);   // index of the old score
batch.put(new_score_key, b""); // index of the new score
batch.put(data_key, &updated_player); // primary data table
keyspace.write(batch)?;        // atomic write — all succeed or all fail
```

Without an atomic batch, a crash after only half the writes → dirty data in the ZSET index (a member appearing at two score positions simultaneously, or going missing). The KV engine's `Batch` API guarantees a single atomic commit to the underlying WAL.

In cluster deployments, the batch's atomicity is guaranteed by the consensus layer — see the §Distributed KV section for the principle; the batch's atomicity extends from single node to cluster.

## Design Patterns: System-Level Paradigms on a Pure KV Substrate

In the pure KV world, algorithms are no longer glue code floating outside the database — the key encoding format itself is the index layer, the cache layer, and the isolation layer.

### Access-Pattern-Driven: The Starting Point of Modeling

Above the technique set you need a ground floor: **enumerate all access patterns first, then derive the key layout backwards**. Every access pattern (point lookup of an entity, fetch the most recent N by prefix, scan a field interval, paginate, join via a relation back to the primary table) should land as a point lookup or a prefix scan — a missing pattern is a missing key segment. Predictable query patterns are not just an argument for KV's applicability vs. SQL (see [SQL Translation Layer vs KV Pipeline Chain](#sql-translation-layer-vs-kv-pipeline-chain-a-decisive-advantage-under-predictable-query-patterns)); they are the **input to hand design**: SQL lets you declare "I will query this way"; KV's discipline is to enumerate the access patterns fully first, then ensure every query has a corresponding path in the prefix space.

This discipline, as an executable modeling method — four layers per entity (ns → primary key → sort fields → access methods), mandatory access methods, covering-index restraint, the composite-key boundary — see the [OKM Modeling Guide](https://github.com/orbsh/okm/blob/main/docs/MODELING.md).

### Key-Space Pattern: Logical Encoding Layer vs Physical Partitioning

The key-space pattern is often mistaken as requiring engine-level multi-partition capability, but it is defined at the **logical encoding layer**: encode the namespace into the key (`ns:entity:field` prefix, composite key encoding, a u16 namespace dictionary), independent of the engine's physical layout. Any KV — including flat-key-space engines — supports it at zero cost, because keys are already flat byte strings and the namespace is just a convention you encode into them.

By contrast, **physical partitioning** (Fjall `Partition`, RocksDB Column Family) is a means of **performance and resource isolation** — independent memtable / flush / compaction (controlling write amplification, reclaiming space per namespace, isolating I/O) — not a prerequisite of the key-space pattern. When namespaces are not physically isolated, KV still works fully; it just lacks those optimizations.

The distinction in one sentence: **the key-space pattern is defined at the logical encoding layer; physical partitioning is only an accelerator when isolation is needed.** If you need fine-grained control over write amplification or space reclamation per namespace → choose an engine with partition capability (Fjall); otherwise logical prefix encoding + any engine suffices. Counter-example: sled (0.34.7, 2024-10, unmaintained for roughly two years as of 2026-08) has only logical trees and no physical partitions, yet it is **entirely suitable for the key-space pattern** — the only criterion that actually rules it out of production selection is **maintenance stagnation**, not the lack of partitions.

### Physical Locality: Related Data Adjacent in Byte Order

Range scans are naturally sequential I/O, but "how many bytes get scanned" is determined by the key layout. Physical locality: design data that **will be accessed together** as contiguous bytes — all records under the same prefix are physically adjacent, so a single prefix scan fetches the whole batch, turning random I/O into sequential I/O (10-100x on HDD, several times on SSD; see [Complexity and Physical I/O](#complexity-and-physical-io)). This is Bigtable's founding principle: the row key both defines the logical entity and determines physical placement.

Two sources of locality: prefix aggregation of an entity's fields, and complement inversion putting the newest first on a timeline (see [Physical-Layer Encoding Paradigms](#physical-layer-encoding-paradigms-universal-physical-layer-techniques-for-key-encoding)). The criterion: **what is read together should also be adjacent in byte order** — otherwise scans jump across pages, and prefix/reverse encoding cannot recover the lost locality.

### Key Length Is a Cost Variable

Every additional byte in the key amplifies four things at once: MemTable residency, page cache/index footprint, I/O bytes, and shard space. And since keys are fully resident in memory (MemTable/cache), length optimization is a **constant-factor memory reduction**, not amortization — this is the first iron rule of HBase rowkey design. The high-water judgment: put only "fields required for locating and sorting" in the key; metadata that can be projected later goes in the value rather than being stuffed into the key. For compression techniques, see the numeric namespaces / length prefixes / fixed-width offsets in [Physical-Layer Encoding Paradigms](#physical-layer-encoding-paradigms-universal-physical-layer-techniques-for-key-encoding); what's given here is the unified cost model behind them.

### Index-Only Scan (Index Is the Data)

Leave the secondary index's value empty, with the entity_id encoded at the end of the key. At query time, traversing only the key sequence yields all matching primary key IDs — no need to go back to the primary table to read values — zero disk I/O.

```
Primary data table:
  d:{tenant}:{type}:{id}  →  [full business entity]

Attribute index table:
  i:{tenant}:{type}:{attr}:{value}:{id}  →  ""  (value left empty)

Query "all users with status active":
  Scan("i:1:user:status:active:") → traverse the key sequence → extract the trailing id
  No values read at all — zero lookups back to the primary table
```

**Physical effect**: the prefix scan only touches the LSM-Tree's MemTable + the tiny index-layer SSTables, never the large value files of the data layer. Suitable for high-frequency attribute filtering (gateway route matching, agent status filtering).

### Bitmap Front-Interception (Zero KV Calls on the Hot Path)

Gateways frequently check "is this IP blacklisted" and "is this token valid". Calling `kv.get()` on every request carries hash-lookup overhead even on a cache hit. The solution: keep a Roaring Bitmap or Bloom filter resident in application-layer memory.

```
Write path (when a new IP is banned):
  1. kv.put("blacklist:ip:1.1.1.1", "")   ← persist to disk
  2. bitmap.set(hash("1.1.1.1"))           ← in-memory marker

Read path (gateway interception, zero KV calls):
  Gateway receives a request
    → bitmap.check(hash(ip))
    → miss → pass through directly (million-level QPS, pure CPU bit tests)
    → possible hit → kv.get() for the final authoritative decision
```

**Physical effect**: the hot path (99%+ of normal requests) completes entirely within CPU L1 cache, triggering no KV engine calls whatsoever. Only the rare requests that hit the bitmap descend to the storage layer. Suitable for gateway blacklists/whitelists, rate-limit counters, and token validation.

**Difference from a Bloom filter**: a Bitmap supports exact deletion (`bitmap.unset()`); a Bloom filter can only add, never remove. For frequently changing blacklists use a Bitmap; for add-only token whitelists a Bloom filter is more memory-efficient.

### Application-Layer MVCC (Time Travel Without a Native MVCC Engine)

Fjall/SlateDB do not support native MVCC. A framework team achieves application-layer multi-versioning by encoding version numbers into the key skeleton:

```
Key: data:agent:101:v:[u64::MAX - 1001]  →  memory state v1001
Key: data:agent:101:v:[u64::MAX - 1002]  →  memory state v1002 (newest)
```

- **Regular reads**: `Scan("data:agent:101:v:").next()` → after complement inversion the newest version sorts first; the latest state is obtained in sub-microseconds
- **Time travel**: position the prefix pointer after the target version number, then a forward scan yields the historical version chain. No database snapshot locks needed — a pure key design achieves lock-free history rollback

**Difference from SurrealKV's native MVCC**: SurrealKV's built-in `tx.get_at(key, timestamp)` queries historical versions directly, with no application-layer encoding needed. Fjall/SlateDB require manually encoding version numbers into keys. Different costs, same effect.

### Read-Modify-Write: No KV Engine Provides a Partial-Update Primitive

"Modify one field inside the value" does not exist as an API primitive on any mainstream KV engine — `insert`/`put` is always a wholesale value replacement, including the B-tree engine redb. This is a universal constraint of the KV interface shape, not an LSM-specific problem. The storage structure only affects the physical cost of the "rewrite the whole record" approach:

- **In-place-update B-tree** (InnoDB page-level updates): bytes modified directly on disk, low write amplification
- **redb (COW B-tree)**: changing one field = copying the page path from leaf to root; physically not in-place, so a single write can actually cost more than an LSM append
- **LSM**: naturally append-oriented; old versions are shadowed by level and reclaimed asynchronously by compaction — "rewrite the whole record" is the cheapest

**put needs no prior deletion of the old value** — the new version directly shadows the old: LSM via level shadowing + compaction reclamation, B-tree via in-place overwrite. An explicit `del` is only needed when "the data no longer exists"; "update a value" is always a pure put. (The case requiring explicit deletion of the old key is secondary indexes — see §Secondary Index Updates: the index key encodes a dimension that changes, and without deleting the old key it stays in the wrong place forever.)

High-frequency field-level updates (counters, weight accumulation) should not go through read-modify-write loops; the proper solution is **delta appending**:

```
# Don't touch the value; append an increment record
put  edge:123/read/20260902T1000  → +1
# On read, merge the deltas to get the current count; during off-peak hours batch-fold back into the primary record
```

Purely sequential writes, no read-modify-write races (concurrent deltas never overwrite each other), lock-free write path. This is exactly the application-layer equivalent of RocksDB's **Merge Operator** (`merge(key, delta)` — the engine automatically merges the delta chain and folds it during compaction) — Fjall/SlateDB don't have a built-in merge, so it must be built at the application layer; the pattern is as above. Isomorphic to §Application-Layer MVCC: turn "modify" into "append a new version", except the delta approach appends only the changed fields rather than the whole record, at lower cost.

**Delta is not unconditionally superior — the criterion is the concurrency model, not performance.** Delta and RMW have identical write-path costs on LSM (both are one append write, with old versions shadowed and reclaimed); the difference is in concurrency and the read path:

| | RMW direct write (`get count → put count+1`) | Delta append |
|:--|:--|:--|
| Write path | 1 put (including 1 get) | 1 put |
| Read path | O(1) single-point get | Merge the delta chain; requires a background fold to bring it back to O(1) |
| Concurrency correctness | Two writers with the same get/put lose updates; needs CAS/transactions as a backstop | Naturally lock-free, linear merge |
| Complexity | No extra machinery | Delta chain + background fold task |

Selection criteria:

- **Single writer / access already serialized** (a single agent process, sequential updates within the same WriteBatch) → **RMW direct write**. A hot counter key is almost certainly in the block cache; the get is μs-level — far simpler than maintaining a delta chain + fold machinery
- **Multiple writers concurrently updating the same counter** (multi-process, multiple agent instances sharing an S3 source of truth) → **delta**. RMW needs CAS/transactions as a backstop (most embedded engines lack single-key CAS); delta eliminates the entire concurrency problem
- **Extremely high update frequency** (multiple times per request on the hot path) → **delta**. Eliminates each get round-trip; batched folds amortize the cost

A note on where the fold lands: the merged result should be put into a separate counter key (the value is a single integer — a whole-value replacement, writing the new value directly with no RMW), not folded back into a msgpack field of the primary record — that's another read-modify-write loop, which merely moves the pattern you're trying to eliminate from the hot path to the background. The problem isn't really eliminated.

### Secondary Index Updates: Whether the Mutable Dimension Enters the Key

When an index key encodes a "mutable dimension" (e.g., weight `W:<weight>:<edge_id>`), a change in that dimension requires **deleting the old index key + writing the new index key**, and both steps must be committed atomically in the same WriteBatch:

```
put  W:<new_weight>:<edge_id>  → ""
del  W:<old_weight>:<edge_id>      # writes a tombstone on LSM — deletion is also a write
```

Missing the old-key deletion means scans by weight will surface stale entries. Changing a weight once = 2 writes (new index + tombstone); with frequent index updates, tombstone backlog depends on compaction for reclamation. TiDB/SurrealDB's in-transaction index maintenance is the engine-builtin version of this put+delete pair.

Two deletion-free alternative layouts:

- **Change-log style**: `WI:<edge_id>:<ts> → weight`; the index is a record of weight changes, and at query time the current weight of each edge is merged in memory. Purely append, zero tombstones; the cost is an extra merge layer at query time
- **Versioned lazy cleanup**: encode a version number in the key, `W:<weight>:<version>:<edge_id>`; old entries are not deleted and queries skip non-latest versions, with bulk reclamation during off-peak hours

The selection criterion is **the update frequency of the indexed dimension**: low frequency (e.g., memory-edge weights, flushed in batches) uses standard put+delete — tombstone overhead is negligible and the implementation simplest; high-frequency real-time updates switch to the append style.

### Value Packing Granularity: Whole Row vs One Key Per Field

OLTP access patterns are **entity-centric** — "fetch this entity's whole row, modify a few fields, write it back". Therefore the default should be to pack the **entire row** (e.g., postcard serialization) into the value: reading a whole row is 1 point lookup + 1 deserialization — a perfect match.

One key per field is a disaster under OLTP:

| Operation | Whole row packed | One key per field |
|:---|:---|:---|
| Read whole row | 1 point lookup | N point lookups (or 1 scan + reassembly) |
| Modify 3 fields | 1 read + 1 write | N reads first + 3 writes, and cross-key atomicity requires a transaction |
| Write amplification | 1 write per row | 1 write per field, N-fold amplification |
| Single-field point lookup | Read the whole row and deserialize (wasteful) | 1 point lookup (optimal) |

**The key correction**: "one key per field" is **not the cure for DDL pain** — it merely trades DDL's pain for four new pains: N round-trips to read a whole row, scan-based row reassembly, write amplification, and multi-key atomicity. To solve the pain of adding fields, **don't change the storage layout — make the serialization format itself evolvable**. The root cause of whole-row packing being "unfriendly" to DDL is not that the whole-row-packing decision is wrong, but that **a non-evolvable serialization format was used** (postcard is fixed-width, without field tags, weakly self-describing).

**The real DDL solution: evolvable serialization, not a layout change**

1. **Versioned payload (simplest)**: carry a `schema_version` in the value header. Adding a field = a new version; old rows are read per the old version, using lazy migration (upgrade on read/on write). Keys unchanged, zero DDL. Fields remain positional; new fields can only be appended at the tail.
2. **Tagged/TLV encoding (recommended)**: change the payload to `field_id + len + bytes`. Adding a field = adding a new field_id; old readers **skip unknown field_ids** — old rows need no migration and old and new rows coexist. This is the core capability of Cap'n Proto / SBE / FlatBuffers (schema evolution + compatibility; see the "semi-dynamic layer" in [Serialization Protocol Comparison](serialization-protocol-comparison-en.md)). The cost is slightly larger than pure postcard, in exchange for zero-migration field additions.
3. **Hot/cold separation**: core hot fields live in a fixed-width compact region (fixed offsets), while new/cold fields go into a tagged extension region. Reading an old row: hot fields are read directly, and the extension region may be empty. A compromise: zero parsing on the hot path, full compatibility on the evolution path.

**Recommendation**: use 2 as the backbone (evolvable), and add 3 when extreme O(1) single-field reads are needed.

**Field-level auxiliary indexes (introduce based on measurement, not by default)**: if whole-row access dominates but there is an occasional high-frequency single-field hot path (e.g., `get(id) -> name`), you can go dual-track: the primary value = the whole-row blob, plus an auxiliary key = `field_idx:{id} → single field value`. Single-field reads go through the auxiliary key; on write, update the primary row + affected auxiliary keys in sync. But **auxiliary indexes = write amplification** — an optimization, not a default. Only introduce one when a single-field hot path is actually measured; don't pay in advance for an imagined hot path (each added one costs the same as one-key-per-field).

### WiscKey Key-Value Separation (The Write-Amplification Cure for Large Values)

Typical scenario: keys of tens of bytes, values of several MB (conversation history, 3D asset binaries, multimodal feature vectors, raw log payloads). A traditional LSM-Tree rewrites keys and values bundled together during Compaction — write amplification of tens of times.

WiscKey's core: keys and values are physically separated — the index layer stores only a small pointer, and large values are appended to a separate log file.

```
【Memory + SSTable index layer】
 Key: "asset:mesh:uuid_abc" → [File_ID : Offset : Length]  ← a pointer of a dozen-odd bytes
                                    │
                                    ▼ random point lookup on disk
【Separate value log (sequential append)】
 offset_3402 → [large binary asset: 3D model / conversation history / multimodal features]
```

**Physical effects**:
- **Compaction**: only the small pointer keys (a few bytes) are moved; the large value files are untouched. Write amplification drops from tens of times to nearly 1
- **Reads**: once the index locates the pointer, one random disk point lookup (~10μs on NVMe) fetches the large value
- **Applicable scenarios**: any pattern with small keys and large values — agent conversation history, 3D assets, multimodal features, log payloads

Fjall 3.0 natively supports WiscKey (KV separation), and SurrealKV achieves the same effect via Blob Log. SlateDB relies on S3's Range Get to read large values.

### Storing Arrow/Parquet in KV Values? Columnar Storage in Disguise on KV

Driven by the shortcoming that "KV is row storage and cannot project by column", a natural idea is to store Arrow in KV values. But this idea's value depends entirely on **one precondition — granularity**. Arrow is a columnar in-memory format; columnar benefits are only realized when one key corresponds to a **multi-row batch**. Splitting it into two interpretations, the conclusions are entirely different:

**Interpretation A: key → single row (value = one row serialized as Arrow)** — this is just ordinary KV row storage with Arrow as a different serialization format. Columnar benefit ≈ 0; each row is a 1-row batch, and you additionally carry Arrow's schema header. A pure loss.

**Interpretation B: key → columnar batch (value = N rows of Arrow IPC buffer)** — this is the real design. key = segment id, value = one columnar chunk. Reading it into a DataFrame directly yields a columnar memory layout with **zero row-to-column transposition**; within a segment you can compress by column (zstd/lz4), and the schema travels in the buffer itself. But interpretation B is essentially **converting the KV store into a columnar batch store**, at the cost of giving up what makes KV most valuable:

1. **Point lookups (OLTP) are broken** — a logical record is buried inside a batch; you must read the whole segment to get it. KV's original advantage of "embedded, zero-RTT, row-level positioning" is strangled by the batch granularity.
2. **Row-level updates are rewrites** — Arrow buffers are approximately immutable; appending/modifying one row in a batch requires rewriting the whole segment. This is naturally an append-first shape, which fits LSM's append nature, but runs counter to row-level updates.
3. **Cross-segment scans still read everything** — to aggregate a column across all segments, you must range-scan and read the **entire bytes** of every value before interpreting. The KV layer doesn't understand Arrow and cannot push down "project only this column" into storage. The transposition is saved, but reading everything is not.

**Memory model mismatch (the in-process zero-serialization philosophy)**: if the architecture's in-process representation is `ciborium::Value` (zero serialization), then storing Arrow in the storage layer and converting back to ciborium on read just moves the transposition cost to a different place and back. Arrow-as-value only holds if both storage and memory use Arrow (pass-through to DataFrame) — which means the entire data path is bound to Arrow's memory model, abandoning the in-process zero-serialization philosophy. You can only pick one of the two paths.

**Format choice**: if you truly need to store columnar batches, choose **Parquet** rather than Arrow IPC — Arrow IPC is a transport/memory format (oriented toward zero-copy streaming movement), while Parquet is the storage format (columnar + row-group statistics + predicate pushdown). But there's an awkward catch: a standard KV returns the **entire value**, so Parquet's advantage of "skipping within the value by column" is wasted at the single-value level — you're reading the full bytes anyway. This only further shows that at batch granularity the KV store is just "a bucket for bytes"; all the columnar intelligence lives in the value format, and the KV layer itself contributes nothing.

**An honest conclusion**: this is columnar storage in disguise on KV, serving "internal, analytics-dominant, append-first, single-node medium-scale" workloads. If the dominant workload is analytics → go straight to real columnar (Parquet/S3/DuckDB); don't take a detour through KV carrying the baggage of row-level access. If you need row-level access / point updates → don't batch; go back to ordinary row-oriented KV. Arrow-as-value patches the shortcoming of "KV is row storage and can't project columns", but the way it patches it is by turning the store into a columnar batch store — at the cost of losing KV's foundational row-level access, effectively switching lanes.

## Physical-Layer Encoding Paradigms: Universal Physical-Layer Techniques for Key Encoding

These paradigms do not belong to any particular Redis-structure mapping; they are general physical-layer techniques that control key byte order, width, prefixes, and boundaries to exploit the LSM-Tree's lexicographic order and write path.


### Lexicographic Encoding of Numeric Values: Unsigned / Signed / Float / Cross-Type (Overview)

This is the taxonomy of "how a numeric segment inside a key gets encoded into byte order". It splits into four tiers by "domain"; the deeper the tier, the more extra transformation plain big-endian encoding needs to preserve order:

| Domain | Byte-order encoding | Order-preserving? |
|:--|:--|:--|
| **Unsigned non-negative integers** | Fixed-width big-endian, zero-padded | ✅ Lexicographic order = numeric order, zero transformation |
| **Signed integers** | Add a bias, or flip the sign bit + invert remaining bits | ⚠️ Transformation required |
| **Floating-point** | Flip sign bit for positives, invert all bits for negatives (±0 / NaN by separate convention) | ⚠️ Transformation required |
| **Cross-type** | Type tag + payload (FoundationDB tuple style) | ℹ️ Ordered within a tag, with a large ordering across tags |

Most keyspaces operate only in the **unsigned non-negative integer** domain — it is zero-transformation and cheapest; its two important implementations are **big-endian zero-padding** (the forward-order entry point) and **complement inversion** (reverse order layered on this branch, used for timestamps). See the subsections below. Signed / float / cross-type are skills to reach for when you go out of bounds: the document's multi-tenant composite key is a fixed-width struct, and the tuple is its "variable-length, general-purpose polymorphic" version.

**Alternative route outside this coordinate system: custom Comparator.** The above encodes ordering **into the key** by default; RocksDB / HBase / LevelDB also offer custom Comparators, letting the engine compare keys in a given order. Trade-off: if you need cross-engine portability or depend on prefix ordering → encode into the key; single engine with frequently evolving sort criteria → a Comparator removes redundant sort fields from the key, at the cost of cross-engine generality.

#### Implementing the unsigned non-negative integer: big-endian zero-padding (engineering pitfall)

**Never** convert numbers to strings and concatenate them into a key. Lexicographic order differs from numeric order:

```
String order:   "9" > "10"   (compares '9' > '1', '9' has a larger char code)
Numeric order:  9 < 10
```

The correct approach is a fixed-width big-endian byte array. Rust implementation:

> The examples below use colons for readability. Production should use fixed-width offset encoding; see the [OKM project](https://github.com/orbsh/okm) ([English README](https://github.com/orbsh/okm/blob/main/README.md)).

```rust
fn score_key(prefix: &[u8], score: i64, member: &[u8]) -> Vec<u8> {
    let mut key = prefix.to_vec();
    key.extend_from_slice(&score.to_be_bytes());  // 8 bytes, most significant first
    key.push(b':');
    key.extend_from_slice(member);
    key
}
```

All scores share the same byte length (8 bytes); high-order zero-padding is done automatically by hardware instructions, and lexicographic order = numeric order.

##### Why big-endian: memcmp compares left to right, so the most significant byte must come first

memcmp compares byte by byte and **decides as soon as it hits a difference**. For byte order to equal numeric order, the numeric weights must be laid into the byte stream from high to low — which is precisely the definition of big-endian. Little-endian puts the low bits first, so the first byte is the least significant part; once a value crosses a byte boundary (carrying past the first byte), the byte-by-byte comparison result decouples from the numeric comparison.

Take the u16 (2-byte) values 256 = 0x0100 and 1 = 0x0001; here is how the two endiannesses lay them out in the byte stream:

```
Value:        256 (0x0100)          1 (0x0001)
              weights: high8|low8    weights: high8|low8

Big-endian (BE):  [0x01][0x00]      [0x00][0x01]
                  high byte first    high byte first
memcmp:           first byte 0x01 > 0x00 → 256 > 1  ✓ consistent with numeric order

Little-endian (LE): [0x00][0x01]    [0x01][0x00]
                    low byte first   low byte first
memcmp:           first byte 0x00 < 0x01 → decides 256 < 1  ✗ byte order reversed vs numeric order
```

Little-endian puts the lowest byte of 256 (0x00) at the most influential position in the comparison, with the information-carrying "big" side relegated to the second byte — yet memcmp has already issued the opposite verdict using the first byte. **Root cause**: the meaningful information of 256 lives in the high bits, but little-endian lets the information-free low bits speak first. This is not a contrived special case — any value that crosses a byte boundary (256, 65536…) breaks under little-endian; only values that happen to fit entirely in the first byte survive by luck.

Big-endian also has a structural benefit for prefix scans: **numeric encoding and key prefixes are the same thing**. When the namespace occupies the top 15 bits of a u16, the first two bytes are the complete namespace discriminator — "scan all keys of a namespace" is a plain memcmp prefix match with no mask arithmetic. Under little-endian, the namespace bits straddle two bytes and interleave with the direction bit; prefixes would need bit-aligned handling and the scan path degrades.

> For the bit-level details (direction bit in the low position, range partitioning via the high bits, multi-flag ordering), see ADR-0001 / ADR-0002 of the OKM project.


#### Key application: complement encoding — reverse-ordered timestamps

A KV engine strictly sorts keys by byte order, ascending (lexicographic). As timestamps increase, new data naturally lands at the bottom. To achieve "newest data on top" (handy for fetching Top-N most recent memories), the framework design has a textbook-grade physical hack — max value minus current value:

```rust
// Standard ascending key: old data on top, new data at the bottom
let key_ascending = format!("log:{}:{}:", session_id, timestamp).into_bytes();

// Descending key: newly written data naturally sorts to the front!
// u64::MAX - timestamp: the larger (newer) the timestamp, the smaller the result
// smaller → lexicographically earlier in the LSM-Tree → physically "newest first"
let inverted_time = u64::MAX - timestamp;
let mut key_descending = Vec::new();
key_descending.extend_from_slice(format!("log:{}:", session_id).as_bytes());
key_descending.extend_from_slice(&inverted_time.to_be_bytes()); // big-endian guarantees correct bitwise comparison
```

**Physical meaning**: `u64::MAX - 1722500000` = `18446744072007051616`, and `u64::MAX - 1722500001` = `18446744072007051615`. The latter is smaller and sorts earlier in the LSM-Tree — the larger (newer) the timestamp, the smaller the key's lexicographic value, so a forward iterator scan naturally yields reverse-chronological results.

**Use cases**: loading agent conversation history newest-first, leaderboards fetching latest records, reading log timelines in reverse. Any prefix-scan scenario that needs "newest first" qualifies. It complements `SeekForPrev`-style reverse iterators — the latter depend on engine support, while complement inversion works on all KV engines.

> **Alignment**: here `timestamp` / `inverted_time` are u64, falling under the overview's "unsigned non-negative integer big-endian order-preservation" guarantee — complement inversion is precisely the reverse-order transformation layered on this branch. The transformation applies to all fixed-width integers: signed two's-complement integers enter the unsigned order-preserving domain via a bias (adding the absolute value of `i64::MIN`), and bitwise inversion likewise preserves the reverse-order property; floats are excluded because IEEE 754 byte order is not isomorphic to numeric order and must first be quantized into integer buckets before inversion (implementation reference: the `Reversible` trait in ADR-0004 of the [OKM project](https://github.com/orbsh/okm)).

### Index primary key ID: always appended to the end of the key, value left empty

A secondary index needs to fetch the full record from the primary table, so it must be able to locate the primary table's primary key ID. **Unified policy: append the primary key ID to the end of the index key, leaving the value empty (Index-Only Scan)**, without switching layouts based on whether the index is unique or multi-valued. Why no branching by cardinality:

- **Performance difference is negligible**: when the prefix digest is unique, scan localization is O(log N) and hits exactly one record — comparable to a get. Putting the ID in the value only saves the constant overhead of "parsing the ID once from the end of the key", which is trivial. Maintaining two layouts for a constant-level difference is not worth it.
- **Consistency with non-unique indexes**: multi-valued indexes inherently require the ID at the end of the key (KV keys must be unique, otherwise later writes overwrite earlier ones). With the ID uniformly in the key, unique and multi-valued indexes share one layout and one `scan(idx, scan_fn)` prefix-scan API; "unique or not" becomes just a caller-side check of "is the scan result Vec length 1 or >1", no longer a layout divergence.
- **Range scans are the norm**: with the ID at the end of the key, fetching multiple records by **range** on the indexed field comes naturally (e.g. "all users with city=sh", "all messages in a timestamp interval") — a plain sequential scan, no table lookup. If the ID were in the value, the query side would not know the full key to scan and could not use a point get; range queries would be impossible. For secondary indexes — structures inherently built for "filtering/listing" — range scans are far more frequent than point lookups.

**The only case where the value is worth it**: pure point-lookup hot paths — the indexed field is guaranteed unique by the business and never used for range listing, purely to save that one constant "parse ID from key" and the 16 bytes the ID occupies in the key. This is an optimization done only after measuring an actual bottleneck, not a default.

```rust
// ID uniformly appended to the end of the key
i:1:user:status:active:{user_id}   →  ""
// Unique or not does not affect layout: scan("i:1:user:status:active:") result length speaks for itself
```

**Why the value can stay empty**: the ID is already at the end of the key; walking the key sequence yields all IDs without touching large values and with zero table lookups. If a full record from the primary table is truly needed, do a point get on the primary table using the trailing ID — this step is required whether the ID lives in the key or the value, and the **table-lookup cost is the same**.

**First, a quasi-technical premise to correct**: a unique prefix digest **≠** the ability to get. A get needs the full key, and when the ID is in the key, the query side, at query time, precisely **does not know the ID** — so it still scans the prefix and parses the ID from the end of the key after a hit. This is the other side of "performance difference is negligible" — even with the ID in the value, a get saves only that one parse, not a scan.

### Secondary indexes must append the primary key ID at the end

When designing a secondary index (e.g. looking up conversation messages by timestamp), the globally unique entity primary key ID must be appended to the very end of the index key.

**Reason**: under high concurrency, two user operations may produce exactly identical timestamps within the same microsecond. If the index key is only `idx:time:{timestamp}`, the later write silently overwrites the earlier one (data loss). Appending `{message_id}` at the end guarantees absolute key uniqueness while using lexicographic order to keep the timeline tidy.

```
# Wrong: same timestamp overwrites
idx:time:1722500000 → msg_abc  (overwritten)
idx:time:1722500000 → msg_def

# Correct: primary key ID guarantees uniqueness
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

### Hash prefix sharding: avoiding the write-amplification disaster of auto-increment sequences

When the input source is strictly monotonic and increasing (high-frequency event sequence numbers, log ticks), all writes hammer the tail of the LSM-Tree (hotspot), background compaction produces severe disk write amplification, and IOPS spike.

Solution: inject a hash salt at the front of the key, spreading consecutive writes across multiple partitions:

```rust
use murmur3::murmur3_32;

// Generate a 32-bit hash over the key's core content, mod 16 buckets
let hash_prefix = (murmur3_32(&mut cursor, 0).unwrap() % 16) as u8;

let mut sharded_key = Vec::new();
sharded_key.push(hash_prefix);  // 1-byte hash salt
sharded_key.extend_from_slice(b"metrics:ts:");
sharded_key.extend_from_slice(&timestamp.to_be_bytes());
```

Physical effect: the continuously increasing write pressure is evenly spread across 16 independent in-memory trees (MemTable); multi-threaded concurrent flushes proceed in parallel, avoiding local block lock contention and doubling write throughput. The cost is that prefix scans must traverse all buckets — suitable for write-heavy workloads where reads are exact-key point lookups.

### Length-prefix encoding: eliminating inefficient string-splitting scans

When concatenating multiple string attributes into a key, colon separation (`table:app_name:user_id`) requires writing a `split` loop at read time to find delimiters — O(N) string scanning. Use a fixed 2-byte length prefix instead:

```rust
pub fn pack_string_component(buf: &mut Vec<u8>, component: &str) {
    let bytes = component.as_bytes();
    let len = bytes.len() as u16;  // 2 bytes store the string's physical length
    buf.extend_from_slice(&len.to_be_bytes());
    buf.extend_from_slice(bytes);
}
```

During deserialization, the CPU reads the first 2 bytes to learn the length, then jumps directly to the corresponding offset to extract the target field — going from O(N) string scanning to O(1) memory-address offset arithmetic.

**Comparison with colon separation**:

| Dimension | Colon-separated `a:b:c` | Length-prefixed `[2]a[1]b[1]c` |
|:--|:--|:--|
| Deserialization | split loop + string comparison | 2-byte length read + pointer offset |
| Binary safety | Fields cannot contain `:` | Any bytes allowed |
| Fixed width | No (delimiter-dependent) | Yes (each segment = 2-byte length + N bytes data) |
| Use case | Human-readable debugging | Production high-frequency read/write |

### Numeric namespace dictionary

Namespace requirements are predictable and closed (changing them means changing code), so **compile-time constants** compress string prefixes into fixed 2-byte u16 namespace IDs:

| String scheme | Numeric scheme | Compression |
|:--|:--|:--|
| `"user_sessions:"` (14 bytes) | `u16::to_be_bytes()` → 2 bytes | **85%** |

Namespace IDs are folded into instruction immediates by a proc macro: zero runtime lookup when writing keys — faster than even an L1-resident HashMap (no hashing, no load).

**Fallback**: only if you forgo the proc-macro/constant scheme (leaving ns→ID resolution to runtime) do you need a HashMap (small enough to be L1-resident) — a necessary implementation when the constant optimization is absent, not the first choice; runtime "open" registration of namespaces is pointless since the access pattern lives in the code — adding dynamic registration still requires code changes and buys no dynamic access.

The global namespace dictionary (compile-time constants, proc-macro implemented; see the [OKM project](https://github.com/orbsh/okm)):

```
Namespace 1 → sessions
Namespace 2 → users
Namespace 3 → logs
```

### Encoding optimization: compressing variable content into compact representations (overview)

KV must not only align key structure but also compress the **content** of fields. Principle: first ask about the data's **dimension**, then pick the technique — the goal is trading size for faster comparison / indexing / sorting, at the cost of decode complexity on the read path. **Type-driven implementation: see the [OKM project](https://github.com/orbsh/okm) (ADR-0004 "Value side and wrappers")** (`Enum<u8>` / `Offset<i32>` / `Delta<i32>` … declared once, encode/decode automatically).

Five dimensions by "what gets compressed":

- **Cardinality dimension** (few distinct values)
  - **Low-cardinality numbering**: closed-cardinality values like enums → numbered; **bit width is locked to the fixed byte width of the chosen type** (≤256 variants use `u8`), not dynamically derived from the current cardinality — derivation requires runtime registration (`LazyLock` participating in encoding computation), and auto-expanding bit width as variants grow would make historical key layouts drift. Log error types / status / categories.
  - **Block-level dictionary** (generalization of low cardinality): large value domain but **intra-block repetition** → build a dictionary per block and store index references (columnar dictionary / bitmap). Dictionary built at runtime; random access sacrificed.
- **Distribution dimension** (values clustered)
  - **Baseline offset**: set a baseline (recent, e.g. last year) and store the **signed difference** — event times concentrate within a few years of now, fitting in one small signed integer; negative offsets express earlier values.
  - **Delta encoding** (advanced): sequences store the **difference** from the previous value, usually a small positive number, saving even more with VarInt; the cost is that random access requires sequential reconstruction.
- **Repetition dimension**
  - **RLE run-length**: consecutive identical values stored as `(value, count)` — logs repeating the same error, states staying unchanged for long periods.
- **Bit-width dimension** (compress common small values / magnitudes)
  - **VarInt**: common small values take 1 byte, large values extend (LEB128) — complementary to fixed-width; compresses "common values" rather than "the ceiling".
  - **Magnitude scaling / precision normalization**: reduce time precision (ns→ms) or bucket it (hourly buckets), directly reducing cardinality / bit width.
  - **Bit-field packing**: multiple enums / booleans packed into the same byte.
- **Sort-direction dimension** (reversing byte-order scan order)
  - **Complement inversion**: bitwise inversion of fixed-width integers turns ascending byte order into descending — the first entry of a prefix scan is the newest (timelines, newest-first lists). Principle: see the complement inversion section of "physical-layer encoding paradigms"; implemented as the `Reverse<T>` wrapper with the `Reversible` trait as a compile-time whitelist (fixed-width integers only, floats excluded); see OKM project ADR-0004.

**Trade-offs**: the four dimensions are not mutually exclusive and can be stacked (an `Offset<i32>` field can still participate in prefix scans). The core cost = read-path **decode complexity** (VarInt / Delta especially) and **loss of random access** (Delta / RLE order dependence). "Zero parsing" only holds for fixed-width / global-baseline cases with O(1) reads; variable-length compression requires decoding on reads and is only worth it when the compression gain exceeds the decode cost.

### Multi-tenant composite key codec: industrial-grade implementation

A production-grade component — zero-heap-allocation multi-tenant composite key serialization/deserialization:

```rust
use std::convert::TryInto;

pub struct AgentMemoryKey {
    pub tenant_id: u32,        // 4 bytes (tenant isolation)
    pub session_id: [u8; 16],  // 16 bytes (raw UUID binary)
    pub timestamp: u64,        // 8 bytes (reverse-ordered timestamp)
}

impl AgentMemoryKey {
    /// Serialize: domain model → fixed 28-byte binary stream (no heap allocation)
    pub fn serialize_to_bytes(&self) -> Vec<u8> {
        // 4(tenant) + 16(session) + 8(time) = 28 bytes, all fixed-width, split by offset, no delimiters
        let mut buf = Vec::with_capacity(28);

        // Tenant ID (big-endian)
        buf.extend_from_slice(&self.tenant_id.to_be_bytes());

        // Session UUID (fixed 16 bytes, no encoding needed)
        buf.extend_from_slice(&self.session_id);

        // Reverse-ordered timestamp (complement inversion, newest first)
        let inverted_time = u64::MAX - self.timestamp;
        buf.extend_from_slice(&inverted_time.to_be_bytes());

        buf
    }

    /// Deserialize: binary slice → domain model (zero-copy, nanosecond scale)
    pub fn deserialize_from_bytes(bytes: &[u8]) -> Result<Self, &'static str> {
        if bytes.len() != 28 {
            return Err("illegal physical key length constraint violation");
        }

        let tenant_id = u32::from_be_bytes(bytes[0..4].try_into().unwrap());
        let mut session_id = [0u8; 16];
        session_id.copy_from_slice(&bytes[4..20]);
        let inverted_time = u64::from_be_bytes(bytes[20..28].try_into().unwrap());
        let timestamp = u64::MAX - inverted_time;  // invert back

        Ok(Self { tenant_id, session_id, timestamp })
    }
}
```

**Design points**: all fields are fixed width (4+16+8=28 bytes); fixed-width fields are split by offset alone, no delimiters; deserialization requires no parsing and no heap allocation — pure pointer/slice operations. The reverse-ordered timestamp is embedded in the key, so a forward iterator scan yields "newest first" results.

## KV Implementations of SQL Operations

Several patterns from the design-patterns chapter (Index-Only Scan, Bitmap pre-interception, application-layer MVCC, WiscKey separation) exploit the physical properties of a KV engine (LSM-Tree lexicographic order, complement inversion, WiscKey separation). This chapter starts from the SQL perspective: how multi-dimensional queries, JOINs, and aggregation — the core operations of relational databases — are implemented on a pure KV foundation.

### Inverted index intersection (KV implementation of multi-dimensional queries)

SQL's multi-condition combined query:

```sql
SELECT * FROM orders WHERE status = 'shipped' AND region = 'east' AND amount > 1000;
```

KV has only two primitives: point lookup and prefix scan. Multi-dimensional queries are implemented via **inverted index + in-memory intersection**, akin to the posting list intersection algorithms of Elasticsearch/Lucene.

#### Write path: one index per dimension

```
Primary data table:
  data:order:{order_id}  →  [full record]

Dimension indexes (one key prefix per dimension):
  idx:status:shipped:{order_id}  →  ""    ← value empty, ID at end of key
  idx:region:east:{order_id}     →  ""
  idx:user:u7:{order_id}         →  ""
```

**The entity ID is encoded at the end of the key** (§ "Secondary indexes must append the primary key ID"), so scan results are naturally ordered. The value is left empty — Index-Only Scan, never touching the large values in the data layer.

Each data write atomically commits the primary record plus all index entries in one Batch:

```rust
let mut batch = kv.batch();
batch.put(&order_key, &order_data);
batch.put(b"idx:status:shipped:00123", b"");
batch.put(b"idx:region:east:00123", b"");
batch.put(b"idx:user:u7:00123", b"");
batch.commit()?;
```

#### Read path: scan → intersect → table lookup

```rust
// 1. Two prefix scans
let a = kv.scan(b"idx:status:shipped:").map(extract_id).collect::<Vec<_>>();
let b = kv.scan(b"idx:region:east:").map(extract_id).collect::<Vec<_>>();

// 2. Merge intersection (two-pointer over sorted arrays, O(N+M), CPU L1 cache, nanosecond scale)
let common = intersect_sorted(&a, &b);

// 3. Table lookups (one get per ID, ~1μs on NVMe)
let results: Vec<_> = common.iter()
    .map(|id| kv.get(&data_key(id)))
    .collect();
```

The `intersect_sorted` implementation: two-pointer merge, the same algorithm as Lucene's posting list intersection and the [Merge Join in the JOIN section](#batch-joindouble-pointer-mergemerge-join).

#### Performance analysis

```
Suppose: status hits 1000 records, region hits 500, intersection is 200

Step 1: two sequential scans → 1500 iterator Next calls (sequential I/O, ~μs scale)
Step 2: merge intersection → 1500 comparisons (CPU, nanosecond scale)
Step 3: 200 point lookups → ~200μs (NVMe random reads)

Total: ~1ms
```

SQL B-Tree index scans do the same thing: two index seeks + merge + table lookup, also on the order of ~1ms. **Mathematically equivalent, different I/O patterns**: each B-Tree node seek = a random read; an LSM-Tree prefix scan = a sequential read. On NVMe, sequential reads are 5-10× faster than random reads.

**Zero-cost sorting is a structural advantage of KV**. SQL's Merge Join has an implicit precondition: both tables must already be sorted on the JOIN column. Without an index, the database must first run an O(N log N) sort before the two pointers can start. KV's prefix scans return sorted results by nature — the LSM-Tree's lexicographic order *is* the sort; MemTable + SSTable iterator output is strictly key-ordered. Skipping the sort step, inverted index intersection goes straight into the O(N+M) merge phase, one full sort fewer than a SQL Merge Join.

Caveat: when SQL does have an index on the JOIN column, the B-Tree leaf chain is naturally ordered and sorting is likewise free. KV's advantage is not "SQL cannot be ordered" but that **prefix scans guarantee ordering in every case, independent of index choice** — at design time the key prefix embeds the ordering; at runtime no decision about whether to use an index is needed.

#### Write amplification: the price per dimension

Each additional index dimension means one more put per data write. 3 dimensions = 4 writes (1 primary + 3 indexes); 10 dimensions = 11 writes. This is the physical price of every indexing system — SQL's cost of maintaining B-Tree indexes is essentially the same.

#### Comparison of three strategies

| Strategy | Dimension combination | Read latency | Use case |
|:--|:--|:--|:--|
| **Prefix encoding** (Index-Only Scan) | Fixed at design time, ≤3 dimensions | 1 scan | Few dimensions, fixed combinations |
| **Inverted index intersection** | Any combination | 2+ scans + intersection + table lookup | Many dimensions, unpredictable combinations |
| **Bitmap interception** (bitmap pre-interception) | Fixed at design time, limited dimension values | 0 KV ops (pure CPU) | High-concurrency hot paths |

#### The evolution of data-assembly paradigms: SQL JOIN → ORM → application-layer assembly

SQL JOIN, ORM, and embedded KV are three stations on the same evolutionary line: **data assembly is progressively reclaimed from the engine black box to the application layer**.

**SQL JOIN: declarative abstraction leakage**. JOIN is declarative data assembly — you describe "what you want" and the optimizer decides "how to do it" (Hash Join / Nested Loop / Merge Join). The cost is entrusting the execution path to a black box: when data volume or distribution changes, the optimizer may suddenly switch to a catastrophic plan (full table scan followed by a Cartesian product), and developers are forced to write `STRAIGHT_JOIN` or add hints to save themselves — effectively tearing up the declarative abstraction with their own hands to fight the engine.

**ORM: a procedural mental model stretched over the network**. ORM is popular because when humans handle complex business logic, a procedural + control-flow mental model is far more intuitive than SQL's set theory: query table A for a list of IDs, then query table B, assemble the object graph at the application layer — intermediate results have variable names and the logic is visible. But ORM runs on top of SQL, and every assembly step requires a network round trip (application ↔ database) — list queries are accompanied by N related queries, and N+1 collapses performance. **ORM's failure point is not the direction of "application-layer assembly" but that every assembly call carries one RTT**.

**Embedded KV: localizing the assembly operations**. KV keeps ORM's procedural assembly mental model (`get(user_id)` for the main record, `scan(prefix)` for related records), but drives the cost of each call down to local μs scale — once the fatal term of N+1 (network round trip) disappears, 100 in-process point lookups can take less total time than one complex JOIN across the network. The assembly path *is* the code path; there is no chance for an optimizer to pick the wrong index.

The rationale of this evolutionary line is the migration of the cost function: declarative saves the cost of "a human writing the execution plan", with the black-box risk of "the optimizer guessing wrong"; procedural pays "number of assembly steps × per-call latency", and the drop of per-call latency from network RTT to local μs is the structural precondition that makes procedural assembly viable again in embedded KV.

## Denormalization vs Application-Layer JOIN

The essence of a SQL JOIN is "merging two entities by their join keys". KV has no notion of a join table — there is only one core decision: **merge at write time, or merge at read time**.

#### Write-time denormalization (read-heavy scenarios)

Embed the frequently used fields of related entities redundantly into the primary record at write time. One get at read time retrieves everything, zero JOINs:

```rust
let mut batch = kv.batch();
// Primary table: stores only its own data
batch.put(&order_key, &order_data);
// Denormalized view: embeds frequently used fields of the related entity
batch.put(&full_key, &denormalized_data);  // {amount, status, user_name, user_email}
batch.commit()?;
```

**Cost**: data redundancy + write amplification (one extra put per redundant copy). **Use when**: related fields are small (name, email) and reads far outnumber writes.

#### Read-time application-layer JOIN (write-heavy / large related fields)

Keep entities stored independently; two queries + application-layer merge at read time:

```
Orders table: data:order:{order_id}  →  {user_id, amount, timestamp}
Users table: data:user:{user_id}    →  {name, email, phone, avatar, settings}
```

```rust
let order = kv.get(&order_key)?;                          // point lookup, ~μs
let user = kv.get(&data_user_key(order.user_id))?;       // point lookup, ~μs
let result = merge(order, user);                           // application-layer merge
```

**Cost**: 2 KV point lookups (~μs each), no redundancy. **Use when**: the related entity is large (a full user profile) or updated frequently (writes then need no synchronized update of the denormalized view).

#### Batch JOIN: two-pointer merge (Merge Join)

When you need to batch-join two ordered sets, KV's LSM-Tree lexicographic order naturally supports **Merge Join** — the same algorithm as SQL's Sort-Merge Join, but KV skips the sorting step (prefix scans already guarantee order).

```
Two ordered sets (obtained via prefix scans):
A: [order:101:2026-01, order:101:2026-02, order:101:2026-03]  ← sorted
B: [user:101, user:102, user:103]                              ← sorted

Two-pointer merge:
i=0, j=0
while i < len(A) and j < len(B):
    if A[i].key == B[j].key:
        result.append(merge(A[i], B[j]))
        i++; j++
    elif A[i].key < B[j].key:
        i++
    else:
        j++
```

```rust
// Pseudocode: batch JOIN
let orders = kv.scan(prefix: "order:101:");   // O(N), already sorted
let users = kv.scan(prefix: "user:");         // O(M), already sorted

// Two-pointer merge, O(N+M), zero extra sorting
let mut i = 0; let mut j = 0;
while i < orders.len() && j < users.len() {
    if orders[i].user_id == users[j].id {
        result.push(merge(&orders[i], &users[j]));
        i += 1; j += 1;
    } else if orders[i].user_id < users[j].id {
        i += 1;
    } else {
        j += 1;
    }
}
```

**Compared with SQL**: SQL's Sort-Merge Join needs a sort first (O(N log N)); KV's prefix scans are naturally ordered (O(N)), saving one sorting pass. This is the same algorithm as the inverted index intersection (two-pointer merge) above — the LSM-Tree's lexicographic order is a natural accelerator for Merge Join.

**Use when**: batch joins where both sets are already sorted on the join key (guaranteed via key encoding).

#### Many-to-many relationships: pointer indexes

When the relationship itself is a query dimension (tags, categories, many-to-many), store the related IDs in a secondary index:

```
Order relationship indexes:
  rel:order:{order_id}:user  →  "user_abc"              (single value)
  rel:order:{order_id}:items →  [item_1, item_2, ...]   (multi-value, length-prefix encoded)
```

Essentially, this moves the relational database's foreign key index up to the KV layer. Relationships can evolve independently of the entity data.

#### Choosing a JOIN strategy

| Scenario | Recommendation | Rationale |
|:--|:--|:--|
| Order + username (small field, read-heavy) | Write-time denormalization | One get gets everything, lowest latency |
| Order + full user profile (large field) | Read-time JOIN | Avoids redundantly storing large objects |
| Many-to-many relationships (tags, categories) | Pointer indexes | Relationships evolve independently |
| Relationships over a fixed 2-3 tables | Composite entity encoding | Flatten directly into one large key |

### Pre-aggregated counters (KV implementation of GROUP BY)

SQL's `SELECT region, COUNT(*), SUM(amount) FROM orders GROUP BY region` computes on demand at query time. KV has no aggregation primitive; the two solutions each carry costs:

#### Maintaining pre-aggregations at write time

On each write of the primary data, synchronously update the counters of all GROUP BY combinations:

```rust
let mut batch = kv.batch();
// Primary table
batch.put(&order_key, &order_data);
// Aggregations by day + status + region
let date = timestamp_to_date(order.timestamp);
batch.put(&count_key(b"agg:daily:", date, &order.status, &order.region), &increment(1));
batch.put(&sum_key(b"agg:daily:", date, &order.status, &order.region, "amount"), &add(order.amount));
batch.commit()?;
```

Reading is one point lookup: `get("agg:daily:2026-08-05:shipped:east:count")` → sub-microsecond return.

**Cost**: write amplification grows exponentially with the number of GROUP BY dimensions. Day × status × region × category = dozens of counters updated per order write. Suitable for scenarios with few dimensions (2-3) and extremely frequent queries (real-time dashboards).

#### Full scan + application-layer aggregation

Suitable when data volume is small or queries are very infrequent:

```rust
let results = kv.scan(b"idx:time:2026-08:")
    .filter(|r| r.status == "shipped")
    .group_by(|r| r.region)           // application-layer GROUP BY
    .map(|(region, orders)| (region, orders.len(), orders.sum(|o| o.amount)))
    .collect();
```

**Cost**: full scan + in-memory aggregation; OOM risk at large data volumes. Same physical cost as SQL's full-table-scan GROUP BY — SQL performs no optimization for GROUP BY fields without indexes either.

### KV's DDL: index management and field evolution

SQL has `CREATE INDEX` / `ALTER TABLE`; KV has no equivalent declarative DDL. All schema changes are application-layer encoding problems. But generic tools can be designed to make the operations mechanical.

#### Adding an index: backfilling historical data

New writes use the new key scheme immediately, at zero cost. Historical data needs a migration backfill:

```
Old key scheme:  m:{session_id}:{reverse_ts}:{msg_id}
New index scheme: idx:{session_id}:{msg_id}    ← look up a single record by session
```

A generic migration tool interface:

```bash
kv-migrate \
  --source-prefix "m:{sid}:" \
  --target-prefix "idx:{sid}:" \
  --transform "extract_msg_id_from_value" \
  --batch-size 10000
```

**How it works**: `scan(source prefix)` → for each record → `encode(target scheme)` → `batch.write()`. Batches commit atomically; the LSM-Tree's append-only writes do not lock readers or writers, so migration can run concurrently with normal serving. Resumability: the last processed key is recorded in a system table; after an interruption it resumes from the checkpoint.

**Dropping an index**: simply stop writing to that prefix. Old keys need not be deleted — LSM-Tree Compaction reclaims the space automatically in the background. If immediate release is required: `scan(old prefix) → batch.remove()`.

#### Adding/removing fields: versioned encoding

The value is a byte stream with no schema. Field changes are managed through encoding versions:

```
V1: [version=1][name:u16_len][age:u8]
V2: [version=2][name:u16_len][age:u8][email:u16_len]   ← field added
V3: [version=3][name:u16_len][email:u16_len]            ← age removed
```

Dispatch by version number on read:

```rust
fn decode(buf: &[u8]) -> Record {
    match buf[0] {
        1 => decode_v1(&buf[1..]),
        2 => decode_v2(&buf[1..]),
        3 => decode_v3(&buf[1..]),
        _ => unreachable!()
    }
}
```

**Core advantage**: old data needs no migration; old and new versions coexist. New writes use V3; old data is read with V1/V2 decoders — both live side by side in the same database. Migration can be deferred to idle windows.

**When a backfill migration is needed**: once the decoder maintains 3+ versions, the code starts to rot. At that point run a migration: scan the values of old Versions → re-encode to the latest Version → overwrite with Batch writes. After migration, the old decoder branches can be deleted.

#### Choosing an encoding strategy

| Strategy | Use case | Pros | Cons |
|:---|:---|:---|:---|
| **Versioned encoding** | Frequent field changes (development phase) | Zero migration of old data, old/new coexist | Decoder maintains multiple versions |
| **Append-only encoding** | Add only, never remove | Simplest, zero migration | Value bloat (holes) |
| **Migration backfill** | Cleanup once fields stabilize | Compact values, single-version decoder | One-time migration cost |

#### The essential difference from SQL DDL

| Operation | SQL | KV |
|:---|:---|:---|
| Add index | `CREATE INDEX` = in-database full table scan + rebuild | New key scheme + migration tool backfill |
| Drop index | `DROP INDEX` = in-database deletion | Stop writing; Compaction reclaims automatically |
| Add field | `ALTER TABLE ADD COLUMN` = minutes-level lock on large tables | New version encoding, zero DDL |
| Drop field | `ALTER TABLE DROP COLUMN` = table lock + data rewrite | New version encoding, old data unmigrated |
| Rename field | `ALTER TABLE RENAME COLUMN` | New version encoding (KV has no "column name" concept) |

**Verdict**: KV's DDL does not exist at the storage engine layer; it is application-layer encoding conventions + generic migration tooling. Versioned encoding turns the cost of change from "must backfill immediately" into "can be deferred", and migration tooling turns the backfill cost from "hand-written scripts" into "one command".

### Division of labor between the encoding layer and the engine layer

Application-layer encoding (OKM-style type encoding, physical-layer encoding paradigms) and the storage engine's built-in mechanisms (block compression, strongly-typed APIs) each handle a segment; the boundary is often confused. Use fjall and redb as reference points to clarify the division.

#### Engine-level block compression cannot replace key encoding

fjall (via lsm-tree) ships with SST block-level compression: `CompressionPolicy` configured per LSM level (e.g. L0 uncompressed, deeper levels Lz4), `CompressionType::{None, Lz4}`. The same standard playbook as RocksDB. But it only applies to SST blocks on disk; the remaining cost dimensions of key length persist unchanged:

- **MemTable resides fully in memory, uncompressed** — every extra key byte is a constant-factor RAM cost
- **Separator keys of the block index, bloom filters** are mostly stored uncompressed
- **The comparator must decode the full key bytes on every comparison** — length directly affects CPU

The high-water-mark judgment of "the key holds only fields necessary for localization and sorting" cannot be replaced one bit by engine compression.

Sorting semantics are something compression cannot provide either: `Reverse<T>` complement inversion, field order as scan semantics, namespace clustering — these are **structural semantics** (how bytes are laid out determines whether scan works); LZ4 is **byte-pattern statistics** (how to store shorter), and after decompression the byte order is restored. Compression does not change ordering; ordering does not save space. Orthogonal.

The two actually complement each other: block compression feeds on prefix repetition — compressing the namespace into a 2-byte u16 and the natural clustering of same-namespace records lets Lz4 find longer common prefixes within a block, actually improving compression ratios. Discrete random keys compress poorly.

What genuinely overlaps with block compression in function is **field-level variable-length compression** (VarInt/Delta/RLE) on the "saving disk bytes" axis. But the motivations differ: Delta/VarInt shrink fixed-width fields so the MemTable and caches also benefit (block compression only works at the disk layer); and block compression is whole-block granularity — reading one record requires decompressing the whole block, while field-level encoding keeps the point-lookup path decompression-free. Concrete trade-off: **compression in the disk dimension should be left to the engine's block compression; field-level variable-length encoding is used only when in-memory residency is the dominant concern**.

#### Strongly-typed KV (redb) cannot replace layout semantics

redb's API is strongly typed: each key/value type implements the `Key`/`Value` traits (`as_bytes`/`from_bytes`/`fixed_width`/`compare`), tables are declared as `TableDefinition<K, V>`, and storage/retrieval use Rust types directly. This "type ↔ bytes" boilerplate **overlaps heavily in form** with OKM-style encoding macros — but what redb provides are implementation hooks, not design:

- **Layout semantics**: redb's `compare` defaults to the derived byte order (memcpy order or postcard order). Namespace prefix clustering, field order = scan semantics, reverse-order scans, prefix matching under variable-width offsets — these byte-layout designs in redb require a custom `compare` + carefully constructed `as_bytes`; the design itself (i.e. the encoding rules) cannot be omitted.
- **Cross-type layout discipline**: redb derives each type independently, and byte formats drift per type; cross-type prefix scans (a global timeline across sessions) rely on coincidental convention. A compile-time-locked unified physical layout (anchored by hex tests) is an application-layer concern.
- **`fixed_width` is only metadata**: redb uses it for within-page optimizations but does not manage composite layouts of "fixed-width fields + length prefixes + TLV extension area".

Thus with redb the encoding layer does not disappear; rather, **the macro's emit target changes**: `KeyEncode` expands to `impl redb::Key` (including a custom `compare`), `ValueEncode` expands to `impl redb::Value`, with the encoding rules unchanged word for word. This also confirms the design of the encoding macros as "zero I/O, pure encode/decode" — switching engines changes only the emit target; the encoding contract stands.

**Division-of-labor overview**: the engine provides physical-layer mechanisms (block compression, page management, type-conversion boilerplate); the application layer provides semantic-layer design (byte-order layout, scan semantics, cross-type discipline, memory-residency optimization). The two are orthogonal and mutually reinforcing — a good layout feeds the engine's compression, and the engine's compression amplifies the benefits of a good layout.

### Multi-model capabilities on top of KV: vector search, full-text retrieval, graph queries

All three capabilities are encoding patterns on top of KV — key prefixes simulate data structures, value encoding stores data, and scan simulates traversal. Algorithm libraries provide the computation logic; KV provides persistence and scanning.

#### 1. Vector search (DiskANN / HNSW)

**Encoding pattern**:
```
vec:{id}              → [float32 × dims]       ← the vector itself
graph:{id}            → [u32 × K]              ← graph adjacency list (K neighbor IDs)
meta:entry_point      → [u32]                   ← graph entry node
```

**Query flow** (greedy search):
1. Start from the entry_point
2. `get(graph:{current})` fetches the K neighbors
3. `batch.get([vec:{n1}, vec:{n2}, ...])` fetches the vectors in bulk
4. Compute distances; greedily jump to the nearest neighbor
5. Repeat until converged

**Existing algorithm libraries**:

| Library | Algorithm | Integration |
|:---|:---|:---|
| **usearch** | HNSW + Vamana (DiskANN family) | FFI linkage, graph stored in KV |
| **arroy** | HNSW (Meilisearch core) | Direct dependency |

The library handles graph building and distance computation; KV handles persisting the graph structure. `usearch`'s `add()` and `search()` are pure algorithms, not bound to storage — you can dump its internal data structures to KV and read them back from KV on load.

**Industrial-grade offline hibernation and zero-latency wake-up** (using fjall as the example): vector retrieval does not need to be resident in memory 7×24. The HNSW graph is serialized and persisted to KV; on node start/wake-up it is deserialized into memory and queries run against the in-memory graph; when no new vectors are inserted, the graph snapshot + raw vectors are packed back to KV and the memory is released into hibernation. The whole lifecycle is a "load → in-memory query → persist and hibernate" cycle:

```rust
// Pseudocode: fusing vector persistence into the host Booth's KV store
pub fn save_vector_node(&self, partition: &PartitionHandle, vector_id: &str, embedding: &[f32]) {
    // postcard compact serialization (verdict from the serialization protocol comparison)
    let serialized_vec = postcard::to_allocvec(embedding).unwrap();

    // Write to local disk (fjall example; any embedded KV is equivalent)
    partition.insert(format!("V:{}", vector_id).as_bytes(), serialized_vec).unwrap();
}
```

#### 2. Full-text search (BM25)

**Encoding pattern**:
```
fwd:{doc_id}           → [{term₁, [0,2]}, {term₂, [1]}]     ← forward index
inv:{term}:{doc_id}    → [tf, [positions]]                     ← inverted index
meta:df:{term}         → [u32]                                  ← document frequency
meta:avg_dl            → [f32]                                  ← average document length
meta:doc_count         → [u32]                                  ← total document count
```

**The value of the inverted index**: `inv:hello:42 → [2, [0,2]]` means the word "hello" appears 2 times in document 42 (TF=2), at positions 0 and 2. BM25 scoring needs TF; the position list serves phrase queries and highlight localization.

**One document becomes N keys**: the document "hello world hello" (doc_id=42) produces:
```
fwd:42          → [{hello, [0,2]}, {world, [1]}]
inv:hello:42    → [2, [0,2]]
inv:world:42    → [1, [1]]
```

A document with 100 unique words = 100 inverted keys. This is the essence of an inverted index — split by word, not stored by document. At insertion the keys are scattered across different regions of the key space (the "foo" region, the "hello" region, the "world" region) — inherently out of order. But the LSM-Tree's MemTable is a sorted structure: whatever the write order, the flush to SSTable is ordered — this is the LSM-Tree's core advantage over the B-Tree.

**Query flow**:
1. Tokenize: `"hello world"` → `["hello", "world"]`
2. Batch query the inverted index: `batch.get([inv:hello:*, inv:world:*])` → doc_id lists
3. BM25 scoring: `score = Σ IDF(term) × (tf × (k1+1)) / (tf + k1 × (1 - b + b × dl/avg_dl))`
4. Heap sort for Top-K

**BM25 scoring**: the core formula is a dozen-odd lines of code; no library needed. What you need is tokenization:

| Library | Purpose |
|:---|:---|
| **jieba-rs** | Chinese tokenization (Rust bindings for jieba) |
| **tantivy** | Built-in tokenization + BM25 + highlighting; reusable whole, or take just the BM25 module |

English tokenizes on whitespace; Chinese needs jieba or n-gram (simple but lower accuracy). Stop-word filtering is application-layer logic — maintain a HashSet and filter after tokenization. Search itself is a KV capability: tokenize → query the inverted index → BM25 scoring → sort; no separate search engine required.

**Source example** (using fjall, with two partitions: forward `D:` and inverted `T:`):

```rust
// [document_store]  Key: "D:<Doc_ID>"        → Value: raw plain text
// [inverted_index]  Key: "T:<Term>:<Doc_ID>" → Value: term frequency (for TF-IDF / BM25 ranking scores)

pub struct SearchEngine {
    docs: PartitionHandle,
    index: PartitionHandle,
}

impl SearchEngine {
    pub fn new(keyspace: &Keyspace) -> Self {
        Self {
            docs: keyspace.open_partition("text_docs", Default::default()).unwrap(),
            index: keyspace.open_partition("text_index", Default::default()).unwrap(),
        }
    }

    // 1. Text analyzer: tokenize and build physical inverted keys
    pub fn index_document(&self, doc_id: &str, content: &str) {
        // Store the raw text first
        self.docs.insert(format!("D:{}", doc_id).as_bytes(), content.as_bytes()).unwrap();

        // Minimal tokenization/cleanup (general dev environments can use rust-stemmers or tantivy-tokenizer for enhancement)
        let words: Vec<String> = content
            .to_lowercase()
            .retain(|c| c.is_alphanumeric() || c.is_whitespace()); // strip punctuation

        let mut term_counts = std::collections::HashMap::new();
        for word in content.split_whitespace() {
            *term_counts.entry(word.to_lowercase()).or_insert(0u32) += 1;
        }

        // 2. Atomically write term frequencies into the inverted index partition
        for (term, count) in term_counts {
            let index_key = format!("T:{}:{}", term, doc_id);
            let val = postcard::to_allocvec(&count).unwrap();
            self.index.insert(index_key.as_bytes(), val).unwrap();
        }
    }

    // 3. Keyword retrieval: intersect the posting lists of multiple terms
    pub fn search(&self, query: &str) -> Vec<String> {
        let search_term = query.to_lowercase();
        let prefix = format!("T:{}:", search_term);
        let mut matched_docs = Vec::new();

        // Fast prefix scan
        for item in self.index.prefix(prefix.as_bytes()) {
            if let Ok((key, _)) = item {
                let key_str = String::from_utf8_lossy(&key);
                let parts: Vec<&str> = key_str.split(':').collect();
                if parts.len() == 3 {
                    matched_docs.push(parts[2].to_string()); // got the Doc_ID
                }
            }
        }
        matched_docs
    }
}
```

#### 3. Graph Data Storage and Querying

**Encoding pattern (property graph, one Key per edge)**:
```
node:{type}:{id}           → [properties msgpack]      ← node properties
edge:{src}:{label}:{dst}   → [properties msgpack]      ← edge properties
adj:{src}:{label}:{dst}    → []                         ← forward adjacency (empty value; the Key itself encodes the relation)
radj:{dst}:{label}:{src}   → []                         ← reverse adjacency
idx:{type}:{prop}:{val}    → [u32 × N]                  ← secondary index
```

**Why one Key per edge instead of merging the adjacency list into a single Value**:

```
// ❌ Single-Key adjacency list
adj:alice:knows → [bob, charlie]
// Adding an edge requires read-modify-write: get → append → put, with a transaction

// ✅ One Key per edge
adj:alice:knows:bob     → []
adj:alice:knows:charlie → []
adj:alice:knows:dave    → []    ← newly added; a direct put, an atomic operation
```

| Operation | Single-Key adjacency list | One Key per edge |
|:---|:---|:---|
| Add edge | get → append → put (3 ops, needs transaction) | put (1 op, atomic) |
| Delete edge | get → remove item → put (3 ops, needs transaction) | remove (1 op, atomic) |
| Query neighbors | get (1 op, O(1)) | prefix scan (O(K), K = fan-out) |
| Concurrency safety | Needs transactions/locks | Naturally safe |
| Key count | N nodes = N keys | N edges = N keys |

One Key per edge is the standard practice. The Redis Graph module and Neo4j's storage layer both use this pattern. The only advantage of the single-Key adjacency list is "fetch all neighbors in one get," but the price is that adding/deleting edges requires transactions — not worth it. It is also a textbook instance of the RMW anti-pattern — when RMW cannot be eliminated by re-encoding (one Key per edge) and a collection-type value must be updated, delta appending is a lighter path than "get → append → put + transaction" (see §Read-Modify-Write).

**Query example** ("friends of Alice's friends who live in Beijing"):
```
1. scan(adj:alice:knows:)                   → [bob, charlie]
2. batch.get([adj:bob:knows:, adj:charlie:knows:])  → [[dave, eve], [frank]]
3. batch.get([node:person:dave, node:person:eve, node:person:frank])
4. filter where city == "北京"
```

Multi-hop queries = multiple rounds of scan/batch.get, each round O(K × log N), where K is the fan-out. This is essentially the same as SurrealDB's graph traversal — no query language wrapper, but the same physical path.

Graph querying is itself a KV encoding pattern — `scan(adj:...) → batch.get(nodes) → filter → recurse`; no library needed. Only complex graph algorithms require dependencies:

| Library | Purpose |
|:---|:---|
| **petgraph** | Algorithms such as PageRank, community detection, shortest paths, connected components |

When querying, load the subgraph from KV into petgraph to run the algorithm, then write results back to KV. Simple BFS/DFS/multi-hop traversal is implemented directly at the KV layer, without loading into an in-memory graph structure.

**Source code example** (using Fjall as the example; `open_partition`/`prefix` are equivalent to the partition/range-scan interfaces of any LSM-Tree KV):

```rust
// [Partition: Out-Edges] Key: "E:out:<Src_ID>:<Edge_Type>:<Dst_ID>" → Value: MsgPack(weight/properties)
// [Partition: In-Edges]  Key: "E:in:<Dst_ID>:<Edge_Type>:<Src_ID>"  → Value: None (exists only as a fast reverse index)

use fjall::{Keyspace, PartitionHandle};
use serde::{Serialize, Deserialize};

pub struct GraphIndexer {
    out_edges: PartitionHandle,
    in_edges: PartitionHandle,
}

impl GraphIndexer {
    pub fn new(keyspace: &Keyspace) -> Self {
        Self {
            out_edges: keyspace.open_partition("graph_out", Default::default()).unwrap(),
            in_edges: keyspace.open_partition("graph_in", Default::default()).unwrap(),
        }
    }

    // 1. Insert an edge: atomically write both out-edge and in-edge directions
    pub fn insert_edge(&self, src: &str, edge_type: &str, dst: &str, weight: f32) {
        let out_key = format!("E:out:{}:{}:{}", src, edge_type, dst);
        let in_key = format!("E:in:{}:{}:{}", dst, edge_type, src);
        let val = postcard::to_allocvec(&weight).unwrap();

        // Sequential writes to disk; the Bloom filter automatically accelerates single-key lookups
        self.out_edges.insert(out_key.as_bytes(), val).unwrap();
        self.in_edges.insert(in_key.as_bytes(), &[]).unwrap();
    }

    // 2. Stream out-degree traversal (zero-copy retrieval of all downstream neighbors of node A)
    pub fn get_out_neighbors(&self, src: &str) -> Vec<String> {
        let prefix = format!("E:out:{}:", src);
        let mut neighbors = Vec::new();

        // Leverage the LSM-Tree's powerful sequential range scan (Range Scan)
        for item in self.out_edges.prefix(prefix.as_bytes()) {
            if let Ok((key, _)) = item {
                let key_str = String::from_utf8_lossy(&key);
                // Cleanly split the string and extract the Dst_ID
                let parts: Vec<&str> = key_str.split(':').collect();
                if parts.len() == 5 {
                    neighbors.push(parts[4].to_string());
                }
            }
        }
        neighbors
    }
}
```

#### Overall Architecture

```
┌─────────────────────────────────────────────────┐
│  Application layer                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ Vector   │ │ Full-text│ │ Graph    │        │
│  │ search   │ │ search   │ │ queries  │        │
│  │ usearch  │ │ tantivy  │ │ petgraph │        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘        │
│       │            │            │               │
│  ┌────┴────────────┴────────────┴─────┐         │
│  │  Encoding layer (Key/Value schema) │         │
│  │  vec: / graph: / fwd: / inv: / adj:│         │
│  └────────────────┬───────────────────┘         │
├───────────────────┼─────────────────────────────┤
│  KV storage layer │                             │
│  ┌────────────────┴───────────────────┐         │
│  │  Fjall / SlateDB / RocksDB         │         │
│  │  scan / get / batch / remove       │         │
│  └────────────────────────────────────┘         │
└─────────────────────────────────────────────────┘
```

All three capabilities are encoding patterns on top of KV, not new storage engines. Combined, they form a multi-model database — architecturally of the same lineage as SurrealDB, just without a query language layer.

**Engine-agnostic vs engine-specific**: The encoding paradigms and source code examples in this chapter are presented in an engine-agnostic way (examples use Fjall; `open_partition`/`prefix` correspond to the partition/range-scan interfaces of any LSM-Tree KV). For the **polyglot collaborative retrieval architecture** that combines these indexes (Steel/PyO3/Rust-driven routing), see [Embedded Scripting Language Selection](embedded-script-languages-en.md) §4.3. For the engine-selection watershed (integrated vs disaggregated storage: the upstream application layer of the object-storage AI stack), see [HelixDB vs LanceDB Object-Storage AI Stack](helixdb-vs-lancedb-en.md).

## Distributed KV: Replication, Consistency, and Mainstream Implementations

The embedded duo (Fjall / SlateDB) covers single-machine and disaggregated storage; when requirements escalate to **strongly consistent data replication + horizontal scaling**, you enter the domain of distributed KV. This chapter covers the construction principles, why most scenarios should not build their own, and how to choose among mainstream implementations. **Consensus/Raft discussion is concentrated here** and not repeated elsewhere.

### Two Orthogonal Axes: Sharding × Replication

A distributed KV's capabilities are determined by two axes; always evaluate them separately:

- **Sharding**: Horizontally partition the key space across multiple nodes — if a single machine cannot hold the data, shard. Includes range/hash partitioning, splitting/migration, and routing.
- **Replication**: Redundancy to guarantee consistency and availability — reads still work when a node/disk dies. Includes consistency protocols and replica counts.

Sharding solves "capacity/throughput"; replication solves "reliability/availability." Many systems do only one (Redis Cluster shards but replicates asynchronously; etcd replicates but does not shard).

### Consistency Models: Strong Consistency vs Eventual Consistency

Reads after replication are determined by the consistency protocol:

- **Leader-based strong consistency (Raft / Paxos)**: Single-writer multi-reader, quorum acknowledgment, **linearizable** (read-your-own-writes, automatic leader election). Pros: simple, provable, no client coordination. Cons: no sharding, full replica on every node, writes must go through quorum.
- **Eventual consistency (gossip / LWW / weak quorum)**: Any node can write, asynchronous propagation, row-level last-write-wins. Pros: always-available, low write latency, horizontally scalable. Cons: reads may be stale, conflicts need reconciliation.

### The Principles and Limits of Raft

Raft is the standard implementation of leader-based replication: append log → quorum acknowledgment → state machine apply → automatic leader election (term/vote/quorum). **Linearizability** makes it the gold standard for metadata consensus — **but its fundamental limitation is: no sharding; each node holds a full copy of the data**. Storage and write amplification therefore grow linearly with node count (3x for 3 replicas is acceptable, but each additional node is another full copy — no further scaling). Conclusion: **Raft should only carry metadata-scale workloads** (registry, configuration, locks, Catalog pointers) — this is exactly the entirety of etcd / Consul / ZooKeeper. For principles and limits see the [Consensus Protocol document](consensus-protocol-en.md); for the implementation perspective (state machine / log repository / snapshot compaction) see its "Implementation Perspective" chapter; for Openraft integration with Fjall see §Consensus and Coordination Hierarchy below.

### How Sharding Rescues Raft: One Raft Group per Shard

Raft does not shard — so shard first, then **run one Raft group per shard** — each Region/shard independently holds 3 replicas; total replication goes from "node count × full copy" to a fixed "3x per shard," no longer growing with node count. This is the **TiKV (one Raft per Region)** path, which actually solved Raft's linear amplification.

But the cost of sharding rescuing Raft is that 80% of the heavy lifting lies **outside sharding**:

- **Shard management**: Splitting / merging / cross-node migration / balancing (PD-level fallback)
- **Routing**: A metadata service mapping key → shard → node (which itself needs consensus-based replication)
- **Cross-shard transactions**: Within one shard it's cheap; across shards you need 2PC / Percolator / a commit coordinator, involving locks, deadlocks, retries

### Comparison of Mainstream Distributed KV

Divided by "consistency × scale" into four classes, corresponding exactly to four use cases:

| Class | Representatives | Positioning |
|:---|:---|:---|
| A. Data scale · Strong consistency · Raft per shard | TiKV (+PD), CockroachDB, YugabyteDB | Ready-made sharding + replication; cross-shard via 2PC/Percolator |
| B. Data scale · Strong consistency · Global ACID | FoundationDB, Spanner | Commit coordinator handles cross-shard transactions; applications are free of 2PC |
| C. Metadata scale · Consensus | etcd, Consul, ZooKeeper | Empirical proof that "Raft is only fit for metadata" |
| D. Data scale · Eventual consistency | Cassandra/Scylla, DynamoDB | always-available, tunable consistency |

**Strong consistency at data scale, two choices (FDB vs TiKV)**:

| Dimension | FoundationDB | TiKV + PD |
|:---|:---|:---|
| Sharding/replication | Ordered global key space + range partitioning; replication at the log/storage layer, not Raft per shard | One Raft group per Region; PD manages placement/balancing/routing |
| Transaction model | Global ACID; cross-shard handled by the commit coordinator; no 2PC for applications | MVCC snapshot isolation; cheap within one shard, cross-shard via Percolator 2PC |
| Conflict handling | Optimistic concurrency: write conflicts abort + retry | Lock-based; under high write conflict, OCC aborts more than lock-based |
| Consistency | Snapshot isolation + conflict detection (often described as equivalent to serializable) | Snapshot isolation (TSO timestamps) |
| Operations | Multi-process roles (coordinator/log/storage), complex | PD + TiKV multi-component, also non-trivial |
| Backing | Apple, Snowflake | TiDB/TiKV, trillion-row scale |

**Selection**:

- Metadata / configuration / service discovery → **etcd / Consul / ZooKeeper** (Class C, the correct use of Raft)
- Strongly consistent distributed data + worry-free cross-shard ACID → **FoundationDB** (Class B)
- Straightforward per-shard Raft + horizontal scaling, low-frequency cross-shard transactions → **TiKV** (Class A)
- SQL integration on top of KV → **CockroachDB / YugabyteDB** (Class A)
- always-available, can tolerate eventual consistency → **Cassandra / Scylla / DynamoDB** (Class D)

### Position Within This Architecture

Metadata coordination goes through consensus (Class C, see the [Consensus Protocol document](consensus-protocol-en.md)); business data is **not sharded or replicated** — Fjall local + lake backup, or SlateDB + S3 (disaggregated storage, replicas fixed at 3-AZ independent of node count). **When you truly need strongly consistent distributed data, don't cobble Fjall into sharding + Raft (that's rebuilding TiKV) — use the ready-made Class A/B systems directly**. Fjall/SlateDB and "sharding + Raft" address two different classes of need; a three-way comparison:

| | Fjall (local) | SlateDB + S3 | TiDB / TiKV (Raft per shard) |
|:---|:---|:---|:---|
| Write latency | < 1ms | 1-10ms | < 1ms |
| Storage cost | 1x (local) | 1x (S3 price) | 3x (Raft replicas) |
| Capacity | Local disk | Unlimited | Local disk × node count |
| Strong consistency | Naturally strong on one machine | S3 eventual consistency | Raft strong consistency |
| Operational complexity | Low | Low | High (PD + Region scheduling) |
| Suitable scenario | Single machine, small scale | Most scenarios | Low latency + strong consistency |

Verdict: pick Fjall for small single-machine scale, SlateDB + S3 for most scenarios, and TiDB / TiKV only for low latency + strong consistency — the 3x storage cost is the price paid for strong consistency; when you don't need it, SlateDB's cost advantage is overwhelming.

Extending the embedded KV engine into distributed coordination infrastructure splits traffic by direction: **North-South traffic** (the network access layer between client ↔ KV server, see the "KV Server's Network Access Layer" section below) and **East-West traffic** (distributed locks and consensus coordination between nodes) — consensus at the bottom guarantees consistency, and consensus-based locks are provided on top of it.

### Consensus and Coordination Hierarchy

```
Business Coordination (locks, scheduling, election)
    └── Meta-Coordination (consensus protocol)
         ├── Log ordering
         ├── State machine state
         └── Membership changes
```

**Core principle**: Metadata consensus is the bedrock of infrastructure, not a storage engine's responsibility. Redis has no consensus layer → Redlock is built on a sandcastle. For the consensus protocol proposal see the [Consensus Protocol document](consensus-protocol-en.md).

**Never rewrite consensus algorithms** — rewriting introduces new bugs at a cost far exceeding the benefit.

**Architecture deployment**:

```
Application layer (locks, scheduling, configuration, sessions)
        │
  State machine (Fjall engine — embedded persistence)
        │
  Fjall engine (LSM-Tree KV — local NVMe)
        │
  ┌─────┼─────┐
  Node1  Node2  Node3    (single-machine or cluster deployment)
```

**Key insight**: Fjall is an in-process embedded engine — local reads involve zero network hops. Metadata consistency needs no consensus layer: aura's ruling is a per-node meta instance (control-plane single-writer, see [Aura Architecture](aura-architecture-en.md)); consensus applies only to data-level strong-consistency replication via external off-the-shelf systems (see the [Consensus Protocol document](consensus-protocol-en.md)).

> **Openraft example (historical analysis)**: Fjall + Openraft integration was analyzed as a candidate path for data-level strong-consistency replication — Raft commits log entries → the state machine `apply`s them into local Fjall. This path is explicitly not part of aura's default architecture (metadata single-writer without consensus; replication delegated to external TiKV-class systems).

### Distributed Scenarios: Concurrent Ordering and Shard Balancing

The single-machine perspective treats key design as local physical layout; across nodes with multiple writers, key design gains two distributed dimensions.

**Concurrent ordering (Versionstamp)**: The reversed-complement encoding assumes the caller can know the timestamp in advance. When appending child items concurrently to the same parent record (multi-threaded/multi-node writers to one event stream, log, or time-series sub-records), two writers may collide on the same timestamp — requiring **server-side serialized allocation of monotonically ordered keys** (versionstamp): order first-ness by observable commit order, guaranteeing appends land in a stable sequence. On a single machine an atomic counter suffices; across nodes, sequence numbers come from the consensus layer or a Leader's monotonic allocation, giving cross-node append logs/event streams a globally consistent commit order. It complements "caller-knows-the-timestamp reversed-complement encoding": one relies on prediction, the other delegates ordering under concurrency to the engine.

**Shard placement and balancing**: Across nodes, key prefixes also determine **shard boundaries** — shards (tablets) are split along key lexicographic order, and same-prefix data lands in the same shard. This cuts both ways: a prefix is both a logical aggregation and a physical locality boundary (good for batch scans/local transactions), but access concentrated on a single prefix creates **hot keys** — one shard saturated while the rest idle (DynamoDB's hot partition). On a single machine, the doc uses hash salting to spread write hotspots (even dispersion, see [Physical-Layer Encoding Paradigms](#physical-layer-encoding-paradigms-universal-physical-layer-techniques-for-key-encoding)); across nodes you must trade off between **locality** (same prefix, same shard — good for batch scans) and **balance** (dispersed prefixes — avoiding hot shards) — hash salting preserves balance but destroys prefix locality; prefixes preserve locality but risk hotspots. This is the most fundamental difference between distributed and single-machine key design: single-machine only manages write amplification; distributed must also manage placement balance.

## The KV Server's Network Access Layer: North-South Traffic (East-West Traffic Belongs to Distribution)

**The default form is single-machine**: embedded engine + network access layer is a complete KV server, with no distributed layer at all — "single machine + network service" is the more pragmatic scenario. Redis and PostgreSQL are likewise single-machine service forms, but for single-machine network services the KV server beats them: zero parsing, zero DDL, the key paradigm as the model (see "When to Use SQL" above); Redis's single thread and memory cost (see [Redis Critique](redis-critique-en.md)) are the degraded aspects of this form.

The term "North-South traffic" exists only to distinguish it from node-to-node "East-West traffic" — **normally only the North-South layer exists**: client ↔ KV server, which is all this section covers. If you truly need cross-node replication/sharding, use ready-made distributed KV directly (FDB/TiKV, see "Distributed KV" above); do not assemble a distributed layer by hand on a single machine: East-West traffic sits at a lower layer, transparent to callers — clients still see the same North-South access layer.

In one sentence: **KV server = KV engine (storage kernel) + network access layer (North-South traffic)**; East-West traffic belongs to the distributed layer and is not needed by default.

### The KV Server's Composition: KV Engine + Network Access Layer

A centralized KV server is not a pre-existing product form (Redis/etcd/FDB/TiKV each carry their own features and historical baggage) but the assembly of two things: **KV engine (storage kernel) + network access layer**. The storage kernel directly reuses an embedded engine (Fjall/SlateDB and the like, see the engine comparison and selection chapter below) — the only genuinely new part is the network access layer: turning "in-process calls" into "cross-process calls."

The channel trade-off is straightforward:

- **HTTP** — provides **convenience**: complete tooling (curl, browsers, SDKs for any language); instructions packaged into a pipeline serve generic clients; the cost is protocol overhead (request headers, handshakes, per-request parsing), with a low performance ceiling.
- **WS (WebSocket)** — provides **performance**: long-lived connection reuse, full duplex, binary frames; request-response becomes frame delivery over a multiplexed connection, with throughput and latency approaching raw TCP; and it is natively compatible with text protocols — JSON instructions run directly over WS, making debugging, logging, and cross-language alignment easy. Between internal services, **WS + Postcard** is usually the better choice.
- **gRPC** — actually no longer necessary: both of HTTP/2 + Protobuf's selling points are neutralized by WS (header overhead replaced by binary frames, strong typing covered by Postcard), while the cost is an entire protobuf toolchain and complex gateways on top. HTTP provides convenience, WS provides performance; gRPC sits in the middle and gets neither.

In one sentence: **KV server = KV base + HTTP/WS**. In business-semantic terms, this architecture effectively **merges MVC's Models and Database into one layer** — there is no object-model-to-schema mapping layer; business semantics are fixed directly in the engine's byte space via the key paradigm ("model" is key encoding, "database" is the value byte string, see the "Composite Key Encoding" section above). Pooling, cloud hosting, and statelessness are all built on the same merged primitives — which is also why "centralized KV server" is independent of SQL and of specific forms like Redis: it is the engine plus two protocol layers.

### Network Layer: A Minimal WS Wrapper Breaks the Single-Thread Limit

Redis's single-threaded model was the optimal solution under 2009 hardware conditions. On multi-core servers, Redis's command execution is locked to a single core — multi-instance sharding introduces client-side routing complexity (see the horizontal selection table in the engine comparison and selection chapter).

The Fjall + Tokio + WS combination provides an equivalent network interface while breaking the single-thread limit:

```
[WS Client]   ← Internal: Postcard binary frames; generic clients: JSON text frames (WS natively compatible)
      │
      ▼
[WS Server — Tokio multi-threaded async runtime]
      │  Multiple CPU cores handle different WS connections simultaneously
      ▼
[Fjall LSM-Tree — Arc<Keyspace> thread-safe]
      │  Multiple threads read/write the same storage instance concurrently
      ▼
[NVMe SSD]
```

**Compute layer**: Tokio is multi-threaded async by nature. Dozens of CPU cores handle different connections in parallel; Redis's single-thread bottleneck does not exist. A maliciously blocking command (e.g., a full KEYS *) only affects a single Tokio task and does not block other connections.

**Storage layer**: Fjall's `Arc<Keyspace>` provides thread safety. Multiple threads can read and write the same Keyspace concurrently; the LSM-Tree's lock-free read path (MemTable + SSTable) and background compaction threads are naturally concurrent.

**In-process read/write path**: When the WS service and Fjall are embedded in the same process, the hot path (Booth state read/write) still uses in-process direct calls (ns-level); WS is only for cross-process external access. Dual paths coexist: in-process zero RTT + standardized network requests.

> **Tonic + gRPC deprecated**: Early versions used Tonic + gRPC for network access; it has since been abandoned entirely. Reasons (channel trade-offs summarized in "The KV Server's Composition" above):
> - **Inflexible**: gRPC binds RPC and serialization into one package (Tonic is Rust's implementation of that package); scheduling, middleware, and connection forms are all nailed down by the HTTP/2 + Protobuf framework — switching load balancers or encodings means introducing new components rather than changing configuration;
> - **Weaker ecosystem than HTTP**: curl, browsers, SDKs for any language, gateways and observability tooling — HTTP/WS is far more mature and widespread than gRPC;
> - **No performance advantage**: compared to WS long-lived binary frames, HTTP/2 headers and streaming RPC complexity buy neither lower latency nor higher throughput — a WS frame is a bare message with no header to trim;
> - **Less universal than WS**: WS is a browser-native protocol with the best natural traversal; the same channel serves internal binary frames and external clients alike; gRPC is unfriendly to both browsers and generic clients;
> - **Industry endorsement**: SurrealDB's default protocol is WS (LIVE SELECT real-time subscriptions depend entirely on it) — among Rust distributed data systems, the leading project chose WS over gRPC; that says it all.

**Performance model**:
- **Single read/write**: ns~μs (in-process) vs Redis 0.1~2ms (network RTT)

- **Throughput (async batch processing)**: Fjall's multi-threaded concurrency scales with core count; Redis caps at ~80K ops/s (single-threaded)

- **Resources**: no separate process needed, LZ4 compression, no dedicated DRAM allocation

## When to Use SQL

Layer two abstractions on top of pure KV — a Parser (query parsing layer) + Optimizer (query optimizer) — and you have a complete database engine. SurrealDB, CockroachDB, and TiDB all took the same path: take a ready-made KV engine (RocksDB/Pebble/SurrealKV) as the base, and write a query language parser and cost-based optimizer on top.
```
Client text query ("SELECT * FROM users WHERE age = 25")
        │
        ▼
┌─ Parser ──────────────────────┐  Text → AST (abstract syntax tree)
└───────────────┬───────────────┘
                ▼
┌─ Optimizer ───────────────────┐  AST → optimal physical execution path
└───────────────┬───────────────┘
                ▼
kv.scan("idx:age:25:") → point lookup with table fetch   ← the KV instructions you write by hand
        │
        ▼
┌─ KV Engine (Fjall/SlateDB) ──┐  Binary bytes to disk
└───────────────────────────────┘
```

### The Myth of SQL's Dynamism

A common argument against KV is "SQL is more flexible/more dynamic" — examined closely, it holds only in a very narrow sense and **does not hold at all at the program-internal level**:
- **Inside the program: SQL is just as hard-coded**. The set of queries an application can issue is fixed at compile/deploy time — SQL statements are hard-coded in source; even with an ORM dynamically assembling SQL, the ORM's mapping rules are themselves hard-coded code that can only generate those few query shapes. So from "what the program can do," SQL is not more dynamic than KV.
- **True dynamism exists only on the "ad-hoc query surface"**: a SQL store can answer arbitrary queries and DDL at runtime for external clients (psql, BI, migrations, consoles) without recompiling the application; bare KV has no such standard query surface — external tools lack key codec knowledge and target types, so they cannot even begin. But this is a **product/architecture decision about whether to deliberately expose a live queryable interface, not inherent to storage semantics**. The vast majority of applications do not expose it, so SQL's internal access is as fixed as KV's.
- **The remaining differences are design language and operations tooling, not dynamism/superiority**: migrations, Catalog, EXPLAIN, indexes form a mature toolchain; key-space modeling is no harder than SQL schema — the ecosystem is just younger. Half of SQL's "standard" comes from everyone having invested learning time — **sunk cost, not innate advantage**.
- **Performance troubleshooting is not easier**: complex plans, joins, cardinality estimation are hard to reason about; KV point-scan semantics are simpler. At a real performance bottleneck, "SQL is easier to debug" is often exactly backwards.

### Resource Pooling / Cloud Hosting / Statelessness: Topological Properties, Not Properties of SQL

A claim often treated as a "SQL advantage" is: a single SQL instance can serve many projects (resource pooling), can be cloud-hosted (ops-free), and projects themselves can be stateless. **All three are true and valuable — but they all come from "storage being abstracted into an independent centralized service," not from the SQL query language**:

- **Resource pooling**: multiple projects sharing one database instance, reusing connections and buffers — this is a **centralized vs embedded** topological difference. An in-process KV cannot do it, but a **centralized KV server** can pool just as well — its essence is **KV engine + network access layer** (North-South traffic form, see "The KV Server's Network Access Layer" above); what's pooled is the merged primitive, independent of specific product forms like Redis.
- **Cloud hosting**: outsourcing state to a managed service to avoid self-operations — an **external hosting** architecture decision, unrelated to the query language. DynamoDB / Redis Cloud / a managed KV server work equally well.
- **Stateless projects**: the application holds no state and relies only on external storage as the single persistence point — a benefit of the **externalized state** architecture. Placing state in a centralized KV server makes applications stateless just the same.

The real axis is not SQL-vs-KV but **embedded vs centralized**:
- **Embedded storage** (Fjall/SQLite/D1) ships with the application — zero network, zero server, but cannot be pooled across projects;
- **Centralized storage service** — poolable, cloud-hostable, stateless applications, but introduces network + server + an operations surface.

If you want the "pooling + stateless" dividend, pick a **centralized KV server** — **no need to switch to SQL**. Treating these three as SQL's victory points you in the wrong direction.

**Pooling has a load ceiling**: the pooling dividend of a shared instance **is only realized under light load** (JAMStack-style light architectures, function computing, multiple instances sharing a storage pool). When data volume makes the shared pool the bottleneck and even dedicated instances max out, **heavy load ultimately can only choose KV** — in-process zero RTT, zero parsing, a single instance scaling linearly; if volume keeps rising there is still a smooth scaling exit (horizontal splitting along the key space, see [Unified Data-Layer Architecture](unified-data-layer-en.md), "Load Weight / Scale").

**Being fair (SQL's only real advantage is its engineering ecosystem, not the language)**: SQL's **operational abstractions are more mature** — RDS/CloudSQL's managed hosting, tenant isolation, role permissions, quotas, connection pooling, WAL replication are all commodity-grade out of the box; building a compact KV cluster yourself means grinding through all of these. This is an **engineering-maturity difference** — "more convenient to do today," not "the SQL language is inherently irreplaceable." Only by admitting this can you avoid falling into the reverse worship of "KV is omniscient and omnipotent."

### Why Stick with Pure KV (The Framework Team's Correct Choice)

When the query pattern is 100% fixed at design time, Parser + Optimizer are redundant runtime overhead:

- **Zero parsing overhead**: hard-code `kv_engine.scan(&prefix)` at compile time; no runtime SQL string parsing
- **100% deterministic response times**: no risk of the optimizer "acting up" and slowing down; P99 latency stays rock solid
- **Single binary size**: no code bloat from a SQL parser, optimizer, or type system

There is also a fundamental advantage missed in most comparisons: **computation hugging the data is a starting-point property of embedded KV, whereas a SQL engine needs to stack another layer to compensate**. For a database to run intensive computation close to the data, it needs an in-process **extension host** — SurrealDB splits execution into two worlds for this reason:

- **Basic queries (declarative SurrealQL)**: scheduled directly by the underlying core, using native indexes, with nearly zero serialization overhead;
- **Intensive algorithms (WASM extension Surrealism)**: precompiled machine code, near-native speed — but every time data is exchanged between the database core and the WASM virtual machine, it must pass through the Host Bridge for serialization and memory copies (Context Switch); boundary-crossing overhead becomes significant when traversing massive data.

Typical AI RAG atomic operations are therefore split into two interlocking layers:

```SurrealQL
-- Outer declarative transaction: lock rows, perform updates
-- Inner WASM extension mod::ml::embed: generate vectors squeezing CPU/GPU right next to the data
UPDATE article SET embedding = mod::ml::embed(content) WHERE id = 'tech_news';
```

Embedded KV has this integration from the start: query and computation are **in the same process, the same binary** — `embed` is just a native function call: zero serialization, zero boundary crossing, naturally atomic. The extension host is a tax SQL engines pay for "separating computation from data," not a mandatory standard (for the decision-level argument see [Unified Data-Layer Architecture](unified-data-layer-en.md), "Why Path B also doesn't choose a single engine" — "compute pushdown").

### What Makes a "Predictable Query Pattern": Demand Controllability (Agency) Decides

KV's zero parsing overhead, deterministic responses, and single-binary size are only realized when the query pattern can be fixed. But **predictability is not a technical property — it is a matter of demand controllability (agency)**:

- **Demands you control and that are stable**: queries can be fixed at design time; KV's physical advantages (append-only writes, zero DDL, deterministic responses) are fully realized, and "stick with pure KV" holds.
- **Demands driven by volatile external forces**: queries are unpredictable. Here **SQL is more universal** — even when SQL cannot implement a requirement, the failure is attributed to "SQL's technical limitations"; this is the industry's unified understanding and responsibility can be externalized; KV, by contrast, turns "why it can't be done" into your implementation responsibility.

The criterion is not the organizational identity of "product vs outsourcing" but **whether demands are under your control**. Two ways to lose agency: **outsourcing** (the client decides requirements), and **building a product that serves customers' volatile demands** (e.g., enterprise management software — customer needs trump everything). The latter is ineffective even if you're the boss: if the business inherently serves others, you have no agency.

### At Large Data Volumes, KV Does Not Require a Predictable Query Pattern

"Predictable query pattern" is KV's first-priority guideline — predictable patterns can be fixed into key paths, eliminating runtime overhead at compile time with zero parsing cost. But there is another easily overlooked dimension: **data volume itself**.

When data volume outgrows what a relational engine can handle, KV's physical advantages still hold even when query patterns change frequently. The reason is that the LSM-Tree's write cost is O(log N), whereas a relational engine's schema-change cost is proportional to table size:

| Operation | KV's cost | Relational's cost |
|:---|:---|:---|
| New query pattern | Start writing keys with a new prefix, **zero DDL** | ALTER TABLE + migration scripts; minutes of locking on large tables |
| New secondary index | Write a new prefix scan function, **zero rebuild** | CREATE INDEX = full table scan + rebuild; hours on large tables |
| Data migration | Old keys retained, new keys written with the new encoding, **zero downtime** | Large-table repartitioning = long locks + copying |

**Physical essence**: the LSM-Tree's SSTables are append-written and immutable. A new key pattern is just one more prefix in the newest MemTable; existing SSTables need no modification. Relational indexes are B-Trees; every structural change involves page splits and reorganization, with cost proportional to table size.

**Practical impact**: even if you change the query pattern once a month — adding a prefix scan, adjusting a composite key layout — KV's cost is O(1) (write a scan function), while the relational cost is O(N) (N = table size). Once N exceeds a certain threshold (starting around a million rows), KV's physical advantage outweighs the engineering friction of "unfixed patterns."

**Verdict**: a predictable query pattern is KV's **best** use case, but not a **necessary** condition. Large data volume is itself a reason to choose KV — the physical advantages of scale (append-only writes, no index rebuilds, zero DDL locks) reduce the cost of pattern changes from O(N) to O(1).

### Requirement Changes Are O(1) for KV: Adding Fields/Tables/Indexes Is Naturally More Controllable

For ordinary requirement changes (adding fields, tables, indexes), KV isn't just "also fine" — it's **naturally O(1)**, more controllable than SQL:

| Requirement change | KV's cost | Relational's cost |
|:---|:---|:---|
| Add field | Evolvable serialization adds a new field; Key unchanged, zero DDL | `ALTER TABLE ADD COLUMN` = large-table lock |
| Add table | New prefix space | `CREATE TABLE` |
| Add index | Write a new scan function | `CREATE INDEX` = full table scan + rebuild |
| Change query | Modify the scan function, zero migration | Optimizer + index redesign |

For whole-row packing: adding a field = adding a field to the serialization (field_id / version); the Key is completely unchanged, old data is read by the old-version reader, old and new coexist — zero DDL, zero migration. Adding a table = a new prefix space; adding an index = adding a scan function — both are "one more line of code," touching no historical data and requiring no online schema migration.

If you adopt a columnar layout with one key per field, adding a field is indeed simpler — just add a new prefix Key. But **that very "simplicity" is why it's inadvisable**: the columnar layout demotes "adding a field" from serialization evolution to "adding a scan dimension," in exchange for N point lookups per whole-row read under OLTP, write amplification, and the cost of cross-key transactions (see "Value packing granularity"); it merely shifts DDL pain into query/write pain — it is not free added controllability — unless the workload is inherently columnar aggregation (OLAP), in which case a layout with that field as prefix is what's needed.

**Corollary — SQL's real remaining advantages**: SQL's advantages lie **not in handling requirement changes** (where it actually loses to KV's zero DDL) but in two things:

1. **Unpredictable ad-hoc composite analysis** — SQL is declarative; Parser + Optimizer is a ready-made general-purpose query engine; new queries require no code; KV is imperative — composite aggregation (joins across tables + multiple conditions + bucketed statistics) must be hand-written scan combinations + application-layer aggregation. Predictable at design time → a tie; surfacing only after the fact → SQL wins.
2. **Cross-boundary sharing** — a key space is a **private protocol**; SQL is the **industry's common currency**. Within a closed single team, a private language is efficient; if you need to integrate with third parties or off-the-shelf BI tools, the key space requires inventing your own protocol + writing documentation + making people learn it.

**Insight**: SQL's remaining advantage is not "stronger at handling requirements" but "delegating query capability to a general-purpose parser you didn't invent." When a team both controls the requirements and is willing to build its own key space, that delegation loses its necessity — but the price is giving up the two safety nets of ad-hoc unknown queries and external sharing's "ready-made engine."

### When You Must Evolve into a Database

Only when a system must open up to external third-party developers, allowing end users to freely write unpredictable complex queries via low-code/dynamic plugins — to prevent them from writing garbage full-table-scan queries that overwhelm the underlying storage — must you add a Parser + Optimizer layer at the front as a query gate.

**AdHoc analysis** is another typical trigger. The data analysts AdHoc serves (broadly including devops analyzing logs) are essentially doing **dynamic multi-dimensional queries** — constantly modifying ranges, drilling down, pivoting aggregations — which cannot be fixed into prefix scans at design time. More fundamentally, this is an **analytical (OLAP) workload**, while KV is essentially **row storage**: each key maps to one value byte stream; prefix scans must read in and decode entire records, with no column projection, compression, or vectorization possible. So even if analysts know the key schema and can write their own scan functions, a row-oriented KV still falls far short of a **columnar OLAP engine (e.g., DuckDB)** on analytical workloads — column storage reads only needed columns, compresses same-typed values at high ratios, and can push down filters and aggregations; the performance advantage is structural. Pure KV's proper domain is point lookups and fixed-pattern scans (OLTP form); **the correct home for dynamic multi-dimensional workloads like AdHoc is a columnar OLAP layer, not pure KV**

**Convergence verdict**: AdHoc goes to DuckDB, concurrent CRUD goes to KV + API, cross-boundary goes to Lakehouse — under the unified KV + Lakehouse architecture, "evolving into a full relational database" has essentially no remaining triggers. Dissecting each trigger:

- **AdHoc analytics → DuckDB**: dynamic multi-dimensional workloads are analytical; DuckDB-class embedded columnar engines can carry them with zero ops — no need to evolve into a full relational server.
- **Multi-user concurrent CRUD → KV + API layer**: KV has atomic batches and even full ACID (Fjall WriteBatch atomic commit + crash rollback / SurrealKV full transaction domain); multi-tenant connections, authentication, and concurrent sessions are carried by the API gateway layer, not the storage engine. Rust + embedded KV + API serving externally on the same machine vastly outperforms the traditional "multiple backends + one database" — whose causes vary by language but are none of them structurally necessary:
  - **Python: performance weakness** — interpreted execution, GIL and similar constraints make it hard for backends to efficiently serve high concurrency in-process, hence converging to a single-point database for centralized hosting.
  - **PHP: architectural constraints** — beyond performance issues, no resident business process and stateless short-lived workers cannot hold state in-process, forcing an external shared-state server (first the database, later Redis added for speed) — a major reason for Redis's popularity.
- **Cross-boundary sharing → Lakehouse**: third parties / off-the-shelf BI read data via the **Lakehouse** (Iceberg on S3 as the standard table format), with no need to erect a SQL layer on top of KV — the Lakehouse is already part of this architecture, not an additionally introduced database. The only corner case left: third parties bypassing the API and lakehouse to connect directly via SQL to **online mutable state**.
- **Distribution → consensus for metadata, disaggregated storage for data**: metadata coordination uses consensus (for Raft's principles and limits see §Distributed KV); large-scale data goes SlateDB + S3 (disaggregated, replicas fixed at 3-AZ) or Fjall writing locally + Lakehouse landing in the lake.

**Verdict**: the framework platform's correct posture is to stick with pure KV + composite key encoding. The database is a wrapper above KV, not a replacement for KV. If your query patterns are predictable, Parser + Optimizer is spending runtime CPU to solve a problem that can be eliminated at compile time.

## Engine Comparison and Selection: Architecture · Scenarios · API

A comparison and selection guide for four embedded/adjacent key-value stores (Fjall, SlateDB, redb, SurrealKV) plus Redis and SQLite, developed along three axes: **architecture** (how the engine is organized and deployed), **scenarios** (when to pick which), and **API** (the developer-facing interface form).

### Architecture

Fjall and SlateDB are both pure-Rust LSM-Tree KV engines (Apache-2.0) with similar underlying mathematical logic. But they take opposite extremes on Source of Truth and network topology, corresponding to two completely different deployment models.

#### Engine Positioning Comparison (Fjall vs SlateDB)

| Dimension | Fjall | SlateDB |
|:--|:--|:--|
| **Source of Truth** | Local NVMe/SSD | Cloud object storage (S3/GCS/MinIO) |
| **Flush path** | MemTable → local disk (syscall I/O) | MemTable → S3 (async network write) |
| **Point-lookup latency (cache miss)** | μs-level (local NVMe) | ms-level (S3 Range Get network round trip) |
| **Capacity ceiling** | Limited by local disk | Unlimited (S3 bucket capacity) |
| **ACID transactions** | Mature (3.0+ WriteBatch/Transactions) | Rapidly evolving; advanced transaction controls being completed |
| **Language bindings** | Rust only (cross-language requires a KV server layer, see §Centralized KV) | Official Python / Go / Java / Node / UniFFI (Swift/Kotlin) |
| **Design goal** | Single-machine bare-metal, extreme latency | Cloud-native, stateless nodes |

#### Path One: Fjall (Single-Machine Local Deployment)

```
[Application layer] → [Fjall engine] → [Local NVMe] → return
                            ↓
                      WAL + SSTable
```

**The source of truth is the local disk**. Writes go directly to Fjall in-process, no network loss, returning immediately after the write completes. Latency is determined by NVMe's physical characteristics (μs-level), unaffected by network fluctuations.

**B-tree alternative for the local path**: if local deployment is dominated by point lookups/range scans and you want to eliminate background compaction stalls, choose between Fjall and redb — Fjall (LSM, write-intensive) or redb (COW B-tree, read-deterministic); capability differences are in "Core API Feature Comparison" below.

**ACID batching**: AI Agent scenarios frequently need to atomically modify multiple composite keys (update the conversation main table + update the ranking index + update the tag index). Fjall 3.0's WriteBatch commits atomically in the WAL in one shot, rolling back as a whole on crash, guaranteeing index consistency.

**Capacity ceiling**: data cannot exceed the local high-performance disk. When you need unlimited storage, don't bolt cold-data offloading onto Fjall — use SlateDB directly.

**Cluster deployment**: in multi-node scenarios, metadata is per-node independent (control-plane single-writer, no consensus — aura's ruling; see §Distributed KV chapter and the [Consensus Protocol document](consensus-protocol-en.md) it references). Fjall itself focuses on the local storage engine's responsibilities.

#### Path Two: SlateDB + S3

```
[WS compute node (stateless)] → [SlateDB] → [S3 bucket] → return
                                                    ↑
                                          Source of truth in the cloud
```

**The source of truth is S3**. After data commits, it is pushed directly to S3. S3 itself provides 11 nines of reliability and cross-region replication — multi-node synchronization is physically guaranteed by S3.

**Stateless compute nodes**: multiple Rust WS services connect to the same S3 bucket. After a node crashes, it restarts on a new machine, mounts the same S3 path, and is back serving within seconds. This is the physical basis of Scale-to-Zero — S3 is durable, and compute can appear and disappear at will.

#### Fjall vs SlateDB: Write and Read Path Comparison

**Write path**:

| | Fjall | SlateDB |
|:---|:---|:---|
| Write target | Local NVMe (MemTable → WAL) | MemTable (in memory) → async flush to S3 |
| MemTable write | μs (local) | μs (local, same as Fjall) |
| Flush | Background writes local SSTables | Background writes S3 SSTables |
| Write latency (user-perceived) | μs (MemTable) | μs (MemTable); flush is async and non-blocking |

**The two write paths are essentially the same**: both accumulate batches in the MemTable → async flush to SSTables. The only difference is the flush target (local vs S3); flush is a background async operation and does not block writes. The write-latency gap is smaller than commonly believed.

**Read path**:

| | Fjall | SlateDB |
|:---|:---|:---|
| Hot data | Local SSTables → μs | Local cache (block cache + SST cache) → μs |
| Cold data | Local disk → μs | S3 GET → ms |
| Data volume ceiling | Local disk size | Unlimited (S3) |

**The core difference is not performance but deployment model**:

| | Fjall | SlateDB |
|:---|:---|:---|
| S3 dependency | None (works offline) | Required |
| S3 API cost | None | Yes (PUT/GET/transfer fees) |
| Storage ceiling | Local disk | Unlimited |
| Replication | Handle it yourself (Lakehouse lake backup; metadata consensus see §Distributed KV) | Handled inside S3 |
| Operations | Self-managed disks | Stateless compute + S3 |

#### Fjall's Differentiating Advantages (Even if SlateDB Gains Local Storage Support)

Even if SlateDB perfectly supports local storage in the future, Fjall retains structural advantages in the following areas:

**1. KV separation (Value Log)**: Fjall has a built-in `value-log` component (inspired by RocksDB's BlobDB/Titan). When writing large values (images/documents/audio), values are stored separately in dedicated files; the LSM-Tree keeps only key + pointer. This greatly reduces write amplification; for large-object scenarios, local write and Compaction performance far exceed SlateDB's.

**2. More mature transaction support**: Fjall has built-in serializable transactions and optimistic/single-writer transaction models (`OptimisticTxDatabase` / `SingleWriterTxDatabase`). For multi-concurrent local transaction control and atomic commits across multiple Keyspaces, it is closer to a mature local RDBMS core.

**3. Extreme local optimization**: Fjall 3.0 completely reworked the local-disk block format — sparse indexing, prefix truncation, partitioned Bloom filters, optional hash index. On cache miss, local-disk random point lookups and range scans are 2-100x faster, with extremely low memory overhead.

**4. Purebred lineage**: SlateDB's core design is "Zero-Disk" (zero local-disk dependency); its concurrency locks, fencing, and flush strategies are all optimized around network object-storage latency. Even with local-write support, these architectural burdens (e.g., instant-durability delays caused by aggressive batching adapted to the network) are hard to fully erase. Fjall is designed 100% for local NVMe/SSD throughput and the OS filesystem.

**5. The cross-language answer is servitization, not bindings**: SlateDB officially provides Python / Go / Java / Node / UniFFI bindings; polyglot teams can embed the same source of truth directly — something Fjall structurally cannot offer (Rust only). But Fjall's answer is not to switch engines; it is KV server assembly (see §Centralized KV): Fjall + Tokio + WS turns the embedded engine into a network service accessible from any language, while the in-process hot path still retains zero-RTT direct calls. The cost is one more network access layer and ops surface; the benefit is that the Python/Java side gets a cross-process standardized interface with even higher binding-layer API completeness.

#### SlateDB's Local-Disk Mode

SlateDB is built on the `object_store` crate, and `LocalFileSystem` is a legitimate implementation — the in-memory variant in examples is just demo convenience; the local-disk path works out of the box, making deployment flexible (local disk ≈ Fjall's deployment position, S3 = disaggregated storage). But local mode has one structural limitation: SlateDB's concurrency mutual exclusion (fencing, put-if-absent) is designed for object-storage semantics; the local filesystem's mutual-exclusion primitives are weaker than S3's, and the officially recommended posture is an external lock (DynamoDB / Postgres lock) to guarantee a single writer. Single-process use is fine; multi-process concurrent writes to a local path are unsafe.

#### Fjall's S3 Plans

**There is no official plan for native S3 support**. Fjall is positioned as an embedded single-machine storage engine (a pure-Rust RocksDB/LevelDB), keeping the core library lightweight, deterministic, and 100% Safe Rust. The community has indirect solutions: bridging Apache OpenDAL through a VFS, or using `s5_store_fjall` to map the underlying storage to S3.

### Scenarios

#### Four-Engine Horizontal Selection (Fjall / SlateDB / redb / SurrealKV)

| Scenario | Recommendation | Rationale |
|:--|:--|:--|
| API gateways, high-concurrency middleware | **Fjall** | Partitions give precise physical division; local disk bursts to hardware limits |
| Serverless AI Agents, cloud-native knowledge bases | **SlateDB** | Purely async, blends into Tokio, batches and pushes to S3, scales to zero |
| Read-deterministic local services without background stalls | **redb** | COW B-tree has no compaction; deterministic read path for point lookups/range scans |
| Concurrent accounting, historical version rollback | **SurrealKV** | MVCC transactions + timestamp queries, saving thousands of lines of application-layer version bookkeeping |

#### Single-Machine Deployment: Cost Comparison

**Hardware costs (2026 market prices)**:

| Resource type | Unit price | Redis typical usage | Fjall typical usage | Cost difference |
|---------|------|---------------|---------------|---------|
| DRAM | ~$5/GB | 100GB dataset = 500GB RAM (incl. replication buffers, expired keys) | 100GB dataset = 30-50GB RAM (indexes + cache) | **10x** |
| SSD | ~$0.10/GB | N/A (pure in-memory) | 100GB dataset = 30-50GB disk (LZ4 compressed) | **N/A** |
| CPU | ~$50/core | Single-threaded model; high concurrency needs vertical scaling | Multi-threaded concurrency, horizontal scaling | **5-10x** |

**TCO analysis (3-year horizon, 100GB dataset)**:

| Cost item | Redis | Fjall |
|--------|-------|-------|
| Hardware (server) | $15,000 (512GB RAM) | $2,000 (64GB RAM + 1TB NVMe) |
| Ops labor | $30,000 (configuring RDB/AOF, monitoring big keys, failure recovery) | $5,000 (zero config, automatic compaction) |
| Network bandwidth | $10,000 (cross-process communication, cluster sync) | $0 (in-process calls) |
| **Total** | **$55,000** | **$7,000** |

**Conclusion**: Fjall's TCO is **1/8** of Redis's.

#### Single-Machine Deployment: Operational Complexity

| Dimension | Redis | Fjall |
|------|-------|-------|
| **Deployment** | Standalone process + config file + persistence strategy | Embedded in the application, zero config |
| **Persistence** | Must manually choose RDB/AOF, configure save policies, handle fork blocking | Automatic WAL + SSTable, background compaction |
| **Monitoring** | Must monitor memory usage, big keys, slow queries, connection counts | No separate process; application-level monitoring suffices |
| **Failure recovery** | RDB recovery is slow (minutes); AOF risks data loss | WAL + SSTable automatic recovery, seconds |
| **Scaling** | Manual resharding; cluster instability | Cluster mode auto-syncs data (cross-node requires consensus, see §Distributed KV) |
| **Big-key problem** | Single-thread blocking; must split or delete asynchronously | Multi-threaded concurrency, no blocking risk |

**Quantified ops burden**:

- Redis: 2-4 hours per week (monitoring alerts, persistence tuning, big-key cleanup)
- Fjall: 1 hour per month (log checks, disk space monitoring)

#### Single-Machine Deployment: Selection Decision

| Scenario characteristic | Recommendation | Rationale |
|---------|---------|------|
| **Data < 10GB, read-heavy/write-light** | Fjall | In-process zero RTT, controllable memory usage |
| **Data > 100GB, persistence needed** | Fjall | LZ4 compression; SSD cost far below DRAM |
| **High concurrency (>10K QPS)** | Fjall | Multi-threaded concurrency; Redis's single-thread bottleneck |
| **Distributed locks needed** | Fjall + consensus layer | In-process atomic ops + consensus-layer strong consistency (see §Distributed KV); Redlock is mathematically unsafe |
| **Cross-process shared state (polyglot)** | Redis or SurrealDB | Fjall is an embedded library and cannot cross processes |
| **Caching (loss acceptable)** | In-app HashMap / Caffeine | Faster than Redis, simpler than Fjall |
| **Pub/Sub, Streams needed** | NATS / Kafka | Redis's messaging features are weak, no persistence |
| **Complex data structures (Geo, HLL)** | PostGIS / dedicated libraries | Redis's memory cost is too high |
| **Open-source framework/CLI internal state management** | Fjall | See "SQLite vs Embedded KV" below; C dependencies/write locks/double caching are systemic wear |

**Decision flowchart**:

```
Need cross-process/cross-language sharing?
├─ Yes → Redis or SurrealDB
└─ No → Data > 100GB?
         ├─ Yes → Fjall (SSD cost advantage)
         └─ No → Persistence needed?
                  ├─ Yes → Fjall (automatic WAL)
                  └─ No → Loss acceptable?
                           ├─ Yes → HashMap / Caffeine
                           └─ No → Fjall (in-memory mode)
```

#### Single-Machine Deployment: Performance Pitfalls

**Redis's hidden costs**:

1. **Serialization overhead**: 1-5μs per request (JSON/Protocol Buffers); 10K QPS = 10-50ms/s of CPU time
2. **Context switches**: inter-process communication triggers kernel-mode switches, ~1μs each
3. **Network stack**: TCP/IP protocol stack processing ~10-50μs per packet
4. **Memory fragmentation**: Redis uses jemalloc; after long runs, fragmentation runs 10-30%

**Fjall's advantages**:

1. **Zero serialization**: pass Rust struct references directly in-process
2. **Zero context switches**: function calls, no kernel-mode switches
3. **Zero network stack**: no TCP/IP processing
4. **Compressed storage**: LZ4 compression reduces data volume by 50-70%, less disk I/O

**Measured data (100GB dataset, 10K QPS)**:

| Metric | Redis | Fjall |
|------|-------|-------|
| P50 latency | 0.8ms | 0.05ms |
| P99 latency | 5ms | 0.2ms |
| CPU usage | 80% (single-thread saturation) | 30% (spread across threads) |
| Memory usage | 120GB | 8GB |
| Disk usage | 0GB | 35GB (after compression) |

#### Single-Machine Deployment: Migration Cost

**Effort to migrate from Redis to Fjall**:

| Task | Effort | Risk |
|------|--------|------|
| Key encoding scheme implementation | 1-2 days (AI-generated) | Low (patterned code) |
| State machine `apply` logic | 2-3 days (AI-generated + review) | Medium (edge cases need verification) |
| Data migration script | 1 day (Redis DUMP → Fjall import) | Low (one-time task) |
| Integration testing | 2-3 days (AI-generated cases) | Medium (must cover all Redis commands) |
| Production deployment | 1 day (replace startup scripts) | Low (embedded, zero ops) |
| **Total** | **7-10 days** | **Manageable** |

**Migration benefits (3-year TCO)**:

- Hardware cost savings: $13,000 × 3 = $39,000
- Ops cost savings: $25,000 × 3 = $75,000
- **Total savings: $114,000**

**ROI**: migration cost $5,000 (labor) → 3-year benefit $114,000, **ROI = 22.8x**.

#### SQLite vs Embedded KV

SQLite is a miracle of software engineering, but many projects adopt it simply because they want "single-file, ops-free, locally persistent" storage — not because they truly need relational algebra and a SQL optimization engine. Its advantages on the following three dimensions come from engine-implementation-level ongoing overhead comparisons and are unrelated to whether query patterns are predictable (what query patterns determine is whether to solidify a key-path pattern library; see §SQL Translation Layer vs KV Pipeline):

**C language dependencies and cross-compilation**. SQLite is written in C. Once a Rust project pulls in the rusqlite bindings, user machines must have a C compiler installed (gcc/clang). For cross-compilation (Mac → Linux ARM64), the C toolchain is the main blocker. A pure-Rust KV engine (Fjall) compiles into a statically linked single binary in seconds, with zero external dependencies.

**Double caching and memory waste**. SQLite has an internal Page Cache. Data goes disk → SQLite Page Cache → SQL-parsed row structures → a second copy into Rust object memory. A pure KV engine's LSM-Tree Block Cache maps directly to the application layer — a shorter read path and lower memory usage.

**Write-lock thread blocking**. SQLite uses a database-level exclusive lock for writes. In high-concurrency multi-threaded scenarios (gateways, Agent services), `SQLITE_BUSY` fires frequently and threads hang waiting. Pure-Rust KV engines absorb concurrent writes through a lock-free MemTable (skiplist/radix tree), with multi-core parallelism and no blocking.

##### CLI Tool Scenario: Fjall vs SQLite

| | Fjall | SQLite |
|:---|:---|:---|
| Embedded | ✅ In-process, zero config | ✅ In-process, zero config |
| Single file | ❌ Directory (multiple SSTables) | ✅ Single .db file |
| ACID | ✅ WAL | ✅ WAL |
| Write performance | Better (LSM-Tree, no write lock) | Worse (B-Tree, write-lock contention) |
| Concurrent writes | Good (multi-threaded, lock-free) | Poor (single writer) |
| SQL | ❌ Pure KV | ✅ |
| Cross-language | Rust only | C/Python/Go/Node — all languages |
| Backup | Copy the directory | Copy the single file |

**When to pick Fjall**: pure-Rust CLIs, data is key-value (cache/index/config/state), write-intensive. The LSM-Tree's write performance is an order of magnitude above SQLite's B-Tree + write lock.

**When to still pick SQLite**: SQL queries needed (JOIN/aggregation), single-file needed (copying the `.db` is the backup), cross-language bindings needed, mature ecosystem needed (ORM/GUI clients/migration tools).

**Verdict**: for a pure-Rust CLI tool, Fjall is the better replacement for SQLite — zero C dependencies, no write lock, faster writes. If it's not pure Rust, or you need SQL or cross-language support, SQLite remains the more pragmatic choice.

##### Open-Source Infrastructure Case Studies

| Project | Choice | Rationale |
|:--|:--|:--|
| **Docker / containerd** | bbolt (Go KV) | Container metadata queries are fixed (Container_ID → metadata); KV suffices; SQL is needless overhead |
| **K3s (edge)** | Converged from SQLite toward etcd embedded KV | Edge nodes are CPU/memory sensitive; SQL parser jitter is intolerable |
| **3D asset pipeline (orbsh/wiki)** | LanceDB (columnar/KV) | Asset metadata path-lookup patterns are predictable; relational multi-table resolution is a performance trap |

**Refactoring demonstration**: SQLite config table `configs(app_name, config_key, config_value)` → KV composite key:

```
Key: cfg:{app_name}:{config_key}  →  Value: [raw binary]
```

`save_config` = one `put`, no SQL parsing. `get_all_app_configs` = one `prefix_scan("cfg:{app_name}:")`, no query plan generation. The code is the most efficient execution plan — the LSM-Tree's lexicographic iterator scans the SSTable sequentially.

**Verdict**: SQLite is the business system's "all-purpose compromise"; embedded KV is open-source infrastructure's "iron-law standard." Pure-Rust CLI tools pick Fjall; cross-language/SQL needs pick SQLite.

**The narrowing boundary between SQLite and KV**: capability-wise, both can carry data with predictable access (Chrome using both SQLite and LevelDB is ecological-niche history, not an architectural judgment); code volume reaches parity once index/schema patterns are solidified; migration costs exist on both sides — PG requires dump/restore per major version, SQLite carries the historical baggage of thirty years of format compatibility, KV needs self-built version migration tooling — differing only in who tooling falls on (engine author vs user). The real boundary reduces to one allocation of engineering economics: if users are yourself → a one-time investment in migration tooling in exchange for zero query-layer overhead (KV); if users are unknowable and never breaking the format is a promise → SQLite as the fallback. The boundary criterion has been the same all along: **whether the query pattern is predictable** — what it determines is whether to solidify key paths, not whether to use KV.

#### Selection Criteria (Fjall vs SlateDB)

```
Can you use S3?
│
├── Yes → SlateDB (the default choice)
│   Unlimited storage, S3 handles replication, simple ops
│
└── No → Fjall
    On-premise deployment / offline environments / S3 costs unacceptable
```

**Verdict**: SlateDB is the default choice for most scenarios. Fjall gradually becomes a niche choice — pick it only when there is an explicit "no S3" requirement (on-premise, offline, cost-sensitive). The performance gap between the two is smaller than commonly believed: writes are both MemTable batching, hot-data reads are both local caches. The real gap is in deployment model and storage cost.

Strong-consistency distributed needs ("sharding + Raft") are **not self-built** — use FoundationDB / TiKV; for principles and selection see the **§Distributed KV** chapter above.

#### Selection Criteria (Updated)

```
Can you use S3?
│
├── Yes → SlateDB (the default choice)
│   Unlimited storage, S3 handles replication, simple ops
│
└── No → Fjall
    On-premise deployment / offline environments / S3 costs unacceptable
    ↓
    Fjall's advantages are clear when you need:
    • Large-value scenarios (KV separation reduces write amplification)
    • Complex local transactions (serializable / multi-Keyspace atomic commits)
    • Extreme local performance (sparse index / hash index / Bloom filters)
```

The principles and selection for strong-consistency distribution (sharding + Raft / FDB / TiKV) are unified in the **§Distributed KV** chapter above; only the embedded duo (Fjall vs SlateDB) comparison is kept here. For the Agent memory system implementation see [Use Cases](#use-case-kv-implementation-for-an-agent-memory-system); for the gateway implementation see [Use Cases](#use-case-openresty--kv-gateway).

### API

#### API Pseudocode Feel

##### Fjall: Traditional Industrial-Grade Cascading API

Pursues extreme granularity of control over the local physical disk. Introduces the concepts of `Keyspace` (large namespace) and `Partition` (physically isolated partition).

```rust
// 1. Open the local large-disk space
let keyspace = fjall::Config::new(db_path).open()?;

// 2. Open an independent physical partition (equivalent to RocksDB's Column Family)
let user_table = keyspace.open_partition("users", fjall::PartitionCreateOptions::default())?;

// 3. The classic atomic Batch write
let mut batch = keyspace.batch();
batch.insert(&user_table, b"cfg:app:1", b"payload_bytes");
batch.commit()?;
```

##### SlateDB: Cloud-Native Fully Async API

Its core soul is S3; all APIs are natively and thoroughly asynchronous (`async/await`), and initialization binds directly to a network object bucket.

```rust
// 1. Initialize the cloud object-storage driver
let object_store = object_store::aws::AmazonS3Builder::from_env().build()?;
let path = "my_agent_bucket/db_root".to_string();

// 2. Open the cloud-native KV instance
let db = slatedb::Db::open_with_opts(path, slatedb::DbOptions::default(), Arc::new(object_store)).await?;

// 3. Thoroughly async read/write API
db.put(b"cfg:app:1", b"payload_bytes").await?;
```

##### SurrealKV: Aggressive ACID Transaction-Level API

Built for large database transactions. All read/write APIs must be wrapped in a strict `Transaction` (transaction closure).

```rust
// 1. Open the pure-Rust embedded local engine
let kv = surrealkv::Store::new(surrealkv::Options::new(db_path))?;

// 2. Explicitly begin a writable transaction
let mut tx = kv.begin_rw()?;

// 3. All operations bind to the tx transaction context
tx.set(b"cfg:app:1", b"payload_bytes")?;

// 4. Explicit commit. On failure, physical rollback happens automatically
tx.commit()?;
```

##### redb: B-Tree Transaction-Domain API

A pure-Rust Copy-on-Write B-tree. No background compaction, deterministic read path; naturally suited to workloads where "point lookups/range scans come first and writes aren't massive." Logical multi-table (TableDefinition) organizes namespaces; transactions use borrow lifetimes to express the single-writer constraint.

```rust
// 1. Open the B-tree database (multiple logical tables can be defined within one db)
let db = redb::Database::create(db_path)?;

// 2. Define logical data tables (each TableDefinition is an independent B-tree)
let user_table: TableDefinition<&str, &[u8]> = TableDefinition::new("users");

// 3. Read/write transaction; borrow lifetimes guarantee a single writer
let write_tx = db.begin_write()?;
{
    let mut table = write_tx.open_table(user_table)?;
    table.insert(b"cfg:app:1".as_slice(), b"payload_bytes")?;
}
write_tx.commit()?;
```

#### Core API Feature Comparison

| Feature dimension | Fjall (3.x) | SlateDB | redb | SurrealKV |
|:--|:--|:--|:--|:--|
| **Data structure** | LSM-Tree | LSM-Tree | Copy-on-Write B-tree | LSM-Tree |
| **Async** | ❌ Purely sync, needs `spawn_blocking` | 🚀 Purely async `.await`, blends into Tokio | ❌ Purely sync | ❌ Purely sync |
| **Multi-space isolation** | 🥇 Partitions, physical | ◐ Flat (no physical partitions) | ◐ Logical multi-table (TableDefinition, not physical partitions) | ◐ Flat (no physical partitions) |
| **Transactions** | WriteBatch atomic batch | Basic atomic batch writes | Serializable (borrow-lifetime single writer) | 🥇 Strict MVCC transactions |
| **Large values** | 🥇 WiscKey KV separation | Early evolution | ⚠️ No KV separation | 🥇 Blob Log large-object separation |
| **Time travel** | ❌ | ❌ | ❌ | 🥇 Versioned Queries |

#### Unified Abstraction

The four engines' API differences can be shielded behind a unified Trait abstraction (see the `AuraStorage` trait). Key decision points: async `async` (→ SlateDB), transactional blocks with sync (→ SurrealKV), stall-free deterministic local reads (→ redb). The unified Trait lets one set of composite Key structs switch seamlessly among the four engines.

## Use Case: KV Implementation for an Agent Memory System

AI Agent applications (agent memory stores, long/short-term context management, multi-turn conversation history retrieval) have highly fixed read/write patterns — conversation history scanned along the timeline, state snapshots fetched by exact point lookups, tag-entity secondary indexes. Pure KV's composite key encoding maps directly onto these three patterns, with no ORM or query parsing layer.

### Fixed Read/Write Patterns

| Pattern | Business scenario | Key encoding | Query method |
|:--|:--|:--|:--|
| **Timeline scan** | Load the most recent N messages of a session | `mem:{agent_id}:{session_id}:{ts_be}:{msg_id}` | Prefix Seek + iterator; reverse via `SeekForPrev` or inverted-timestamp encoding |
| **Exact point lookup** | Read an Agent's System Prompt / long-term memory summary | `agent:profile:{agent_id}:{memory_type}` | Single Get, μs-level block-cache hit |
| **Secondary index** | Retrieve conversation fragments by keyword | `idx:tag:{entity}:{agent_id}:{ts}` → `{session_id}:{msg_id}` | Prefix Seek scan of index IDs, then point lookups to fetch originals |

**Reverse loading of conversation history**: when an LLM loads recent conversation it needs reverse order (newest messages first). Two implementations: (1) the `SeekForPrev` reverse iterator (Fjall supports it); (2) encode the timestamp as `u64::MAX - timestamp` so the newest message's key is lexicographically smallest and a forward iterator naturally reads in reverse. The latter is compatible with all KV engines that support only forward scans.

**Natural hot/cold tiering**: the access pattern of Agent conversations — recent conversations re-read every turn (hot data), historical conversations untouched for months (cold data) — naturally matches the LSM-Tree's physical structure. Hot data resides in the MemTable + OS Page Cache (memory-level latency); cold data automatically settles into SSTables (SSD storage). No manual tiering or cache policy configuration needed.

**CPU overhead comparison vs an ORM**: frameworks like LangChain/LlamaIndex wrap storage with an ORM + connection pool + SQL parsing layer. Reading a chat transcript goes through: SQL string parsing → query plan generation → process/thread lock contention → deserialization of data between the network driver and the application layer. Pure KV's read path is: prefix match → contiguous bytes from disk/cache handed directly to the deserializer. In scenarios where the LLM must assemble tens of thousands of tokens of context per turn, the ORM overhead saved accumulates considerably.

### The Boundary of Vector Retrieval

Pure KV's key encoding can emulate all of Redis's data structures, but it **cannot do vector similarity search at the key level** — the LSM-Tree's lexicographic ordering is meaningless for high-dimensional vectors. In Agent applications, vector retrieval (RAG) is a separate architectural layer:

**Embedded vector index + KV storage separation**: introduce a lightweight vector index library in the Rust service (`hnsw-rs`, `faiss-rs`), loaded into memory at startup. The actual text content still lives in the local KV. Retrieval path: vector index returns IDs → KV point lookup for the original text. Zero external dependencies; no standalone vector database needed.

**Natively multi-model engine**: SurrealDB natively stacks a vector index and graph pointers on top of SurrealKV, completing vector retrieval and KV storage within one engine. The cost is introducing the full SurrealQL query layer — a heavyweight solution for a pure-KV scenario.

**Selection criteria**: if the Agent only needs keyword + timeline scans → pure KV suffices. If semantic similarity retrieval is needed → embedded vector index or SurrealDB. Both are an order of magnitude lighter than deploying Milvus/Pinecone separately.

## Use Case: OpenResty + KV Gateway

OpenResty remains the optimal solution for handling HTTP (LuaJIT + cosocket non-blocking I/O). The change is replacing Redis with a local Rust Sidecar + Fjall, communicating over a Unix Socket — network RTT drops to μs-level inter-process calls.

### Architecture

```
[Client] → [OpenResty (Lua, HTTP layer)] → [KV Sidecar (Rust, Unix Socket)]
                                            ↓
                                        [Fjall LSM-Tree]
```

The sidecar protocol is minimal: length-prefixed 4-byte binary frames `[len:4][op:1][key_len:4][key:N][value_len:4][value:M]`, with four operations Get / Put / Delete / PrefixScan. OpenResty connects to the Unix Socket via `ngx.socket.tcp`, with cosocket non-blocking.

### Composite Key Orchestration

| Domain | Key encoding | Value | Query pattern |
|:--|:--|:--|:--|
| **Rate limiting** | `rl:{client_id}:{fixed_window_ts}` | `count` (i64) | Point lookup of current window + prefix-scan aggregation |
| **Response cache** | `cache:{method}:{path_hash}:{etag_hash}` | `{status, headers, body, created_at}` | Exact-match point lookup + prefix-scan batch invalidation |
| **Sessions** | `sess:{user_id}:{session_id}:{field}` | Field value | Prefix scan to load all fields |
| **Circuit breaking** | `cb:{upstream_id}` | `{state, failure_count, last_failure}` | Pure point lookup |
| **Dynamic routing** | `route:{method}:{priority_zp}:{path_pattern}` | `{upset, timeout, retry}` | Prefix scan by method + big-endian priority ordering |

### Key Design Points

**Rate limiting: fixed window vs sliding window**. Fixed window `rl:192.168.1.1:1722500000` (current 10-minute window Unix timestamp) point-lookup counts, rejecting on exceed. A prefix scan `rl:192.168.1.1:` can aggregate multiple windows for a sliding window, but the fixed window is sufficient for gateway scenarios and an order of magnitude faster.

**Caching: prefix-scan batch invalidation**. Traditional Redis uses `SCAN` cursor iteration for path-prefix invalidation (blocking the single thread). KV's prefix scan is a native LSM-Tree capability — the iterator `Seek("cache:GET:")` locates directly, batch Delete writes Tombstones, and foreground requests are not blocked.

**Routing table: ordered one-shot load**. In `route:GET:0010:/api/v1/*` the priority is zero-padded to 4 digits so lexicographic order = numeric order. At startup OpenResty loads the full routing table into the nginx shared dictionary with one prefix scan; at runtime there are zero KV accesses.

### Differences from the Old Redis Gateway Design

The old design's (OpenResty + Redis) composite keys — static/dynamic path separation, prefix-match/exact-match tiering — are equally necessary in KV. This is not a regression; it is the same key-space orchestration philosophy naturally continued on a better engine. The physical-level differences:

| Dimension | Old design (Redis) | New design (KV Sidecar) |
|:--|:--|:--|
| Rate-limit + cache writes | Two network requests, no atomicity | Atomic Batch committed together |
| SCAN batch invalidation | Blocks the single-threaded event loop | LSM-Tree background compaction, non-blocking |
| Restart recovery | RDB/AOF, minutes | WAL + SSTable, seconds |
| Concurrency | Single-threaded serial | Tokio multi-threaded, multi-core parallel |
| Memory usage | Everything resident in RAM | Hot data in MemTable, cold data on SSD |

## Case Study: OpenAI's Architecture Evolution — PG → KV Migration

In early 2026, OpenAI disclosed details of the underlying architecture supporting its 800 million users. This case directly validates this document's core thesis: when query patterns are predictable, KV beats relational databases.

### The Core Query Pattern = Pure KV

The OLTP hot-data queries on the ChatGPT platform are extremely monotonous:

| Business | Key | Value | Query pattern |
|:--|:--|:--|:--|
| User accounts | `user_id` | Account/Token billing/subscription status | Exact point lookup |
| Session list | `user_id` | List of session IDs | Prefix scan |
| Conversation history | `session_id:message_id` | Conversation text JSON/Protobuf | Prefix scan + reverse |

No cross-table JOINs, no complex relational transactions. From the physical essence, these three models are KV's standard use cases — an industrial-scale validation of the Agent use case.

### Why PostgreSQL Was Chosen

Not because the architecture was optimal, but because in 2023 the Rust ecosystem was immature — SlateDB had not yet been born, lightweight KV consensus libraries were still iterating, and the only mature choice was the heavyweight TiKV. Under the pressure of ChatGPT's traffic explosion, PG was the "can't-be-blamed" safe choice: 40 years of industrial validation, absolutely no data loss, strict Schema governance.

The cost: billions of dollars in compute budget + a world-class DBA team tuning around the clock + PgBouncer connection pooling + Redis multi-layer caching interception + most multi-table JOINs banned in production. This is a textbook case of using expensive engineering labor to paper over a mismatch in the underlying storage paradigm.

### PG's Physical Fatal Flaw: MVCC Write Amplification

As the user scale pushed toward 800 million, PG's MVCC mechanism became the bottleneck. Every write (conversation history Insert/Update) creates a new Tuple on disk, producing large numbers of Dead Tuples → Autovacuum running furiously → disk I/O and CPU saturated by write operations.

This is not a PG bug; it is the physical cost of relational MVCC design — in write-intensive scenarios, garbage-collection overhead grows linearly with data volume.

### OpenAI's Migration Path

Write-intensive, relation-free business is being migrated to KV:

| Migrating out of PG | Migrating into KV | Rationale |
|:--|:--|:--|
| Conversation history context | Distributed KV / object storage | Write-intensive, predictable read patterns, no JOINs |
| Session logs | KV | Time-series appends, prefix scans |
| AI state flags | KV | High-frequency read/write, no relational constraints |

PG degenerates into a **metadata safety gate** — handling only small-volume core sources of truth such as user accounts, organization permissions, and purchase orders.

### Validation of This Document's Theses

| This document's thesis | OpenAI's practical validation |
|:--|:--|
| When query patterns are predictable, KV beats SQL (see "When to Use SQL") | Core traffic = pure KV point lookups + prefix scans |
| PG's MVCC is a physical fatal flaw in write-intensive scenarios | At 800M users, Dead Tuples → Autovacuum explosion |
| Composite key encoding replaces relational tables (see "Physical-Layer Encoding Paradigms") | Conversation history = `session_id:message_id` composite key |
| Modern KV engines can now be trusted | OpenAI is migrating; the 2026 ecosystem is mature |

**Verdict**: OpenAI stayed on PG for 3 years because in 2023 there were no mature lightweight Rust KV wheels. In 2026, blindly copying the PG route is anachronistic. Going directly with Fjall or SlateDB+S3, delegating the distributed edge cases to mature Rust infrastructure, is the modern path with the lowest labor cost.

## Cross-References

[Redis Critique](redis-critique-en.md) argues **why Redis fails at every layer**:
- L0 (in-process): Redis is 200-50,000x slower than local memory → **Fjall is exactly an L0 implementation with persistence**
- L3 (distributed coordination): Redis has no consensus → **for the consensus protocol proposal see the [Consensus Protocol document](consensus-protocol-en.md)**

This document is the **constructive counterpart**: not just "Redis is bad," but "here is the precise architecture to replace it."

**Specific citations from the critique document:**
- Critique §Distributed Locks: says "building it yourself is simple" → this document shows Fjall's in-process lock implementation; for the distributed-lock proposal see the [Consensus Protocol document](consensus-protocol-en.md)
- Critique §Cluster Myth: says Redis lacks strong consistency → the [Consensus Protocol document](consensus-protocol-en.md) fills that gap with a validated consensus solution

→ For the SQL comparison argument see [SQL Translation Layer vs KV Pipeline](#sql-translation-layer-vs-kv-pipeline-chain-a-decisive-advantage-under-predictable-query-patterns) and [KV's DDL](#kvs-ddl-index-management-and-field-evolution); for the abstraction-versus-performance trade-off see [Prefix Scan vs SQL](#prefix-scan-vs-sql-the-promise-and-its-price), whose opposite (index order as a leak) is argued in the [MySQL Critique](mysql-critique-en.md), "Index Rigidity".

[Consensus Protocol](consensus-protocol-en.md) details **the essence and limits of Raft as a metadata consensus protocol**: why Raft suits etcd/K8s's MB-scale metadata but not GB-scale data storage (the physical reality of 3x write amplification).