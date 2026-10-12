# Consensus Protocols: The Nature, Boundaries and Proper Use of Raft

> **Languages:** [English](consensus-protocol-en.md) (primary) · [中文](consensus-protocol.md)

**Status:** Initial version
**Core thesis:** Raft is a metadata consensus protocol, not a data storage protocol. Using it for bulk data storage is a category error.
**Architecture context:** [KV Storage Engine](kv-storage-engine-en.md) — the role of Raft in the dual-engine model
**Serialization relation:** [Serialization Protocol Comparison](serialization-protocol-comparison-en.md) — encoding choices for the Raft control flow

---

## The Core Mechanisms of Raft

Raft is a consensus algorithm proposed by Diego Ongaro and John Ousterhout in 2014, with the goal of understandability — compared with Paxos, Raft decomposes the consensus problem into three independent subproblems, each with a clear solution.

### Leader Election

Nodes in the cluster have three roles: **Leader**, **Follower**, **Candidate**.

- Under normal conditions there is only one Leader, the rest are Followers
- The Leader periodically sends heartbeats to Followers to maintain authority
- A Follower that does not receive a heartbeat within the timeout period → becomes a Candidate → initiates an election
- The Candidate requests votes from all nodes (RequestVote RPC)
- Upon obtaining a majority of votes → becomes the new Leader

**Key constraint**: each term admits at most one Leader. A term is a logical clock with an incrementing number, used to detect stale information.

### Log Replication

The Leader receives a client write request → appends it to its local log → replicates it to Followers via AppendEntries RPC → after majority confirmation it **commits** → applies it to the state machine.

```
Client → Leader: put("key", "value")
Leader: append to local log → AppendEntries RPC → Follower 1, Follower 2
Follower 1: write to local log → ACK
Follower 2: write to local log → ACK
Leader: receives 2/3 ACK → commit → apply to state machine → reply to Client
```

**Log Matching Property**: if two nodes have the same term at some index, then all entries before that index are also the same. This is the mathematical basis of Raft's correctness.

### Majority Confirmation

Raft's safety relies on a simple mathematical fact: in a cluster of N nodes, any two majorities must intersect. This means:

- There cannot be two Leaders within one term (two Leaders would each have to obtain a majority vote, and the majorities must overlap in some node, which will not vote twice)
- Committed logs will not be overwritten (a new Leader necessarily contains all committed logs, because it obtained a majority, and a commit requires majority confirmation)

**Fault tolerance**: N nodes tolerate ⌊(N-1)/2⌋ failures. 3 nodes tolerate 1 failure, 5 nodes tolerate 2 failures.

---

## What Raft Is Good For: Metadata Consensus

The essence of Raft is a **metadata consensus protocol** — it provides strong consistency guarantees for a small amount of critical decision data.

### etcd / Kubernetes: Raft's Typical Home Turf

etcd is the sole data storage backend of Kubernetes and uses Raft to guarantee metadata consistency. Typical data scale:

| Dimension | Typical value |
|:---|:---|
| Total data | Tens of MB to hundreds of MB |
| Single record | Hundreds of bytes to several KB (Pod, Service, ConfigMap) |
| Write frequency | Tens to hundreds of QPS (depends on cluster size) |
| Read frequency | Thousands of QPS (all kubelet/watcher polling) |
| Node count | 3 or 5 (etcd's official recommendation) |

What etcd stores is **data about data** (metadata about data): which Pods run on which node, the endpoint mapping of Services, the configuration contents of ConfigMaps. Such data is small in volume and low in change frequency, yet must be highly consistent — a stale Service endpoint may cause requests to hit a Pod that has already been destroyed.

### Distributed Locks and Leader Election

Raft's leader election mechanism naturally supports distributed locks:

- The Leader itself is the "lock holder" — there is only one Leader in the cluster
- etcd's `lease` + `put` implements a distributed lock: create a lease → bind a key → when the Leader fails the lease expires → other nodes compete
- Uses: distributed task scheduling, Leader election, distributed coordination

### Streaming Metadata

Kafka uses a Raft-like protocol (KRaft, a Raft-based variant) to manage partition metadata:

- Which partitions exist and which Broker holds the Leader
- Offset information of consumer groups
- Topic configuration and partition assignment

Likewise "data about data" — Kafka's actual message data (GB to TB scale) is stored on the Broker's local disk and does not pass through Raft. Raft only manages the metadata layer of "which data is at which location".

### General Characteristics of Consensus Protocols

| Applicable scenario | Core requirement | How Raft satisfies it |
|:---|:---|:---|
| Configuration management | All nodes see the same configuration | Log replication + majority commit |
| Service discovery | Consistent node state | Leader maintains the member list, heartbeat detection |
| Distributed locks | Mutually exclusive access | Leader election is naturally mutually exclusive |
| Distributed transaction coordination | Consistent decisions | The coordinator of two-phase commit uses Raft to elect the primary |

---

## What Raft Is Not Good For: Bulk Data Storage

### The Write Amplification Problem

Every write in Raft must be replicated to N nodes. Storage cost grows linearly with the number of nodes:

```
Raft 3-node cluster writing 1 GB of data:
  Node 1: 1 GB (Leader local write)
  Node 2: 1 GB (Follower replication)
  Node 3: 1 GB (Follower replication)
  Total storage: 3 GB (3x write amplification)
  Total network transfer: 2 GB (Leader → 2 Followers)

Raft 5-node cluster writing 1 GB of data:
  Total storage: 5 GB (5x write amplification)
  Total network transfer: 4 GB (Leader → 4 Followers)
```

**This is not a theoretical problem but a physical reality**. Each additional node doubles the storage cost and network bandwidth consumption. For metadata (MB scale) this is acceptable; for user data (GB to TB scale) it is a disaster.

### Raft Replication vs S3 Replication: Cost Comparison

S3 (and all mainstream object storage) automatically performs 3-replica replication in the background, and the storage cost is already included in the price:

| Dimension | Raft 3-node | S3 (3-AZ replication) |
|:---|:---|:---|
| Replica count | 3 (more can be configured) | 3 (fixed, across AZs) |
| Replication cost | Explicit: one server per node | Implicit: included in the storage price |
| When clients increase | Replica count unchanged (N fixed) | Replica count unchanged (3 AZs fixed) |
| Monthly price per GB | Server cost amortized (far above $0.023) | ~$0.023 (us-east-1) |
| Scaling method | Add nodes = add storage cost | No operation needed, handled automatically |
| Consistency | Strong (linearizable read) | Eventual (but S3 provides read-after-write) |

**Core difference**: Raft's replication cost grows linearly with cluster size (each new node = a new full replica); S3's replication cost is fixed at 3 replicas, independent of the number of clients. Raft is "I manage my own replication"; S3 is "just use me, I manage the replication".

### Storing Data with Raft = Buying a Server for Every Node

Suppose the cluster has 3 Raft nodes; each node needs:

- Storage space equal to the total data volume (each node stores a full replica)
- Enough network bandwidth to handle replication traffic
- Enough CPU to handle consensus overhead

When the data volume grows to 1 TB:
- Raft: 3 servers × 1 TB storage = 3 TB total storage cost
- S3: 1 TB storage × $0.023/GB/month = ~$23/month (including 3-replica replication)

**Conclusion**: Raft's storage cost is tens to hundreds of times that of S3; this is not something engineering optimization can solve — it is the inevitable result of the architectural choice.

### Scaling Characteristics: Why 3-5 Nodes Is the Ceiling

| Node count | Fault tolerance | Write latency | Applicable scenario |
|:---|:---|:---|:---|
| 3 | 1 node failure | 1 RTT (Leader → 2 Followers) | etcd/K8s, small coordination services |
| 5 | 2 node failures | 1 RTT (Leader → 4 Followers) | Critical metadata, high-availability requirements |
| 7 | 3 node failures | 1 RTT (but network overhead is significant) | Theoretically feasible, rare in practice |
| >7 | More fault tolerance | Severe write amplification, rising latency | **Not recommended** |

**Why more than 7 nodes is not recommended**:

1. **Write amplification**: 7 nodes means every write is replicated 6 times, consuming enormous network bandwidth
2. **Rising latency**: although a commit only needs majority confirmation, the network overhead of log replication grows with the number of nodes
3. **Operational complexity**: coordinating, monitoring, and recovering 7 servers is far more complex than 3
4. **Diminishing returns**: going from 5 nodes to 7 nodes tolerates only 1 more failure, yet costs 40% more

---

## Raft vs Other Consensus Protocols

### Raft vs Paxos

| Dimension | Raft | Paxos |
|:---|:---|:---|
| Understandability | High (three independent subproblems) | Low (complex proofs, counterintuitive) |
| Leader dependency | Strong Leader model | Leaderless (symmetric) |
| Log replication | Strictly ordered | May be out of order (Multi-Paxos) |
| Industrial implementations | etcd, TiKV, CockroachDB | Google Chubby, ZooKeeper (ZAB variant) |
| Theoretical completeness | Equivalent to Paxos | More general (tolerates more complex failure models) |

**Paxos's real advantage**: Paxos does not require a Leader — in theory proposals can be made concurrently at any node. This is an advantage in extreme scenarios (such as frequent Leader failures). But in practice the Leader model is simpler, and Raft's Leader Election already handles the Leader failure scenario.

**Paxos's real disadvantage**: the engineering implementation of Multi-Paxos is extremely complex; Google's internal Spanner team took years to stabilize it. The core reason the etcd team chose Raft was "being able to hire engineers who can read the code".

### Raft vs EPaxos

EPaxos (Egalitarian Paxos) allows Leaderless concurrent proposals and in theory offers higher throughput:

| Dimension | Raft | EPaxos |
|:---|:---|:---|
| Concurrent writes | Serial (Leader serializes) | Concurrent (any node proposes) |
| Throughput | Limited by the Leader bottleneck | Theoretically higher (no Leader bottleneck) |
| Implementation complexity | Medium | Extremely high |
| Industrial adoption | Widespread (etcd, TiKV) | Almost none (academic prototypes only) |

**Reality**: EPaxos performs excellently in papers, but almost no one in industry adopts it. The root cause is not that the theory is not good enough, but that the engineering complexity is too high — EPaxos's conflict detection and rollback logic make code review a nightmare. Raft's "good enough" philosophy wins out in engineering practice.

### Raft vs CRDT

CRDT (Conflict-free Replicated Data Types) is a completely different paradigm:

| Dimension | Raft | CRDT |
|:---|:---|:---|
| Consistency model | Strong (linearizable) | Eventual (eventual consistency) |
| Network requirement | A majority must be reachable | Any node can write independently |
| Data conflicts | Not allowed (Leader serializes) | Merged automatically (mathematically guaranteed conflict-free) |
| Applicable scenario | Critical metadata, coordination | High availability, low latency, temporary inconsistency allowed |
| Representative systems | etcd, etcd | Redis CRDB, Apple clock synchronization |

**Correct uses of CRDT**: user state synchronization (online document collaboration), hot configuration updates (eventual consistency suffices), edge computing (write offline then synchronize). CRDT is not a replacement for Raft — they solve different problems.

**Key distinction**: Raft answers "who is right" (strong consistency), CRDT answers "how to merge disagreements" (eventual consistency). The two are not interchangeable.

---

## etcd: Raft's Textbook Case

etcd is the sole data storage backend of Kubernetes and the most successful industrial application of Raft.

### Why etcd Uses Raft

| Requirement | How etcd satisfies it |
|:---|:---|
| Strong consistency | Raft majority commit guarantees linearizable read |
| High availability | 3-5 nodes, tolerating 1-2 failures |
| Small data volume | MB-scale metadata, Raft write amplification acceptable |
| Low latency | Local reads (Leader); Followers can handle stale reads |
| Simple operations | etcd operations are an order of magnitude simpler than a MySQL cluster |

### The Data Scale of etcd

etcd data for a typical Kubernetes cluster:

- 100 Pods: ~100 KB metadata
- 1000 Pods: ~1 MB metadata
- 10000 Pods: ~10 MB metadata
- Including all ConfigMaps, Secrets, Services, Endpoints

**This is Raft's sweet spot**: small data volume (MB scale), low change frequency (tens of QPS), yet extremely high consistency requirements (stale data = cluster failure).

### Anti-patterns of etcd

etcd should not be used as:

- **A general-purpose KV store**: when the data volume exceeds GB scale, Raft write amplification is unacceptable
- **A cache layer**: etcd's read latency (millisecond scale) is far higher than Redis (microsecond scale)
- **A message queue**: etcd does not support high-throughput writes (hundreds of QPS scale)
- **Large-object storage**: a single Value should be < 1.5 MB (etcd's default limit)

---

## Architectural Decision: When to Use Raft, When Not To

### Decision Matrix

| Scenario | Recommended solution | Rationale |
|:---|:---|:---|
| K8s cluster metadata | Raft (etcd) | Strong consistency, small data volume, high availability |
| Distributed locks / Leader election | Raft | Leaders are naturally mutually exclusive |
| Service configuration center | Raft | Configuration changes must be consistent |
| Small-scale data storage (<1 GB) | Raft (Fjall + Openraft) | Acceptable for a single cluster |
| Large-scale data storage (>10 GB) | SlateDB + S3 | S3 has a fixed 3 replicas, cost is linearly controllable |
| Massive data (TB scale) | SlateDB + S3 | Object storage is the only economical choice |
| High-availability cache | A separate solution (Redis/Valkey) | etcd latency is too high |
| Message queue | Kafka / NATS | etcd throughput is insufficient |

### Core Principles

> **Raft is a consensus protocol for "data about data", not a storage protocol for "data".**

Raft's value lies in: **a live majority = the system is available; minority failures are recoverable**. This guarantee is crucial for metadata — if the K8s cluster's etcd goes down, the entire cluster spins out of control. But for user data, S3's "3 replicas + automatic repair" is already sufficient, and it costs two orders of magnitude less.

**Raft + Fjall is not suitable for general-purpose data storage**. 3x write amplification, storage cost that grows linearly with node count, operational complexity — these are the inherent costs of using Raft for data storage. For data storage:

- **Small scale** (<1 GB): a single-node Fjall suffices, Raft is not needed
- **Large scale** (>10 GB): SlateDB + S3, leveraging S3's fixed 3-replica replication

The correct way to use Raft: etcd for metadata, K8s for orchestration, distributed locks for coordination. Do not use it to store your user data.

### Raft Choices in Distributed Databases: TiDB

TiDB is the representative architecture of "Raft used for data storage" — and the best case for understanding the storage cost of Raft. It is not a negative example but rather the **correct architecture for a strongly consistent distributed database**, only with clearly defined applicability boundaries.

**TiDB architecture**:
```
SQL layer (TiDB)
    ↓
PD (Placement Driver) ← routing: key → region → node
    ↓
TiKV nodes (each node stores a subset of regions)
    ├── Region 1: Raft Group (3 replicas)
    ├── Region 2: Raft Group (3 replicas)
    ├── ...
    └── Region N: Raft Group (3 replicas)
        ↓
    RocksDB (local storage)
```

**Key design: it is not 1 Raft managing all data, but N small Rafts each managing one shard.** Each Region has an independent 3-replica Raft Group; a write goes only through the Raft of 1 Region (3 nodes), not all nodes. Write amplification is always 3x and does not grow with cluster size.

**Solution to the node-count limit**:

| Problem | TiDB's solution |
|:---|:---|
| A single Raft cannot exceed 7 nodes | Each Region has only 3 replicas and does not grow with the cluster |
| Too much data to fit | Regions split automatically (similar to sharding) |
| Hotspot Region | PD detects it → migrates it to an idle node |
| Node scaling in/out | Regions rebalance automatically |

**But 3 replicas = 3x storage cost, and that has not changed.**

| | TiDB (Raft 3x) | SlateDB + S3 |
|:---|:---|:---|
| Actual storage for 100TB of data | 300TB (3 replicas) | 100TB (S3 internal replication) |
| Storage unit price | Local SSD $100-200/TB/month | S3 $23/TB/month |
| Monthly cost | $30,000-60,000 | $2,300 |
| Write latency | < 1ms (local Raft) | 1-10ms (S3 PUT) |

**Applicability boundary**: TiDB was the optimal solution in 2018 — low latency + strong consistency + private deployment. Once SlateDB + S3 matured in 2026, most scenarios no longer need this architecture. But for latency-sensitive scenarios such as financial transactions and real-time gateways, TiDB is still the correct choice.

→ For distributed KV architecture selection, see [KV Storage Engine](kv-storage-engine-en.md) §11.

### Fjall + Raft vs SlateDB + S3: Storage Architecture Comparison

| Dimension | Fjall + Raft | SlateDB + S3 |
|:--|:--|:--|
| Source of truth | Local NVMe (Raft majority confirmation) | S3 bucket (11 nines of reliability) |
| Write latency | μs scale (MemTable, same as SlateDB) | μs scale (MemTable, same as Fjall) |
| Storage cost | 3x (Raft replicas) + local SSD unit price | 1x (S3 internal replication) + S3 unit price (20x lower) |
| Scale-to-Zero | Must protect the local disk | Just destroy the Pod, S3 is durable |
| Cross-region replication | Raft across data centers (complex) | S3 CRR (a configuration item) |
| Operational complexity | High (Raft cluster + sharding) | Low (stateless compute + S3) |
| S3 dependency | None | Required |

**Verdict**: the two have essentially the same write performance (both batch into a MemTable). The difference lies in the deployment model and cost — SlateDB + S3's storage cost is 20x lower and operations are simpler, making it the default choice for most scenarios. Fjall is considered only when S3 cannot be used (private deployment, offline), and it has structural advantages in large-Value scenarios (KV separation), complex local transactions, and extreme local performance. Fjall has no official plan to support S3.

---

## Implementation Perspective: Openraft Modules, the Main Line and Pitfalls

From the standpoint of "implementing a Raft application with Openraft": first look at the module map, then follow one main line (log → reducer → business data → snapshot) mapping to the interfaces, then the auxiliary modules, and finally the pitfalls. This is based on **0.9.x**; for exact signatures, refer to [docs.rs/openraft](https://docs.rs/openraft). For clarity, we use the simplest KV command set as an example: business commands `Put(key, value)` / `Delete(key)`, with state being a `HashMap<key, value>`.

### Overview: Main Modules

- `openraft` + `openraft-macros` — **Library**: the consensus engine itself + the macros for defining types.
- `Raft` (the `raft` module) — **Engine handle**: the assembled consensus runtime; externally it submits commands, queries, and membership changes.
- `RaftTypeConfig` (`type_config`) — **Type binding**: `declare_raft_types!` pins down the generics such as NodeId / Node / Entry / snapshot.
- `RaftNetwork` + `RaftNetworkFactory` (`network`) — **Transport hooks**: inter-node RPC (append_entries / vote / snapshot).
- `RaftLogStorage` (`storage`, including `RaftLogReader`) — **Log store**: stores the log + hard state + boundary accounting.
- `RaftStateMachine` (`storage`, including `RaftSnapshotBuilder`) — **Business state**: the `apply` reducer + snapshots.
- `Config` — **Engine parameters**: election timeout / heartbeat / snapshot policy.
- **Support types** — data: `Entry` / `EntryPayload` / `LogId` / `Vote` / `RaftMetrics` / `Membership`.

**Division of labor in one sentence**: the engine takes care of "replication, commit, election" for you — you only connect "types, transport, log, business" through four hooks (TypeConfig + Network + LogStorage + StateMachine).

### Main Line: Log → reducer → Business Data → Snapshot

The main line of understanding Openraft is these four concepts, each link joined to the next:

```
Log (ordered command stream)──reducer──→ Business data (state after applying)──snapshot──→ Control log size
```

- **Log = commands + consistency metadata**. Each entry = `log_id` (term + leader + index) + `context` (optional) + `payload` (command). The log store additionally holds hard state (`current_term` + `voted_for`, preventing split-brain duplicate voting) and boundary accounting (`last_log_id` / `last_purged_log_id`). Only `payload` is a business command; the rest is metadata that lets all nodes reach consensus "on the same sequence of commands, in the same order, in the same term".
- **Business data = the state after applying**. The log is "how it came to be" (the command stream); the business state is "what it is now" (the result).
- **reducer(`apply`) = `(state, command) → state`**: folds log commands into the business state in index order; the core of determinism — same command, same initial state necessarily yields the same result.
- **Snapshot = freezing business data, controlling log size**. A log that only grows and never shrinks will expand without bound, so the state up to `last_applied` is frozen into a snapshot and that log prefix is deleted; lagging nodes catch up via the snapshot instead of replaying the entire log.

> Terminology: `apply` (reducer) is formally the transition function of a (large state space, possibly infinite) deterministic automaton — but in the engineering register of "state machine pattern" it does not count as a "state machine", so **reducer** is more apt. For the full analysis see [Automaton vs State Machine](automaton-vs-state-machine-en.md).

The snapshot compaction loop:

```
Log [1][2]...[X] | [X+1][X+2]...      ← X = the snapshot's last_applied
apply advances to X
(1) build_snapshot    freeze "the state up to X" into a snapshot (in this example = serialize the HashMap)
(2) purge_logs        delete log entries ≤ X ── the snapshot has taken over this prefix
(3) install_snapshot  a lagging/new node does not replay [1..X], it ingests the snapshot directly and sets last_applied=X to continue catching up
```

Key points: a snapshot is a freeze of the business data (the result), not a copy of the commands; the trigger point of `purge_logs` is the snapshot point `last_applied` rather than the quorum (the quorum governs "what is committed", compaction governs "how far to delete" — the two are orthogonal); deleting the prefix is safe because the snapshot has taken it over. Cost: after deleting the prefix you lose command-level replay / audit history; if auditing is required, commands must be stored separately (such as in an S3 archive).

**The essence of the main line: this is Event Sourcing.** Map the three layers —

| Event sourcing concept | Main line (the Openraft layer you implement) | Role |
|:---|:---|:---|
| Event log (source of truth) | `RaftLogStorage` (log store) | append-only ordered changes, the sole source of truth |
| Projection (current state) | `RaftStateMachine::apply` (reducer) | folds the log into the business state |
| Snapshot checkpoint | `build_snapshot` / `purge_logs` | freezes the projection, controls log size |

In other words: **the main line the user must implement — log store + reducer(apply) + snapshot — is itself a complete event-sourced system**: the log is the source of truth, the state is the projection, the snapshot is the compaction. What the openraft framework adds on top of this event sourcing is **consensus and consistency** (leader election, majority confirmation, quorum, membership changes, log replication and catch-up, snapshot distribution), so that under multi-replica, distributed conditions all replicas agree on the same log, the same order, the same term.

This line extends further — **traditional databases like PostgreSQL have always been event sourcing at the log level**:

| | Log (source of truth) | Business state | Consensus? |
|:---|:---|:---|:---|
| Raft / Openraft | replicated log | the state produced by `apply` | ✅ majority confirmation + leader election + linearizability |
| PostgreSQL | WAL | tables / indexes | ❌ only one-way replication (primary → replica); the replica replays the projection of the WAL |

PostgreSQL's WAL is the event log; the tables are the state projected by a reducer (a deterministic replay of the WAL); primary-replica replication broadcasts the log one-way to replicas for replay. What it lacks is precisely **consensus** — a replica has no arbitration authority, no leader election, no quorum, so there is no question of multiple replicas reaching a majority on "commit". Therefore **the difference between Raft-replicated storage and PostgreSQL is not whether they are event sourcing (both are), but whether there is a consensus layer above the log to guarantee multi-replica linearizability.**

### Openraft Modules and Interfaces for the Main Line

- **Log data / command stream** → `RaftLogStorage`: `get_log_state` / `get_log_entries` / `try_append_entry` / `save_vote`; `purge_logs` is called by compaction.
- **Business data (state)** → `RaftStateMachine`: the state is held in the `apply` you implement.
- **reducer** → `RaftStateMachine::apply`: `apply(entries)` folds into the business state in log index order.
- **Snapshot (controlling log size)** → `RaftStateMachine` / `RaftSnapshotBuilder`: `build_snapshot` / `get_current_snapshot` / `install_snapshot`; `purge_logs` lives in `RaftLogStorage`; `Config.snapshot_policy` sets the cadence.
- **Command submission entry point (producer side)** → `Raft<C>`: `client_write(cmd)` submits → the engine replicates / commits → triggers `apply`.

Responsibility boundaries of the main line: the log store and the network are **blind store, blind read** (the engine passes raw entries, you store / retrieve them as-is, do not deserialize commands); only two places really "understand" the business — `apply` (runs commands) and the snapshot (reads/writes the business state).

**What you need to write:**
- `RaftLogStorage` — dumb persistence, blind store, blind read.
- `RaftStateMachine::apply` — **business core**: deserialize the command → state transition.
- `build / install_snapshot` — serialize / deserialize the business state.
- `RaftNetwork` — mechanical wiring.
- `RaftTypeConfig` — pure type glue.

### Auxiliary Modules

- **Network**: `RaftNetwork` (`append_entries` / `vote` / `snapshot`) wraps the consensus messages the engine wants to send into your wire format; `RaftNetworkFactory` creates one instance per target node. The engine calls you; mechanical wiring.
- **Type binding**: `RaftTypeConfig` (`declare_raft_types!`) fixes the NodeId / Node / Entry / snapshot types — pure type glue, no logic.
- **Engine handle `Raft<C>`** (you call the engine): `initialize()` creates the cluster on first start; `metrics()` queries the role / leader / term (use it to **locate the leader**); `change_membership` / `add_learner` / `remove_learner` adjust capacity online; `trigger_log_compaction` / `purge_log` proactively advance snapshots; `shutdown()` takes it offline.
- **Parameters `Config`**: election timeout / heartbeat / `snapshot_policy` (e.g. trigger a snapshot every N applied entries), etc.
- **Support types**: `Entry` / `EntryPayload` / `LogId` / `Vote` / `RaftMetrics` / `Membership`.

You do not need to manage these at all (the engine handles them automatically): term / index / leader election / voting / quorum, log replication / commit advancement / follower catch-up / snapshot trigger cadence.

### Pitfalls

- **Determinism**: `apply` must be pure — same command, same initial state necessarily yields the same result, otherwise replicas diverge. This is the precondition for all correctness.
- **Persistence**: the log + hard state must be durable (fsync / written to disk); production must not use a pure in-memory store.
- **Version differences**: 0.8 uses `RaftTypes` + a single `RaftStorage`; 0.9+ uses `RaftTypeConfig` (a trait) + a split into `RaftLogStorage` / `RaftStateMachine`. Before writing, check against the version you use (openraft ships with examples such as raft-kv-rocksdb).
- **Single-leader semantics**: `client_write` can only be sent to the leader; use `metrics()` to locate it and redirect.
- **Error routing**: handle `StorageError` (disk / IO) and `ApplyError` (business rejection) separately; do not merge them into one path.
- **Snapshot safety chain**: `purge_logs` only deletes entries ≤ last_applied — "committed → applied → covered by a snapshot → only then may it be purged"; no index that is uncommitted / unapplied may be deleted.
- **Do not deserialize business commands in LogStorage**: the log store is blind store, blind read; commands are interpreted only in `apply`.
- **Network RPCs may be retried / may time out**: a follower will briefly lag, and catch-up relies on retries and snapshots; do not treat "a single RPC failure" as fatal.

---

## Cross-references

This document forms a complete decision loop together with the following architecture analyses:

- **[KV Storage Engine](kv-storage-engine-en.md)**: the Fjall/SlateDB dual-engine model (metadata is independent per node, no consensus)
- **[Serialization Protocol Comparison](serialization-protocol-comparison-en.md)**: encoding choices for the Raft control flow (Protobuf vs Bincode)
- **[Distributed Collaboration Topology](distributed-collaboration-topology-en.md)**: topological generalization of Raft's positioning — control-plane/data-plane division of labor, large-node federation and flexible transactions, CRDT's boundary of sovereignty
- **[Aura Architecture](aura-architecture-en.md)**: the complete architectural position of metadata single-write (no consensus) and local data planes
- **[HelixDB vs LanceDB](helixdb-vs-lancedb-en.md)**: comparison of two paths for an AI data stack on object storage
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: why Redis was replaced by KV