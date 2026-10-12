# MySQL: Relational Legacy and SQL Anti-Patterns

> **Languages:** [English](mysql-critique-en.md) (primary) · [中文](mysql-critique.md)

**Verdict**: Rejected as the primary data store in new architectures.
**Status**: In the early period, the designer's amateurism in the database domain; in the later period, after the Oracle acquisition, it became commercial strategy—the absence of features isn't an inability to build them, it is to sell cloud services.

## Executive Summary

MySQL's popularity conceals problems on two levels. **Early period (before 2009)**: the designer's amateurism in database theory—incorrect GROUP BY logic, terminology deviating from the SQL standard, isolation level deception—is a manifestation of the designer's lack of basic understanding of set theory, transaction models, and storage engine architecture. **Later period (after the 2010 Oracle acquisition)**: the absence of features changed from a capability problem to commercial strategy—JSON dragged until 8.0, vector search is still absent today. It's not that they couldn't build it; the more capable MySQL is, the less users will migrate to Oracle Cloud or buy an Oracle commercial license.

SQLite implemented window functions, CTEs, FTS5, JSON1, and R-Tree in 100,000 lines of code. MySQL 8.0 took millions of lines of code to add the same features, and still hasn't fully reached the same level. This is not a gap in engineering effort, it is a gap in design capability.

*Analogy with Go: Rob Pike's prestige cannot conceal Go's amateurism in its type system, generics, and error handling. MySQL is the same—popularity does not equal technical depth.*

## 1. Type System Defects

### Storing Time as a String Is Extremely Amateurish

A common MySQL anti-pattern: storing ISO 8601 timestamps in `VARCHAR` or `TEXT`, on the grounds of "good compatibility" or "sorting is fine". Consequences:
- Cannot perform time arithmetic directly (adding and subtracting months, computing differences, all require parsing first)
- Index efficiency is lower than native types (strings occupy more storage than integer/time types)
- Every use requires parsing into a time object, adding runtime overhead
- Semantics are unclear, violating the principle of least astonishment

**Variant: Unix Epoch**. Storing a Unix timestamp as `BIGINT` is slightly better—storage and sorting overhead become normal, but readability is poor, time zone handling requires application-layer intervention, and a query like `WHERE created_at > NOW() - INTERVAL 7 DAY` becomes a manual computation. It is still the mindset of **compensating in the application layer for defects in the database's type system**.

This is not an "acceptable workaround", this is a **design defect**. Any database (MySQL, PostgreSQL, SurrealDB) should use a native time type.

### The String Trap of ENUM

MySQL's `ENUM` is stored as an integer, but is implicitly converted to a string when compared. This leads to:
- `WHERE status = 'active'` uses the index, `WHERE status = 1` does not
- Modifying an ENUM value during migration requires `ALTER TABLE` (table lock)
- Application-layer types are inconsistent with database-layer types

### Strings Have No Static Form (CHAR Is a Joke)

JSON's object/array each have two semantics: object corresponds to map (a dynamic set of keys) and struct (a static field layout), array corresponds to seq (a dynamic sequence) and tuple (a static fixed-length tuple; a vector is an application of tuple, list/set/string are applications of seq). Within this classification there is one asymmetry: **strings exist only on the seq side, with no static counterpart**. A string's identity is a sequence of characters; length is a property of the content, not a property of the type. `CHAR(n)` forcibly manufactures a static form for something that has only dynamic semantics—it is not a string type, it is a storage layout disguised as a type.

The only seemingly exceptional scenario—where length itself carries semantics (such as country codes or MD5 digests)—precisely proves the point: every position of such a value is meaningful and non-commutable; it is essentially a tuple (a fixed-length field record), not a string. MySQL cannot express this semantics: `CHAR(2)` carries no positional constraint whatsoever, degenerating into a variable-length string padded with spaces.

Behavioral defects:

