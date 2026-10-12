# SurrealDB Evaluation Dossier: SurrealQL and Query Language Design

> **Languages:** [English](query-language-design-en.md) (primary) · [中文](query-language-design.md)
> The core architecture did **not adopt** SurrealDB (see [Unified Data Layer](unified-data-layer-en.md)) — its graph/multi-model/compute-pushdown requirements were absorbed by PG (JSONB/extensions) and KV, while analytical loads went to DuckDB·Lakehouse. But the query-language insights that SurrealQL carries — native composability, compute pushdown, anti-ORM — hold independently of SurrealDB: even if SurrealDB does not enter the core, these arguments about "what a query language should look like" remain valid. The complete evaluation follows.

## The Fundamental Flaws of SQL

SurrealQL is at once SurrealDB's greatest weakness and its strength. The weakness is the learning cost (though keeping SQL-like patterns lowers the barrier); the strength is a **redefinition of the query language**.

SQL was originally positioned for business analysts and DBAs, imitating the declarative grammar of natural English so that people who don't program could still query data. But once it was dragged into application development, this grammar designed for "non-programmers" became a developer's nightmare. The mathematical core is complete, but the engineering shell is catastrophically unfriendly to developers.

### Language Flaws

**Syntactic order contradicts execution order**: SQL's logical execution is FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY, but the written order puts `SELECT` at the very front. To imitate natural English, the declarative grammar places the core operation `FROM` at the end and the result-set definition `SELECT` at the start. This non-linear writing enormously raises the cognitive burden in multi-level nested subqueries — you must first work out the data flow, then assemble the statement backwards.

**The disaster of three-valued logic (NULL)**: SQL inherited Codd's three-valued logic. In normal two-valued logic a condition is either true or false. But SQL, because of NULL (unknown), introduces a third state: Unknown. So `1 = NULL` evaluates to NULL, and `NULL = NULL` is also NULL. When writing Rust, the type system forces you to handle the None case of `Option<T>`; but in SQL, one careless moment lets NULL swallow data that should have matched — the semantic trap of `NOT IN (SELECT col FROM t WHERE col IS NULL)` is a classic case, where the entire subquery returns an empty result because of one NULL row.

### Architectural Flaw: The Cascade of Disasters from the Lack of Composability

SQL lacks composability — there is no control-flow structure (loops, conditionals), you cannot compose logic within a query, and you cannot reuse query fragments the way you write ordinary code (CTE and window functions are after-the-fact patches). This is the architectural root cause, and it triggers a cascade of disasters:

**Data movement**: Because logic cannot be composed within a query, developers are forced to pull data up to the application layer for orchestration, creating a mass of unnecessary network round-trips and data movement. Embedding generation is a typical case — under SQL's architecture, the application layer calls an external API → gets the result → writes it back to the DB; the two movements are determined by the architecture, not by the developer writing it wrong.

**ORM is the "Go language" of the data world**: The lack of composability makes SQL hard to express business logic directly, and ORM caters to this cognitive laziness, trying to map relational algebra down into an object graph. The result is that, to paper over the impedance mismatch, the ORM generates extremely inefficient SQL, creating the N+1 query disaster — and in the end the developer has to go back and hand-write native SQL to tune it.

ORM is to SQL what Go is to systems programming — by crippling the expressiveness of the underlying relational model it trades for a false comfort at the object-oriented level, and ultimately leaves the performance disaster to runtime. See [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md).

---

## SurrealQL's Architectural-Level Solutions

### 1. Eliminate Application-Layer Orchestration and Data Movement (Compute Pushdown)

Allow complex business logic to be written directly at the database layer (e.g. `DEFINE TRANSACTION` to implement atomic transfers), pushing computation down to the data layer for execution. Move the "Read-Modify-Write" cycle directly into the storage layer, so complex business rules execute atomically in a single request, significantly reducing network round-trips (I/O) from backend to database.

### 2. Native Composability (First-Class Queries)

SurrealQL solves the engineering pain of SQL's long-standing lack of composability. It treats queries as **values** rather than text. Variables can be piped directly into subsequent operations (`LET $x = SELECT ...; $x | update...`), eliminating dangerous string interpolation and ensuring type safety.

### 3. Multi-Model Unification and JOIN-Free Graph Traversal

Combines the flexibility of a document database with the relational expressiveness of a graph database. It supports direct links between records (`->knows->`), without traditional multi-table JOINs, avoiding the N+1 query problem and complex JOIN syntax.

### 4. Native Real-Time Push (Live Queries)

One of the most disruptive advantages. Via WebSocket, clients subscribe directly to data-change events (`LIVE SELECT`), removing the complexity of having to introduce message-queue middleware such as Kafka or Redis Pub/Sub in traditional architectures.

### 5. "The Third Way" (Anti-ORM)

The industry swings between rigid stored procedures (hard to extend/maintain) and heavy ORMs (N+1 queries, serialization overhead, network RTT bottlenecks). SurrealDB bridges this gap: it offers **distributed scalability** to support logic-in-DB (unlike Oracle), while also offering **native execution** that eliminates the ORM's serialization and network costs.

### 6. Modern Syntax (Pan-Rust Lineage)

SurrealQL shares DNA with modern systems languages such as **Nushell** and **Moonbit**. Concise and ergonomic, avoiding SQL's verbose declarative boilerplate, allowing imperative logic (loops, conditionals) to be integrated naturally. See [Modern Language Design](modern-language-design-en.md).

---

## Reference Frame for Data Interaction: SQL / DataFrame / SurrealQL / KV Key Space

The essential difference among these four ways of interacting with data lies in where composition happens and in generality:

| Dimension | SQL | DataFrame | SurrealQL | KV (key-space design) |
|:---|:---|:---|:---|:---|
| Carrier | String | Method chain (computation graph, lazily realized) | First-class value | Composite-key encoding + prefix scan |
| Composability | None (CTE is a patch) | Yes (method chains can fork/merge) | Yes (variable pipeline passing) | None — the access path is fixed at key-design time |
| Control flow | None | None (purely declarative) | Yes (Turing-complete) | None (application layer) |
| Type safety | None (string injection) | Yes (compile time) | Yes (value passing, no string interpolation) | Yes (compile-time key encode/decode) |
| Optimizer role | Black box, decides everything | Computation graph visible, optimized as a whole | Developer can plan explicitly, optimizer assists | No optimizer — the key prefix is the access path |
| Execution boundary | In-DB (logic forced up to application-layer orchestration) | In-process (host language reads the data in) / data locality (distributed implementations compute near the data) | In-DB (compute pushdown, no data movement) | Application layer directly to storage (embedded, zero RTT) |
| Language lineage | Imitation of natural English | Programming-language method chain | Pan-Rust (let/pipeline/postfix types/expressions) | None (binary keys/data flow) |
| Generality | None (inside the DB sandbox) | Any host-language library (ML, networking, filesystem…) | Inside the DB sandbox (HTTP calls, compute pushdown) | Fully general (any host) |
| Composition cost | N/A | Python indentation and the fluent API are a poor fit; switching between method chain/control flow | Unified, no switching | Key design up front; queries O(1) |

**SQL**: string declarative. A query is text, handed to the optimizer to parse + optimize + execute. Composability is poor — no control flow, no variable passing. Written order and execution order contradict each other (SELECT first, FROM last). The optimizer is a black box, and the developer cannot explicitly control the execution path.

**SurrealQL**: a unified, fused language. A query is a **value**, not text — `LET $x = SELECT ...; $x | update...`, with variables piped directly. Turing-complete, with built-in control flow. It does not rely on the optimizer to guess — the developer can plan query logic explicitly. The key advantage is **seamless composition** — queries, control flow, and variables live in one syntactic space with no switching friction, and the logic executes in-DB with no data movement. The cost is that it only runs in the DB environment and is not general-purpose. "Not being able to call external libraries" is not an inherent limitation — it depends on the security policy: `plpython3u` can likewise call arbitrary libraries, SurrealQL can access HTTP, and Wasm plugins (Surrealism) can call libraries in other languages with precise permission control.

**DataFrame**: the semantic layer for tabular operations. Method chains build the data flow, with type checking, IDE completion, and composability. It shares the same layer as SQL (declarative + logical optimizer) but replaces the string with programming-language syntax — type-safe, no string-injection risk. The key advantage is **generality** — it runs inside the host language (Python/Rust/Scala) and can call arbitrary external libraries (sklearn, networking, filesystem). Switching friction exists between the syntactic space of the method chain and the host's control flow, but it is still a first-class citizen within the program, more direct than SQL's whole layer of strings. The real pain point is in the host language itself — Python's mandatory indentation and the fluent API's method chains are naturally a poor fit, and long call chains are hard to lay out under indentation rules; this is a language-layer constraint, not a query-layer one. Note: DataFrame semantics themselves do not distinguish rows from columns — columnar storage is Polars's implementation choice, not the semantics (Pandas is row-store). Polars, Pandas, and Spark all implement this semantic layer; Spark additionally offers other programming models such as RDD and Dataset, and we take only its DataFrame part here. Underneath, it is often distinguished as a **computation-graph API** (method chains build a DAG, lazy evaluation, the optimizer sees the whole graph and optimizes as a whole at collect time) as opposed to an eager implementation (such as Pandas).

**DataFrame or SQL**: it depends on the user. Internal use, where the query patterns are controlled by your own code → DataFrame is handier (type-safe, composable, host-language-friendly); exposing queries externally, AdHoc dynamic analysis → SQL is more suitable — external or ad-hoc users cannot fix the query patterns at design time, and need a Parser + Optimizer to carry dynamic multi-dimensional queries (the same logic as [KV Storage Engine](kv-storage-engine-en.md) "when it must metamorphose into a database", see its AdHoc analysis paragraph).

**KV (key-space design)**: the extreme of having no query language. Interaction is entirely determined by the **key-space design** — composite-key encoding compiles the access path (the order of the prefix scan) into the keys at compile time, with no parser, no optimizer, no string injection. The price is flexibility: query patterns must be predictable and fixed at design time (see the "requirement controllability" of [KV Storage Engine](kv-storage-engine-en.md)), in exchange for zero parse overhead, deterministic responses, single-binary size, and **embedded zero RTT** (in-process direct connection, no network round-trip — the lowest of the four). For scenarios where "query patterns are predictable", KV is a lower layer than any query language — it introduces no language, you just write keys directly.