- **The two-faced trailing space rule**: `CHAR` pads with spaces on storage and strips them on read, yet comparison depends on the collation—`utf8mb4_general_ci` is PAD SPACE (`'a' = 'a  '` is true), `utf8mb4_0900_ai_ci` is NO PAD (false). The same value, the same column, two charset versions give different equality. Type semantics and collation semantics are entangled; this is confusion at the design level.
- **The performance argument is dead**: the only historical justification for CHAR was that fixed-length rows ease record location (MyISAM fixed row format). InnoDB stores CHAR in a variable-length format as well, so the so-called fixed-length advantage does not exist at the engine layer; and in a secondary index, an index entry for `CHAR(255) utf8mb4` is counted at the full width of 1020 bytes, while `VARCHAR` counts only the actual length—**CHAR is in fact worse on the very dimension for which it is still advertised (index efficiency)**.
- **Any change means a table lock**: changing a CHAR length requires `ALTER TABLE`, the same disease as changing an ENUM value—once width enters the schema type, it becomes part of the structure.

PostgreSQL retains the standard `character(n)` but the community unanimously recommends `text`; CHAR in the SQL standard retains only FIPS compatibility value. Fixed-length strings are a fossil from the 1970s era of record layouts, with no defensible justification at the type system level.

## 2. GROUP BY Logic Error

The `GROUP BY` operation converts a set of rows into groups. The output must be compatible with the **group** structure:

1.  **Grouping key**: included in `GROUP BY`. All rows within the group have the same value. Safe to select.
2.  **Aggregate**: passed to `SUM()`, `MAX()`, etc. The group is reduced to a single scalar. Safe to select.

**MySQL anti-pattern**: allows a column in `SELECT` that is **neither** grouped **nor** aggregated.

*   **Logic error**: you are asking for *one* value out of *a set of different values*. Which one?
*   **Problem**: the engine arbitrarily picks one value (usually the first row it encounters) and presents it as the answer for the entire group.
*   **Essence**: this is a semantic type error. It attempts to convert a `Set` into a `Scalar` without a reduction rule. It implies a single truth when in fact there are many.

This is not a criticism that "MySQL is not a qualified database engine". This is a criticism of the designer's basic understanding of logic. If you cannot distinguish a set of values from a single value, you lack basic set-theoretic reasoning. That this "feature" was celebrated as "convenience" for years shows how low the bar for "industry standard" is.

Although MySQL 5.7+ added `ONLY_FULL_GROUP_BY` to fix this, the damage is done: the culture of "lenient SQL" persists, and the engine enabled this anti-pattern for decades.

## 3. The Stupidity of the Query Optimizer

Modern engines (ClickHouse, SurrealDB) use cost-based optimization (CBO) and dynamic programming to find the best execution plan. MySQL's optimizer is notorious for being fragile:

### Join Order Incompetence
*   **Problem**: the optimizer frequently fails to reorder `JOIN`s efficiently, especially with many tables. It may choose a large table as the driving table, forcing a nested loop scan over a small table.
*   **Hack**: developers must use `STRAIGHT_JOIN` to intervene manually and force the order. If the database engine cannot work out the optimal order, this is a fundamental optimizer failure.

### Index Rigidity (Left-Prefix Rule)
*   **Problem**: a composite index such as `INDEX(a, b, c)` is strictly bound by the left-prefix rule. A query filtering on `WHERE b = 1 AND c = 2` (skipping `a`) cannot use the index and falls back to a full table scan (ignoring the newer, limited Index Skip Scan feature).
*   **Critique**: modern adaptive engines handle out-of-order filtering via bitmap indexes or adaptive scans. MySQL's rigid B-Tree implementation forces schema designers to guess future query patterns years in advance.

### "Count" Scans
*   **Problem**: `SELECT COUNT(*)` on an InnoDB table usually requires a full index scan, because InnoDB does not maintain a cached row count (due to MVCC visibility rules).
*   **Critique**: while theoretically defensible for strict consistency, in practice it means a simple metadata operation on a large table is extremely slow. Relying on a full scan for a basic metadata query is unacceptable in a modern high-concurrency environment.

## 4. Terminology vs. the SQL Standard

MySQL's naming conventions frequently deviate from the ANSI-SQL standard and database theory. This is not merely semantics—it reflects the gap between academic rigor and early engineering pragmatism.

### 4.1 WAL vs. Redo Log