**KV + DataFrame combination**: a division of payload, not a replacement. KV does persistence + row-level access (point lookup / fixed-prefix scan = OLTP), DataFrame does the in-memory analysis layer (read out → multi-dimensional aggregation) — KV provides the persistence DataFrame lacks, DataFrame provides the analytical capability KV lacks; they complement exactly. This is the isomorphic argument that "SQL is a higher-level wrapper over KV", but on this side it is better than SQL: type-safe (compile time), stepwise (scan KV first, then `df.xxx`, with the intermediate state being ordinary data rather than a string black box), host-language-friendly. This forms a complete chain: predictable query patterns + internal → pure KV (not even a wrapper needed); internal flexible analysis → KV + DF; external / AdHoc / uncontrollable requirements → SQL + columnar (DuckDB) or a full database (see [KV Storage Engine](kv-storage-engine-en.md) "when it must metamorphose into a database"). The first two tiers need no Parser + Optimizer — the user is internal, the patterns are controllable, and the DF computation graph is a sufficient query layer. Boundary: DF is an in-memory state; once the data is too large to fit, it must be distributed (Spark), and KV's embedded zero RTT disappears — the ceiling of the combination is mid-scale on a single machine; moreover, reading KV row-store into a DataFrame incurs layout-conversion overhead, and large-scale pure analysis is still better served by direct columnar systems like DuckDB.

These four are not a binary opposition of "purely declarative vs. unified", but a **trade-off of composition position and generality**: SQL cannot compose (string), so logic is forced up to the application layer; DataFrame can compose (host language), is highly general but constrained by host syntax (Python indentation is unfriendly to the fluent API; the implementation must read data in, and distributed implementations rely on data locality to compute near the data but are still a separate compute engine); SurrealQL can compose (unified syntax), with no friction and logic in-DB, at the cost of a runtime environment confined to the DB, though external capabilities can be extended on demand via HTTP and Wasm plugins; KV is the extreme of "no language" — the access path is fixed at key-design time, it is the most general (any host, zero parse), at the cost of query patterns having to be predictable and fixed at design time.



**Network RTT cost** (network round-trips per operation, comparing different storage layers):

> **RTT is the symptom, compute sinking is the means**: what the table below compares is the "number of round-trips per operation" (the symptom), while compute sinking (and embedded direct connection) is the means of achieving low RTT. On Redis (Lua) the same means is a failure case — dragged down by single-threading/Hash Slots, logic falls back to the application layer and the RTT is not actually reduced (see [Redis: the "Network RAM" Trap](redis-critique-en.md) §8).

| Storage | RTT | Reason |
|:--|:--|:--|
| KV (embedded) | **0** | In-process direct connection, no network |
| SurrealDB | **1** | Compute pushdown, graph traversal + logic completed in a single request, one round-trip |
| PostgreSQL | **≈1** | Multi-table/complex operations also approach 1 — CTE/single-transaction batch updates across tables, plpython3u carries complex logic |
| Redis | **N (Lua can compress to ~1)** | Multiple commands N round-trips; Lua (EVAL) can batch, but far less than plpython3u, and complex logic is still limited |

---

## Relational Modeling: SQL's Physical Shackles vs. SurrealDB's Free Topology

SQL's relational model rests on a rigid assumption: every table has a fixed column structure, a foreign key can only point to one definite table, and many-to-many must be bridged through a junction table. This model draws beautifully on an ER diagram, but in engineering implementation it creates a mass of unnecessary physical constraints — you must anticipate every relation type at design time and then weld them down with DDL. SurrealDB's multi-modal architecture (document + graph + relational) provides three completely different relation-expression mechanisms, and you can choose freely according to business complexity instead of being kidnapped by a paradigm.

### One-to-Many: The Progression of Three Mechanisms

**SQL's path**: create foreign keys in multiple downstream tables (orders, comments, logs), all pointing to the primary key of the main table (users). Each additional downstream table means one more foreign-key constraint and one more set of JOINs.

**SurrealDB's path one: Record ID pointer** — every row of data has a globally unique Record ID (e.g. `person:alice`), and other tables store this pointer directly in a field. Underneath, it locates like following an in-memory pointer, with no explicit JOIN:

```SurrealQL
CREATE person:alice SET name = 'Alice';
CREATE orders SET order_no = 1001, buyer = person:alice;
CREATE comments SET content = 'Awesome', author = person:alice;
SELECT * FROM orders WHERE buyer = person:alice;
```

**SurrealDB's path two: graph edges (RELATE)** — instead of creating fields in downstream tables, use `RELATE` to pull out true graph edges. An edge is a first-class citizen that can carry its own properties (time, state), and queries support bidirectional arrow traversal:

```SurrealQL
RELATE person:alice -> bought -> product:iphone SET at = time::now();
SELECT ->bought->product.name AS purchases FROM person:alice;   -- forward
SELECT <-wrote<-person.name AS author FROM article:news;        -- reverse
```

**SurrealDB's path three: polymorphic reference** — a field can dynamically point to any table. In SQL one foreign-key column can only point to one fixed table, and handling "likes" (a user may have liked a product, an article, or a comment) forces multi-table inheritance or multiple nullable FKs. SurrealDB's `liked_item.*` destructures directly, eliminating the application layer's if-else:

```SurrealQL
CREATE likes SET user = person:alice, liked_item = product:iphone;  -- points to a product
CREATE likes SET user = person:alice, liked_item = article:news;    -- points to an article
SELECT liked_item.* FROM likes WHERE user = person:alice;           -- auto-destructure
```

**Selection logic**: simple one-to-many → the pointer is the most intuitive; needing edge properties (time, weight) → RELATE; polymorphic association → polymorphic reference. The three mechanisms are not mutually exclusive — one application can mix them according to the relation's characteristics.