| Terminology | Scope | Description |
|------|------|------|
| **WAL** (Write-Ahead Logging) | Theoretical/general | Principle: the log must be persisted to disk *before* the data pages. Proposed by Jim Gray in *Transaction Processing: Concepts and Techniques*. Used by PostgreSQL, Oracle, SQL Server, etc. |
| **Redo Log** | MySQL/InnoDB implementation | InnoDB's concrete WAL mechanism. Named "Redo" to pair with the **Undo Log**—during crash recovery, InnoDB *redoes* committed changes from the redo log, then *undoes* uncommitted transactions via the Undo Log. |

**Why the difference?**
- InnoDB implements WAL as a circular buffer with a specific recovery role ("redo"). The name reflects the *action* rather than the *principle*.
- The MySQL Server layer also has the **Binlog** (logical, used for replication), which is also a form of WAL. Calling both "WAL" would create ambiguity, so InnoDB's physical log gets the specific name "Redo Log".

**Bottom line**: Redo Log = InnoDB's WAL implementation. The principle is general; the name is engine-specific.

### 4.2 Schema vs. Database

In ANSI-SQL and database theory, the hierarchy is strict:

```
Catalog (cluster/instance)
  └── Schema (logical namespace/package)
        └── Table, View, Procedure...
```

- **Catalog**: the top-level container (usually the entire instance).
- **Schema**: a logical namespace within a Catalog, used for object isolation and access control. Cross-Schema queries are transparent (e.g., PostgreSQL: `SELECT * FROM s1.t1 JOIN s2.t2`).
- **Database**: in theory refers to the entire physical collection of data (that is, the instance itself).

**MySQL's deviation**: `Instance → Database → Table`. No Schema layer.

**Why?**
- Early MySQL (the MyISAM era) mapped each "Database" to a filesystem directory under `/var/lib/mysql/db_name/`. The term "Database" was chosen over "Schema" for simplicity.
- When MySQL 5.0 tried to align with the SQL standard, `DATABASE` was already too entrenched to change. Compromise: `CREATE SCHEMA` is a **synonym** for `CREATE DATABASE`—they produce exactly the same result.
- Unlike a true Schema (purely logical), a MySQL Database is bound to physical storage, file permissions, and character set. Cross-database queries work but have pitfalls (`lower_case_table_names`, permission boundaries).
- **Here, in fact, an orthogonal isolation dimension is lost**: the standard hierarchy uses two layers, **Catalog (environment isolation)** and **Schema (logical isolation)**, to separate different responsibilities—Catalog isolates deployment environments (dev/staging/prod, physical instances), while Schema organizes logical namespaces semantically **within the same environment**. MySQL has only a Database, compressing these two orthogonal dimensions into one: it serves both as environment isolation and as logical namespace. **One fewer isolation layer = losing the expressive power to "freely organize logical modules within a single environment"**—to separate by logical domain within the same environment (such as order/crm/inventory), you have to split into multiple Databases, and each Database is then forced to bind to physical storage and permissions (environment semantics); logical organization is polluted by environment semantics, producing the practical confusion of "database-as-schema" seen with `order db`/`crm db`.

**Bottom line**: MySQL's "Database" = the standard's "Schema". What MySQL users call "creating a database" is in theory "creating a schema".

### 4.3 Other Terminology Gaps

| Concept | Standard terminology | MySQL terminology | Notes |
|------|---------|-----------|------|
| Isolation: repeatable read | The SQL standard RR permits phantom reads | InnoDB RR **prevents** phantom reads via Next-Key Locks | MySQL's RR is strictly stronger than the SQL standard definition |
| Clustered index | Not in relational theory (a table is an unordered set) | InnoDB: data is stored together with the PK index ("index-organized table") | An engineering artifact of InnoDB's B+Tree design |
| Rollback segment | Rollback Segment / Undo log | InnoDB: "Undo Log" | The Undo Log is itself managed as data, and even has its own Redo Log (recursive) |

These gaps exist because MySQL optimized for early-developer ergonomics rather than theoretical correctness. When migrating to a distributed database that follows standard Schema semantics (TiDB, CockroachDB), the mental model must be realigned.

## 5. The Sharding Illusion and the Clustered Index Obsession

### 5.1 The Clustered Index Obsession ("Index-Only Doctrine")

InnoDB is a **clustered index** (index-organized table)—data rows are physically stored in the leaf pages of the primary key B+Tree. This forces MySQL developers into an obsession with primary key choice (the classic auto-increment vs. UUID debate), because the PK *is* the physical storage order.

**Reality (Oracle and PostgreSQL)**: both use **heap tables** by default—data storage and indexing are decoupled. The primary key is just an ordinary unique index; it has no effect on physical row placement. This means:
- No PK obsession—any column can serve as the PK, with no performance consequence for the write pattern.
- Multiple indexes can be equally efficient as access paths; none of them is the "one" physical order.

### 5.2 Sharding Superstition ("Database-and-Table Sharding Superstition")

InnoDB's clustered index caused single-table performance to degrade after roughly 10 million rows in historical versions, so the MySQL ecosystem developed an obsession with application-layer sharding (splitting databases and tables)—a deeply invasive architectural surgery. But InnoDB's "performance advantage" no longer exists against modern hardware and alternatives—PostgreSQL's heap tables + parallel query perform better on the same hardware.

**Reality**: Oracle and PostgreSQL handle hundreds of millions or even billions of rows in a single table through:
- **Native declarative partitioning**—`PARTITION BY RANGE/LIST/HASH`, transparent to queries.
- **Parallel query engines**—`max_parallel_workers_per_gather` in PostgreSQL lets a single query span multiple CPU cores across partitions.

Many scenarios that require full sharding surgery in MySQL can be solved in modern PostgreSQL simply by enabling parallelism and setting up horizontal partitioning—zero application code changes.

### 5.3 Application-Layer Sharding Costs

MySQL was designed as a single-node database. Every attempt at horizontal scaling is a workaround:
```
Request → Middleware → user_id % 1024 → DB-37/Table-12 → Query
```
*   **Cross-shard JOIN**: impossible, or requires expensive application-side assembly.
*   **Migration**: manual data surgery. Splitting a shard requires stopping writes, moving data, and updating routing rules. Distributed native systems do this through background rebalancing.
*   **Consistency**: cross-shard transactions require TCC/Saga patterns, which are weaker than true ACID.

### 5.4 Native Partitioning vs. Application Sharding

| Dimension | MySQL sharding | Native partitioning (PostgreSQL, Delta Lake) |
|------|-----------|-------------------------------|
| **Routing** | Application/middleware computes a hash | The engine decides via the partition key |
| **Cross-partition queries** | Manual assembly, N queries | A single SQL, the engine handles it |
| **Rebalancing** | Manual, with downtime | Automatic, in the background |
| **User experience** | "Find the right database, then query" | Write standard SQL, the engine does the rest |

## 6. Storage Engine Confusion ("Conceptual Confusion of Storage Engines")

**MySQL's approach**: pluggable storage engines (MyISAM, InnoDB, Memory, etc.). This forces MySQL developers to constantly debate "should this table use InnoDB or MyISAM?"—a decision that affects transaction support, lock granularity, crash recovery, and foreign key behavior at the *table* level.

**Comparison with peers**: PostgreSQL, Oracle, and SQL Server all use a **single integrated storage engine** covering all scenarios (OLTP, JSONB, full-text search, GIS, time series). MySQL needs multiple engines to barely cover these scenarios—InnoDB handles transactions, MyISAM handles read-only, Memory handles temporary tables. This is not a "flexible trade-off", it is **complexity that other databases do not have to bear**. Users are forced to make an engine choice at the table level, a choice that simply does not exist in other databases.

## 7. Isolation Level Deception ("Pulling the Wool Over the Eyes on Isolation Levels")

### 7.1 RR as a Cover for a Replication Defect

MySQL's default isolation level is **Repeatable Read**—which appears stricter than the **Read Committed** default of PostgreSQL and Oracle. But this is not a design choice made for correctness; it is **a cover for an engineering defect**.

Early MySQL replication was **statement-based Binlog** (SQL text replication). Under read committed, the same statement could produce different results on the primary and the replica (because different snapshots see different rows). By elevating to RR, InnoDB's MVCC snapshot became stable within a transaction, concealing the inconsistency caused by statement-based replication.

The real fix arrived later, with **row-based replication**—but by then, RR was already the default and the ecosystem had been built around it.

### 7.2 Serializable: Forged Serialization

MySQL's `SERIALIZABLE` isolation level implements "serializability" by **brute-force locking every read** (converting all `SELECT`s into `SELECT ... LOCK IN SHARE MODE`). Under concurrency this is a performance disaster.