### Many-to-Many: The Demise of the Junction Table

SQL's many-to-many is a physical structure of three tables — two entity tables plus one junction table — and queries need multiple sets of JOINs. SurrealDB splits many-to-many into two sharply different physical implementations, and the selection criterion is **whether the relation itself needs to carry business data**.

**Path one: graph edges (RELATE)** — the only correct choice when the relation has properties. The edge exists as an independent graph physical table, to which metadata such as enrollment time or usage status can be attached. No matter how many hops, the query is one line of arrow syntax:

```SurrealQL
-- edges carry their own properties
RELATE student:alice -> enrollment -> course:math SET date = time::now();
-- forward: which courses Alice enrolled in
SELECT ->enrollment->course.title AS my_courses FROM student:alice;
-- reverse: who enrolled in Advanced Mathematics
SELECT <-enrollment<-student.name AS students FROM course:math;
```

**Path two: document array pointer** — the minimalist solution when it is pure association with no properties. Store an array of Record IDs directly inside the document, completely eliminating the junction table. But the array can only hold IDs, cannot bind extra properties to any one ID, and reverse global lookup (`CONTAINS`) does a full-table scan:

```SurrealQL
CREATE student:alice SET
    name = 'Alice',
    my_courses = [course:math, course:cs];
SELECT my_courses.* FROM student:alice;                     -- forward expansion
SELECT name FROM student WHERE my_courses CONTAINS course:math; -- reverse lookup
```

**⚠️ Zombie pointer (Phantom ID)**: the biggest hidden danger of the array pattern. SurrealDB has no physical foreign-key constraint by default; after `DELETE course:math`, the `course:math` in the `my_courses` array of every student document does not disappear automatically — it becomes a zombie reference pointing into the void. In production, a scheduled application-layer cleanup script must be paired with it to scan the whole table, otherwise the database's lower layer gradually fills with invalid pointers. SQL's `ON DELETE CASCADE` solved this problem 40 years ago; the array pattern brings this pain point back to 2026.

### Core Comparison

| Dimension | SQL (PostgreSQL) | SurrealDB (graph mode) | SurrealDB (array mode) |
|:---|:---|:---|:---|
| **Physical structure** | Must have 3 tables (two entities + one relation table) | 2 nodes + 1 edge (the edge is an independent graph physical table) | Only 2 tables needed; the relation exists as an array inside the document |
| **Relation properties** | Junction table adds columns | RELATE SET adds properties to the edge | Not supported; the array can only hold IDs |
| **Query complexity** | Multiple JOINs, multiplying with hop count | Arrow traversal, one line of code no matter how many hops | `.*` property expansion, minimal |
| **Reverse lookup** | JOIN + index, performance controllable | Graph index, O(fan-out) | `CONTAINS` full-table scan, degrades at large data volume |
| **Polymorphic association** | Multi-table inheritance or multiple nullable FKs | Native support; a field can point to any table | Native support; array elements can point to any table |
| **Cascade delete** | `ON DELETE CASCADE`, declarative | No automatic cascade, needs event triggers | No automatic cascade, zombie-pointer risk, needs application-layer cleanup |
| **Physical path of a multi-hop query** | Random jumps between B-Tree pages, frequent CPU cache misses | KV prefix scan + pointer jumps, good CPU cache locality | Single-document expansion is very fast, but multi-hop needs application-layer orchestration |

**Engineering conclusion for the OLTP scenario**: in the OLTP scenario, SurrealDB's graph mode is physically superior to PostgreSQL's B-Tree JOIN on multi-hop relation traversal — this is not a gap that tuning can close, it is decided by the storage-engine architecture. PostgreSQL is a single-node relational database with no native sharding capability, so a multi-hop JOIN is random I/O between B-Tree pages, with poor CPU cache locality. SurrealDB's KV prefix scan + graph-pointer jumps is naturally faster on the same machine. OLAP is a separate domain, handled by columnar engines; see [Lakehouse Research](lakehouse-research-en.md).

### Decision Tree: Relational Modeling Choices in the OLTP Scenario

```
What is your core query pattern?
│
├─────────────────┴─────────────────┐
│                                   │
Point deep drill-down             Global aggregate stats
(from one entity, traverse        (cross-entity COUNT/GROUP BY/
2+ hops along relations)          AVG and other aggregates)
│                                   │
↓                                   ↓
[SurrealDB graph mode]            ⚠️ This is the analytical domain
RELATE edges carry properties     A serious production app should not use
Arrow traversal in one line       a transactional DB like PG for analytics
Must configure Event cascade      ↓
                                  see lakehouse-research-en.md
```

**Why there is no "array mode" branch**: the array pattern is a degenerate form of the graph pattern — it sacrifices edge properties, bidirectional queries, and the graph index, keeping only the simplest pointer storage. The array pattern is meaningful only when the relation is extremely simple (pure association, no properties, no multi-hop requirement) and you are willing to bear the zombie-pointer risk. In production, the graph pattern is the default choice for OLTP relational modeling.

---

## PostgreSQL: The Confidence of a One-Stop Choice

PostgreSQL is not just a "relational database" — its "batteries-included" philosophy makes its one-stop coverage far exceed expectations.

| Feature | PostgreSQL approach | SurrealDB approach | Verdict |
|:---|:---|:---|:---|
| **Architecture** | Extensible monolith (plugins/extensions), 40 years of ecosystem accumulation | Unified multi-model engine, newborn but evolving fast | **PG ecosystem crushes** |
| **Graph queries** | PG 19 previews SQL/PGQ standard support, about to land in GA. PG currently usable in production (≤18) depends on the `AGE` extension or recursive CTEs, a clunky experience. After PG 19 lands, SQL/PGQ as an add-on layer on the SQL standard still lags SurrealQL's native graph semantics in integration depth, but it is "good enough". | SurrealQL's native graph semantics (`RELATE`, `->`, `<-`) are first-class, with deeper integration. | **SurrealDB's experience is better, but the gap narrows after PG 19** |
| **GraphQL** | Storage-independent. GraphQL is an API query language (HTTP layer), typically sitting *on top of* PG/SurrealDB. Both support layers are symmetric. | Same as left: SurrealDB exposes a GraphQL endpoint through a gateway layer, while the native engine is still SurrealQL. | **Tie** |
| **Scripting/logic** | `plpython3`, `plv8`. Awkward syntax but a thick ecosystem — can call arbitrary external libraries, with mature community support. | **SurrealQL**: Turing-complete, built-in, Rust/Nu-like syntax. Ergonomic, but a thin ecosystem. | **PG has a thicker ecosystem, SurrealQL has a better experience** |
| **schema-free** | JSONB fields: one JSONB column per table lets you skip schema design. schema-first is the default, JSONB is the escape hatch. | schema-free by default, optionally add schema. schema-free is first-class. | **A paradigm difference, not a capability difference** |
| **Performance** | Mature optimizer, Index Scan / Bitmap Scan, 40 years of optimizer accumulation. Performance degrades when heavy extensions are stacked. | Rust static semantics, KV engine with fast point lookups, compute pushdown reducing RTT. Lacks optimizer accumulation. | **Tie** |
| **Data consistency** | ACID + foreign-key constraints + ON DELETE CASCADE, declarative, 40 years of validation. | Requires manual event cascading; the array pattern has zombie pointers. | **PG crushes** |
| **Toolchain** | `pg_dump`, `pgbench`, `EXPLAIN ANALYZE`, slow-query logs, `pg_stat_statements` — a full set of ops tools, ready out of the box. | Slow-query logs are an **Enterprise Edition feature**. Mainly single-node deployment. Immature toolchain. | **PG crushes** |
| **Learning cost** | SQL is universal; every developer knows it. | SurrealQL is a new language, but for those tired of SQL it is a liberation. | **PG slightly ahead** |
| **Cloud-vendor support** | Natively supported by all major clouds (RDS, Cloud SQL, Azure Database). | No major-cloud native support. | **PG crushes** |

**Insight**: To make PG do what SurrealDB does out of the box, you would need to install, configure, and maintain multiple extensions, each with its own release cycle and compatibility matrix. But PG's advantage is that — even without any extension, it is already a complete one-stop database. SurrealDB's advantage is a "multi-model experience out of the box", at the cost of an immature ecosystem and toolchain.

### Redis Replacement: The Network RTT Trap

The classic objection to replacing Redis is **latency**.

- **Redis myth**: "Redis is fast because it is an in-memory database."
- **Reality**: network RTT (round-trip time) dominates latency. Whether the backend is Redis or a well-indexed database, the cost is dominated by network hops (microseconds/milliseconds), not the storage engine (nanoseconds).
- **Result**: for most simple queries (such as point lookups), SurrealDB is not much slower than Redis.
- **Trade-off**: at the **same network-latency cost**, you get: rich relations (graph), structured data (relational), real-time subscriptions. Compare Redis: just a Key-Value store.

→ For the full argument see [Redis: the "Network RAM" Trap](redis-critique-en.md): the network RAM trap, the architectural analysis of SurrealDB replacing Redis, the point-by-point demolition of Redis's proprietary data structures, and why the "Redis for cache, PG for storage" layered pattern is outdated under modern languages.

---

## The Challenger Logic: Why SUrrealDB's Advantages Are Not Enough

### The Iron Law of Technological Disruption

For a challenger to displace an incumbent, it cannot merely be "each with its own pros and cons" on a few points — that is the fence-sitting of dialectics. **The Matthew effect decides**: the challenger must reach "overwhelming superiority" on enough dimensions to overcome ecosystem inertia (toolchain, talent pool, community, cloud-vendor support). Out of ten points, seven or eight tied or slightly better, one or two far stronger, one slightly weaker but not too bad — that is the minimum condition for a successful challenge. Six of one and half a dozen of the other = the challenge fails.

PostgreSQL beating MySQL is exactly this logic: PG is tied or crushing on every dimension (ACID, extensibility, JSONB, full-text search, geospatial, materialized views), and MySQL wins on no dimension at all. The result was not "each has its pros and cons" but the systematic dismantling of MySQL's dominance.

**SurrealDB does not currently meet this challenge condition**: among ten dimensions, SurrealDB crushes on language design, polymorphic association, logic sinking, and RTT elimination — but is comprehensively inferior on ecosystem, toolchain, cloud-vendor support, learning cost, and data consistency. Six of one and half a dozen of the other = the challenge fails. SurrealDB's correct positioning is not "replace PG" but "complement PG on the dimensions PG cannot do".

### Schema-free: A Genuine Paradigm Difference

PostgreSQL and SurrealDB both support schema-free, but with sharply different priorities:

```
SurrealDB: schema-free by default → optional schema addition
PG:        schema-first by default → JSONB fields can bypass it
```

This is not a capability difference, it is a design-philosophy difference. SurrealDB puts flexibility first — suited to scenarios where the schema changes frequently, such as rapid prototyping and AI memory systems. PG puts structure first — suited to long-term evolution, multi-person collaboration, and production systems that need data integrity.

**The practical selection criterion**: the frequency of schema change. High frequency of change (AI memory, logs, event streams) → SurrealDB's schema-free is more natural. Low frequency of change (user tables, order tables, configuration) → PG's schema-first is safer. Mixed scenarios: PG as the main store (JSONB fields for the flexible parts), SurrealDB as a specific multi-model subsystem (if the constraints are acceptable).

### Tiered Verdict: Performance Is the Floor, Paradigm Is the Ceiling

Performance is only the ticket to enter the selection range — it suffices to guarantee "not much slower than PG", and is not the core selling point. The real differentiation comes from **paradigm-level user experience**:

| Tier | Dimension | PostgreSQL | SurrealDB | Verdict |
|:---|:---|:---|:---|:---|
| **Floor (performance)** | Single-model queries | Mature optimizer, Index Scan / Bitmap Scan | KV engine, fast point lookups but lacking 40 years of optimizer accumulation | **Tie** |
| **Ceiling (paradigm)** | Language design | SQL: syntactic order contradicts execution order, lacking modularity | SurrealQL: composable, highly expressive, cost of complex logic on par with Python | **SurrealDB crushes** |
| | Polymorphic association | Multi-table inheritance or multiple nullable FKs, clunky | Fields point directly to any table, zero cost | **SurrealDB crushes** |
| | Logic sinking | Logic in the application layer, ORM is a patch to eliminate language switching | SurrealQL is natively composable, business logic written directly on the database side, ORM loses its reason to exist | **SurrealDB crushes** |
| | RTT cost | Every Read-Modify-Write is a network round-trip | A single request completes graph traversal + business logic, zero extra RTT | **SurrealDB crushes** |
| **Ecosystem** | Maturity | 40 years, complete toolchain, native cloud-vendor support, ops tools like slow-query logs out of the box | Newborn, small community, slow-query logs are an Enterprise feature, immature toolchain | **PG crushes** |
| | Learning cost | SQL is universal, every developer knows it | SurrealQL is a new language, but for those tired of SQL it is a liberation | **PG slightly ahead** |
| | Data consistency | ACID + foreign-key constraints + ON DELETE CASCADE | Requires manual event cascading, the array pattern has zombie pointers | **PG crushes** |
| | Cloud-vendor support | Natively supported by all major clouds | No major-cloud native support | **PG crushes** |

**Verdict**: four paradigm dimensions are crushed by SurrealDB, four ecosystem dimensions are crushed by PG. The crushed dimensions happen to be the most painful points of user experience in the OLTP scenario — language design, polymorphic association, logic sinking, RTT elimination — it is not "doing it better", it is "doing different things". The dimensions PG crushes (ecosystem, consistency, toolchain, cloud support) are not a matter of time but a structural advantage of 40 years of accumulation. **Each wins some and loses some; neither crushes the other across the board.**

### Why Not ArangoDB

ArangoDB is also multi-model + graph queries at the language level, with a paradigm similar to SurrealDB. But its execution on every front is inferior to SurrealDB — showing that **a paradigm alone is not enough; execution quality decides success**. Rust's meaning here is not "it is fast because it uses Rust" but a **posture signal**: fast iteration, embracing change, affinity with modern infrastructure (native HTTP/WS/GraphQL support). The counterexample is Helix Editor — it also uses Rust, but evolves slower than Neovim, proving that Rust does not automatically equal fast iteration.

### Architecture: Separation of Query Layer and Storage Layer

SurrealDB decouples the query engine (SurrealQL parsing, permission checks, transaction coordination) from the storage engine. The same query language and client SDK can run on embedded devices, single-node servers, distributed clusters, and SurrealDB Cloud, without rewriting application code.

**Single-node storage engines**:

- **RocksDB**: LSM-tree-based KV storage, high write throughput, suited to SSD and single-node production deployment. Community Edition recommended.
- **SurrealKV** (beta): SurrealDB's in-house storage engine, developed in tandem with the database, with a smaller configuration surface, aimed at embedded and local-first scenarios.
- **SurrealMX**: in-memory storage, supporting snapshots and append-persistence, suited to development and temporary data.

**Distributed storage engine (SurrealDS)**:

SurrealDS is SurrealDB's distributed transactional storage layer, analogous to TiKV for TiDB, or CockroachDB's storage layer. Its responsibilities include: replication and consensus across nodes (Raft), distributed ACID transactions, data sharding and failure recovery. The query layer is transparent to whether the underlying layer connects to RocksDB or SurrealDS.

The Community Edition (BSL) already supports multi-node SurrealDS deployment, but with many limitations — key features such as object-storage backends and distributed Live Queries are provided only in the Enterprise Edition. The Enterprise Edition additionally provides:

- **Object-storage backend**: persist data to cheap object storage such as S3, keep hot data on local disk, sink cold data. The goal is to lower the storage cost of large clusters — object storage is far cheaper per unit than local SSD. Currently being rolled out gradually on the Scale plan.
- **Priority security patches**, **audit logs**, **FIPS-compliant encryption**, **distributed Live Queries**, **commercial support + SLA**