Modern engines (PostgreSQL uses **SSI—Serializable Snapshot Isolation**) achieve true serializability through **conflict detection in the concurrency graph**—with no extra locks. MySQL has none of this at all.

### 7.3 Repeatable Read: the "Sleep Paralysis" Phantom Read Bug

MySQL's RR claims to prevent phantom reads, but it achieves this through an **MVCC snapshot illusion**—not true locking. The snapshot is consistent for *snapshot reads* (ordinary `SELECT`), but once you mix in *current reads* (`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`), the illusion shatters:

- **Snapshot read** sees the old MVCC snapshot (no phantom read).
- **Current read** sees the latest committed data (a phantom read appears).

This creates the well-known **phantom read consistency problem** under MySQL RR—the transaction believes it is protected from phantom reads, but a concurrent `UPDATE` on the "phantom" rows breaks the illusion. PostgreSQL's RR (actually SI—snapshot isolation) honestly states what it guarantees; MySQL's RR pretends to be stronger than it is.

### 7.4 The Hermitage Tests: Three Heavy Blows

**Hermitage** is a taxonomy of concurrency anomaly forms for the correctness of database transaction isolation. Combined with Martin Kleppmann's **Hermitage transaction test project** and Jepsen's **Elle** black-box transaction tester, it provides strict, reproducible evidence of each database's consistency guarantees.

When MySQL's default RR is examined under this framework, it fails on three fronts:

#### (a) G2-item / G-single: Write Skew Out of Control

Under standard RR, the database must prevent **lost updates** and **read skew** caused by concurrent interleavings. The Hermitage tests expose MySQL's fatal failure:

- **Scenario (write skew)**: two concurrent transactions each read a snapshot showing "one doctor on duty". Both decide to go on leave (rule: at least one must remain). Both see the snapshot saying it is safe. Both commit. Result: zero doctors on duty.
- **MySQL's behavior**: RR refuses to acquire read locks on ordinary `SELECT`s, so both transactions read the same snapshot and modify the variable without conflict detection. Both commit successfully—corrupted data lands on disk.
- **PostgreSQL's behavior**: PG's RR implements **Serializable Snapshot Isolation (SSI)**—it detects serialization conflicts in the concurrency graph and aborts one transaction with a serialization failure.

This is not a theoretical edge case. It is the canonical G2-item anomaly that any serious RR implementation must handle.

#### (b) MAV Violation: Time Flowing Backwards ("Time Flowing Backwards")

**MAV (Monotonic Atomic View)** is a basic causal consistency invariant: within a transaction, if your first query observes the effects of another committed transaction, subsequent queries must continue to see those effects (or newer ones)—data must never "go backwards".

MySQL's MVCC implementation violates this due to the lax timing of **Read View** creation and refresh:

1. Transaction A's first query sees data just committed by transaction B (the latest snapshot).
2. Transaction A's second query, under some highly concurrent interleaving patterns, reads a *stale* snapshot from *before* B committed.

This **"seeing the future first, then going back to the past"** is classified in distributed systems literature as a severe **consistency violation**. It means MySQL's RR cannot even guarantee that a single transaction's view of reality is monotonically advancing.

#### (c) Lost Update: The Snapshot Read Trap

The "Eight Immortals diagram" defines eight models of advanced concurrency conflict. In the **lost update** test, MySQL exhibits almost deceptive behavior:

- Two transactions concurrently execute `salary = salary + 1000` on the same row.
- In a strict RR database, the second transaction's read should block until the first commits, ensuring a correct accumulation.
- MySQL's snapshot read implementation allows both transactions to read the same stale value (e.g., 5000), compute independently (5000 + 1000 = 6000), and write back. One transaction's update is silently overwritten and lost.

The fix requires an explicit `SELECT ... FOR UPDATE`—but that is a *user-side workaround* for a consistency guarantee the isolation level *claims* to provide.

### 7.5 Root Cause: the InnoDB–SQL Layer Split ("The Beautiful Disconnect")

Why is MySQL's consistency so much worse than its documentation suggests?

The root cause is a **fundamental architectural disconnect** between the InnoDB storage engine layer and MySQL's SQL parsing/MVCC coordination layer:

1. **Marketing vs. reality**: MySQL advertises its isolation levels based on an "ideal single-threaded business flow"—a best case that never exists in production.
2. **Black-box testing exposes it**: tools such as Elle and Jepsen inject microsecond-level concurrent interleavings, specifically designed to probe gaps in the causal chain within transaction implementations. MySQL's "lock-free snapshot reads"—optimized for speed—create enormous holes under concurrent access.
3. **The split**: InnoDB's MVCC snapshot mechanism runs independently of the SQL layer's transaction coordination. The SQL layer *believes* it provides RR guarantees, but InnoDB's snapshot semantics are weaker than RR requires. Result: a database that *claims* repeatable read but in practice delivers something between read committed and unpredictable behavior, depending on concurrency timing.

**Practical implication**: if your business has hard consistency requirements (financial reconciliation, distributed inventory, concurrent balance updates), you **cannot trust MySQL's nominal isolation level**. You must:
- Add pervasive pessimistic locking (use `SELECT ... FOR UPDATE` for every read that feeds a write)—effectively degrading to manual serialization in application code.
- Migrate to PostgreSQL (SSI-based RR), TiDB (true distributed transactions), or CockroachDB (serializable by default).

The Hermitage tests are the final nail in the coffin: they prove from first principles that MySQL's "RR = SI + phantom read illusion" is not only weaker than advertised—it actively violates the causal consistency invariants guaranteed even by basic SI implementations.

## 8. MySQL 8.0 Improvements: Catching Up or Leading?

MySQL 8.0 introduced significant feature improvements: window functions, common table expressions (CTEs), JSON enhancements, multi-valued indexes, invisible indexes, descending indexes. These are real engineering advances.

But taken together, **MySQL 8.0 has only reached the baseline level of other databases, and may not even have fully reached it**.

**The SQLite comparison**: SQLite implemented window functions, CTEs, FTS5 full-text search, JSON1, and R-Tree spatial indexing in about 100,000 lines of code. MySQL 8.0 took millions of lines of code to add the same features, and still lags on JSON support (a TEXT alias vs. binary JSONB) and vector search. This is not a gap in engineering effort, it is a gap in design capability—a project of 100,000 lines of code crushes a project of millions of lines of code on features, showing how extremely inefficient the latter's design is.

**Commercial strategy after the Oracle acquisition**: after Oracle acquired Sun in 2010, MySQL's feature evolution visibly slowed. JSON dragged until 8.0 (2018), and vector search still has no official support today. This is not a technical capability problem—Oracle has engineering capability on par with PostgreSQL. This is commercial strategy: the more capable MySQL is, the less users will migrate to Oracle Cloud or buy an Oracle commercial license. MySQL's missing features are a **deliberate product positioning**, not technical debt.

| Feature | MySQL 8.0 | PostgreSQL 16 | SQLite 3.45 | DuckDB |
|------|-----------|---------------|-------------|--------|
| Window functions | ✅ | ✅ | ✅ | ✅ |
| CTE | ✅ | ✅ | ✅ | ✅ |
| JSON support | ⚠️ TEXT alias | ✅ JSONB (binary) | ✅ JSON1 | ✅ JSON |
| GIS | ✅ Basic | ✅ PostGIS (full) | ❌ | ❌ |
| Full-text search | ✅ | ✅ tsvector | ✅ FTS5 | ✅ FTS |
| Vector search | ❌ | ✅ pgvector | ✅ vec0 | ✅ Naturally supported |
| Materialized views | ❌ | ✅ | ❌ | ✅ |
| Time-series data | ❌ | ✅ TimescaleDB | ❌ | ✅ Columnar, naturally suited |
| Replication | ✅ Primary/replica/GTID | ✅ Logical/physical | ❌ | ❌ |
| Concurrent writes | ✅ Multi-user | ✅ Multi-user | ⚠️ Single writer | ⚠️ Single writer |
| Deployment complexity | Medium (server-side) | Medium (server-side) | Very low (embedded) | Very low (embedded) |