What the BSL license restricts is "you may not wrap SurrealDB as a database service and offer it to third parties"; it does not restrict self-use. The source is public, and a capable team can implement the S3 backend themselves.

**The key architectural difference from Neo4j/MongoDB**: the latter's distributed capability and storage engine are coupled, whereas SurrealDB's query layer and storage layer are separated — this is the storage-compute-separation version of a database.

### The Difference Between BSL and Open Core

SurrealDB chose BSL (Business Source License) rather than the Open Core model. The essential difference between the two:

**Open Core**: the Community Edition is a complete open-source product, but its features are artificially crippled. The Enterprise Edition profits by adding proprietary features (such as audit logs, SSO, RBAC). The problem is that — the features missing from the open-source version are exactly the ones production environments require, creating the friction of "seemingly open source but actually paid". A typical case is the RisingWave Community Edition, which does not support automatic schema import; with many tables, hand-writing SQL is unrealistic.

**BSL**: the Community Edition is a complete product, with all features present. The restriction is at the business-model level — you may not wrap the software as a database service and offer it to third parties. Self-use, modification, and self-hosting are all unrestricted.

| Dimension | Open Core | BSL |
|:---|:---|:---|
| Feature completeness | Open-source version crippled | Complete product |
| Location of restriction | Feature layer (daily-use friction) | Business layer (one-time legal assessment) |
| Enterprise Edition value | Feature differences | Ops/compliance/support |
| User experience | Hidden friction, continuously consuming | Transparent restriction, assessed once and done |

BSL is more honest with users — the restriction is explicit and one-time; Open Core's restriction is implicit and continuously manufactures engineering friction.

---

## SurrealQL as an Interaction Paradigm

### Background: The "String Injection" Trap