**Key findings**:
- MySQL 8.0's feature improvements are catching up with what PostgreSQL/SQLite already had, not innovation
- **The essential gap in JSON support**: MySQL's JSON type is stored internally as TEXT—it validates the format on insert and provides convenience functions on query (`JSON_EXTRACT`, `->>`), but there is no binary decomposition format. Indexing is possible only through multi-valued indexes on predefined paths. PostgreSQL's JSONB uses binary decomposed storage, supporting GIN indexes (automatically indexing all keys and values), containment queries (`@>`), and key existence checks (`?`). This is not a question of "how big the gap is"—MySQL JSON is TEXT under a different name, PostgreSQL JSONB is a true JSON data type. MySQL's approach to JSON is **slapping on a new name to ride the hype**, exactly the same as its handling of vector search.
- MySQL has no significant feature advantage over SQLite—SQLite 3.25+ likewise has window functions, CTEs, and FTS5
- MySQL's only differentiating advantage is its **mature primary-replica replication ecosystem**—but this is not a feature advantage, it is operational convenience

## Conclusion

MySQL is the superposition of problems from two eras—in the early period a product of the designer's amateurism in the database domain, in the later period a tool of Oracle's commercial strategy. Engineering compromises disguised as design decisions, missing features disguised as technical debt.
1.  **Type system**: storing time as a string is extremely amateurish—no direct arithmetic, low index efficiency, unclear semantics; fixed-length strings (`CHAR`) forge a static form for a type that has only dynamic semantics, and are even worse on the very index efficiency dimension they advertise.
2.  **SQL semantics**: broken `GROUP BY` logic for the sake of convenience—a set-theoretic type error celebrated as "ergonomics".
3.  **Optimizer**: stupid join order and rigid indexing requiring manual hacks (`STRAIGHT_JOIN`).
4.  **Terminology**: WAL→Redo Log, Schema→Database—engineering pragmatism above theoretical correctness.
5.  **Scaling**: clustered index obsession→sharding superstition. PostgreSQL/Oracle handle billions of rows through native partitioning and parallel query—zero application surgery.
6.  **Storage engine**: a unique historical accident that forces users into a choice other RDBMSs do not require.
7.  **Isolation levels**: RR as a cover for a replication defect; Serializable as brute-force locking; phantom read prevention as an MVCC illusion.
8.  **Consistency (Hermitage-verified)**: write skew (G2-item) out of control; MAV monotonic view violation (time-travel reads); lost updates through the snapshot trap. The InnoDB–SQL layer split means MySQL delivers less than its nominal isolation level promises under concurrent workloads.

It remains the "safe" choice for legacy enterprises, but it fails first-principles testing against modern architectures.

**Alternative recommendations**:

| Scenario | First choice | Rationale |
|:--|:--|:--|
| **OLTP (transaction-heavy)** | PostgreSQL | SSI serializable, JSONB, parallel query, native partitioning |
| **Embedded / standalone** | SQLite | Zero deployment, ACID, window functions, FTS5 |
| **Analytical queries (OLAP)** | DuckDB | Columnar storage, vectorized execution, zero deployment |
| **Multi-model / graph+document+relational** | SurrealDB | SurrealQL unified queries, graph traversal, built-in permissions |
| **Distributed transactions** | CockroachDB / TiDB | Serializable by default, horizontal scaling |
| **Legacy migration** | PostgreSQL | Lowest cost to migrate from MySQL, best ecosystem compatibility |

**New projects**: skip MySQL entirely. PostgreSQL equals or surpasses it in every dimension—ACID, scalability, JSONB, full-text search, geospatial, materialized views. MySQL does not genuinely win in a single dimension.

---

## Cross-References

This document forms a complete decision loop with the following architecture analyses:
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: Redis as the "network RAM trap", a core component of the legacy stack alongside MySQL.
- **[Nginx Critique](nginx-critique-en.md)**: Nginx's configuration complexity shares the same root as MySQL's SQL anti-patterns—both are engineering compromises from the 2000s.
- **[Unified Data Layer](unified-data-layer-en.md)**: A modern multi-model database, directly compared with MySQL's relational legacy.
- **[Lakehouse Research](lakehouse-research-en.md)**: Distributed storage alternatives—DuckDB/S3 Tables replacing MySQL for analytical workloads.
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: The selection philosophy of Rune/Steel/Koto matches that of MySQL vs PostgreSQL—reject legacy design, embrace modern architecture.

**The unified first principle**: no technology worship, no swallowing the big pie the open-source world paints; look only at real physical hardware limits and team productivity.