Even in advanced embedded architectures (such as PostgreSQL's `pl/python3u`), the query layer remains foreign to the host language. The problem with pl/python3u is not only that "the outer SQL syntax is awkward" — **the inside is also valid Python code, but executing an SQL query is still in string form** (through the built-in `plpy` client), forcing you to manage two syntax trees. SurrealQL, by contrast, is a **unified, fused language**: variables, control flow, and queries live in the same syntactic space, with no "Python calling an SQL string" boundary.

### Implementation Comparison

| Feature | Legacy pattern (SQL in Python/Pl-Python) | SurrealQL pattern (logic in the database) |
|:---|:---|:---|
| **Logic flow** | Scattered across the application and database layers. | Executed atomically inside the database. |
| **Data access** | **Opaque string injection**: `db.query("SELECT ...")` | **Native composition**: `LET $user = SELECT ...` |
| **Network cost** | **High**: Read → Logic → Write, multiple round-trips. | **Zero**: logic executes on the storage node. |
| **Security** | Injection risk; type checking broken at the string boundary. | Compile-time checking; strongly typed values. |

### Challenges and Mitigations

**Challenge: the legacy fear of "stored procedures"**: historically, in-database logic was rejected because of poor CI/CD, version control, and difficulty extending.

- **Defenses**:
  - **Distributed logic**: unlike legacy monolithic databases (Oracle/PG), SurrealDB is distributed by nature. Logic scales horizontally with the cluster (TiKV/FDB backend).
  - **Git-native**: scripts are versioned in `.surql` files, supporting standard CI/CD.

**Challenge: AI unfamiliarity**: LLMs are less proficient with SurrealQL than with SQL, which may slow initial development or lead to "hallucinated" syntax.

- **Defense**: SurrealQL supports an **SQL-compatible syntax mode** for traditional queries. This provides a safe fallback for AI-assisted generation of basic CRUD, while imperative logic is kept in SurrealQL.
- **Action item**: build an internal **snippet and prompt library** to bridge the AI knowledge gap.

### Conclusion

The productivity gains from native composability, reduced network overhead, and a unified logic layer outweigh the initial learning cost. SurrealQL shifts the paradigm from "the application orchestrating data" to "the application residing in the data".

### Case Study: Compute Pushdown for Embedding Generation

The skillforge memory system needs to generate a vector embedding for each memory (for semantic retrieval). A comparison of two implementation paths directly validates the above principles of "compute pushdown" and "native composability":

**Path A (application-layer generation, anti-pattern)**:
```
Python calls the Ollama API → fetch embedding → write to SurrealDB
```
1. **Double transmission and storage**: the embedding data must ultimately be stored in SurrealDB (the HNSW index needs it), so the transmission on the SurrealDB side is unavoidable; generating on the Python side and then passing it to SurrealDB means passing it one extra time. Both transmission and storage are doubled:

| Path | Transmissions | Storage location | Memory footprint |
|------|----------|----------|----------|
| Generated on the Python side | Ollama→Python + Python→SurrealDB = **2** | Python process + SurrealDB = **2 places** | Python holds 4KB/record (multiplied during batch writes) |
| Generated inside SurrealQL | Ollama→SurrealDB = **1** | SurrealDB = **1 place** | Zero footprint on the Python side |

2. **Maintenance cost of multiple consumers**: multiple consumers such as memory retrieval, Session RAG, and Consolidation all need embeddings. The embedding-generation logic is encapsulated in SurrealDB's `fn::ollama::embed`, invisible to the application layer; consumers share the same function and do not each maintain their own.

**Path B (generation inside SurrealQL, the correct approach)**:
```SurrealQL
CREATE memories SET
    content = $content,
    embedding = (fn::ollama::embed('bge-m3', $content)).embeddings[0];
```

**Essence**: Path A is the "application orchestrates data" mindset of the SQL era — the application layer computes, the database stores. Path B is SurrealDB's "compute pushdown" mindset — embedding generation is the data layer's responsibility, not the application layer's. Because SQL lacks composability (it cannot call external functions within a query), it is forced to choose Path A; SurrealQL's native composability makes Path B possible.

### Rebuttal: The Inertial Thinking of "Traditional Backend Ops Advantages"

When evaluating "whether ops logic should be implemented at the DB layer", a common objection is that a Python backend has natural advantages in retries, concurrency control, observability, model switching, and so on. These arguments hold only on the premise that SQL's expressiveness is insufficient — but SurrealQL is not SQL.

| Objection | Why it does not hold in SurrealQL |
|:---|:---|
| **Retry/circuit breaker** | SurrealQL is a Turing-complete Rust-like language; writing a retry loop (`FOR` + `SLEEP` + a counter) and fallback logic (`TRY {} CATCH`) is no more complex than in Python. Configuration (`base_url`, `api_token`, `max_retries`) is already in the `config` table; just add fields. |
| **Concurrency control** | This is the **downstream's responsibility**, not the DB layer's. The caller does congestion control (AIMD) based on response time; SurrealDB's exposed system tables (`info()`) and `/metrics` endpoint are more direct than Python instrumenting itself. |
| **Observability** | SurrealDB has `/health` and `/metrics` endpoints, and system tables expose internal state. No need for an OpenTelemetry SDK as a middle layer — get it directly from the DB layer, a shorter and more accurate path. |
| **Hot model switching** | `UPSERT config SET value = { model: 'new-model' }`, and all consumers take effect automatically on their next call. Cleaner than changing environment variables/redeploying on the Python side, and it supports canary rollouts (routing to different models by user/tenant). |

**Essence**: these "advantages" are a product of evaluating SurrealQL with SQL's capability boundary. SurrealQL's expressiveness already covers this ops logic, and because it hugs the data layer, the implementation is more direct. Having the Python backend do all this instead adds a layer of indirection — the data is in the DB, the ops logic is in Python, and the two communicate over the network, essentially complicating a simple problem.

### Combined Design of Concurrency Control and Backpressure

Concurrency control is **the caller's responsibility**, not the DB layer's — this is the manifestation of the End-to-End Principle. The bottleneck is Ollama (the embedding service), and the decision-making power over concurrency should lie with the caller (the Python client), because the caller holds the global context (how many tenants, the current total load, business priorities).

Two-layer mechanism: **backpressure is the signal (when to call), congestion control is the algorithm (how to call)**. Latency above a threshold but not timed out = soft backpressure (stop growing but do not punish); timeout/5xx = hard backpressure (multiplicative decrease); consecutive failures = cooldown period (circuit breaker). Soft signal for smooth adjustment + hard signal for fast retreat + cooldown period against oscillation — the three-layer combination avoids the oscillation or sluggish reaction of a single-layer scheme.

SurrealDB exposes `/metrics` (Prometheus format) and the `info()` system table; the caller reads end-to-end latency and errors directly, and the signal is more accurate than Python instrumenting itself — Python instrumentation observes the "Python → SurrealDB" segment, but the bottleneck is in the "SurrealDB → Ollama" segment. Because `fn::ollama::embed()` is a passthrough call, SurrealDB's response directly reflects Ollama's state, with minimal signal distortion.

Having the DB layer do backpressure instead introduces oscillation: `Ollama overloaded → SurrealDB rejects → Python retries → SurrealDB calls Ollama again → overwhelmed`, one extra round-trip, and the retry hits the bottleneck node again. When the caller does backpressure, a single round-trip suffices to complete the retreat.

→ For details see [Congestion Control Design](congestion-control-design-en.md): signal-chain architecture diagram, AIMD decision pseudocode, monitoring-interface details.

---

## Cross-References

This document is the evaluation dossier for SurrealDB as a candidate for the unified data layer, and forms a closed loop with the following architectural analyses:

- **[Unified Data Layer](unified-data-layer-en.md)**: the conclusion of the core architecture — why it adopts a PG/KV + DuckDB·Lakehouse layering rather than a single-engine unification on SurrealDB.
- **[KV Storage Engine](kv-storage-engine-en.md)**: the storage design for KV replacing PG in large-scale or predictable-query-pattern scenarios.
- **[Lakehouse Research](lakehouse-research-en.md)**: the columnar selection and implementation for analytical loads (OLAP).
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: the network RTT trap and cache layering (the argument for SurrealDB replacing Redis).
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: the philosophical consistency between SurrealQL and Rune/Steel/Koto, etc.
- **[Aura Architecture §5](aura-architecture-en.md)**: the lightweight solution when a fully embedded KV and consensus are needed.
- **[MySQL Critique](mysql-critique-en.md)**: a contrast with SQL anti-patterns and the sharding illusion.
- **[Flux Architecture](flux-architecture-en.md)**: SurrealDB's Live Queries (WebSocket real-time push) are one implementation path for the reactive architecture's dynamic layer — the client subscribes to data-change events, no polling needed.
- **[Modern Language Design](modern-language-design-en.md)**: SurrealQL's pan-Rust lineage.

**The unified first principle**: no technology worship; look only at real hardware physical limits and team productivity. Whether a database, a scripting language, or an editor, the same Occam's razor is used to decide.
