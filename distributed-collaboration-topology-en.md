# Distributed Collaboration Architecture: Topology and Consistency as Orthogonal Choices

> **Languages:** [English](distributed-collaboration-topology-en.md) (primary) · [中文](distributed-collaboration-topology.md)

**Status:** Initial version
**Core thesis:** Topology/trust boundary (who owns the data, through whom the data passes) and the conflict-resolution algorithm (how changes are aligned) are two orthogonal dimensions; architectural choice is the cross-product combination of the two rather than a single spectrum; massive data crossing ownership boundaries goes with "large-node federation + flexible transactions" (federated nodes authenticate via the well-known protocol, metadata is independently single-written per node, no consensus), the personal-device-sovereignty scenario goes local-first (CRDT); strong consensus (Raft/Paxos) is reserved for data-level strong-consistency replication (external TiKV-class solutions); the ceiling of the ordering layer is given by causal order — global order is not observed, it is bought (price = one round-trip RTT + an explicit ε), so the legitimate form across domains is one-way events and post-hoc alignment, rather than global ordering.
**Architecture context:** [Consensus Protocol](consensus-protocol-en.md) — Raft's mechanisms and boundaries; [Aura Architecture](aura-architecture-en.md) — the engine implementation of metadata single-write and a local data plane

---

## 1. Two Orthogonal Dimensions

To analyze distributed collaboration systems, first peel apart two dimensions that are often conflated:

- **Dimension A: Topology and Trust Boundary** — who identity belongs to, who data ownership belongs to, and which nodes the data physically passes through. This is a problem of ownership and trust; no protocol can solve it.
- **Dimension B: Conflict Adjudication Algorithm** — when multiple nodes change concurrently, by what rules the machine aligns the accounts. This is an algorithmic problem, and it does not care who the data belongs to.

Traditional discourse treats "distributed" as a single spectrum from single-machine to decentralization, when in fact it is the independent values of these two dimensions in cross-product combination. Grouped by combined form, they fall into five classes of models:

| Model | Dimension A: Identity and Ownership | Topology | Dimension B: Consistency Algorithm | Representative Systems |
|:--|:--|:--|:--|:--|
| Vertically scaled monolith | A single organization, absolute center | Giant single node | Single-machine concurrency locks / ACID (MVCC, no cross-network consensus) | Early mainframes, single-machine Oracle/DB2 |
| Centralized strong consensus | Unified trust domain (guards against crashes, not betrayal) | Same-city/LAN peer nodes | Raft / Paxos / 2PC (Leader dictatorship, a single whole-network view) | etcd, ZooKeeper, TiDB-PD, Spanner |
| Large-node federation | Organization-level distributed hosting; nodes mutually distrustful and peer-equal, identity bound to the owning node | Star-shaped federation mesh, each node itself a heavy monolith | Flexible transactions / eventual consistency (TCC / Saga / asynchronous reconciliation, multiple local views) | Email/SMTP, Matrix, ActivityPub |
| Decentralized relay | Absolute user sovereignty (public key is the ID, a server cannot ban) | Dumb pipes: massive relays that do not communicate with one another | Client multicast and deduplication (clients pull fragments, sort, verify) | Nostr |
| Local-first | Highest sovereignty of the personal device; data belongs locally first, the network is only a sync channel | Pure edge P2P (LAN/Bluetooth/DHT, no servers) | CRDT (monotonic lattice theory, out-of-order input, mathematical self-convergence) | Figma's underlying layer, Automerge, Yjs, Anytype |

When making the choice, answer Dimension A first — the data's ownership and trust boundary is a business fact, not a technical preference; Dimension B's algorithm is then optimized within that boundary. Doing it the other way round (picking the algorithm first and then cramming ownership in) yields category errors such as "unify everything with Raft" or "do the books with CRDT".

## 2. The Nature of Strong Consensus and Its Physical Ceiling

### 2.1 Simulating a Single Machine, Not Scaling

The engineering purpose of Raft/Paxos is not to scale out, nor to decentralize — it is to mask single-machine failure with multi-machine redundancy, simulating an absolutely reliable "single-machine experience" (linearizability) at the application layer. A Leader within the group dictates write orchestration, Followers have no decision sovereignty, and their reason to exist is as election candidates and read-only replicas. This is consistent with the positioning of [Consensus Protocol](consensus-protocol-en.md): **Raft is a metadata consensus protocol, not a data storage protocol**.

### 2.2 The Wide Area Network Is the Physical Ceiling

A write must wait for a majority to persist before it can return success, and on a wide-area network this is nailed down by physical latency:

- Cross-city RTT 30-40 ms (a three-city cluster of Beijing–Shanghai–Guangzhou); under high concurrency each transaction holds its lock for tens of milliseconds → connection pool exhaustion, thread blocking, a throughput cliff.
- Write amplification grows linearly with node count (3 nodes = 3x storage + 2x network transmission) — see [Consensus Protocol §Write Amplification](consensus-protocol-en.md).

This is not something engineering optimization can eliminate — it is the product of the speed of light and majority arithmetic. What can eliminate it is an architectural choice: do not put data that spans a wide area into the same consensus group.

**But "nailed down" must be read as a latency floor, not a prohibition.** Cross-region strong consistency does work in production (Spanner, CockroachDB, FoundationDB); the price is one round-trip RTT per commit — PACELC has already written it as a formula: even without a partition there is a trade-off between latency and consistency. Describing it as "defying the laws of physics" is rhetoric; in reality one is merely paying the speed-of-light bill. What genuinely degrades with scale are three quantities:

| Scale | One-way propagation | RTT |
|:--|:--|:--|
| Same data center (10 km, fiber at 2/3c) | 33 µs | 67 µs |
| Earth–Moon | 1.28 s | 2.56 s |
| Earth–Mars at closest approach | 4.3 min | 8.6 min |
| Earth–Jupiter | 43 min | 1.4 h |
| Voyager 1 (163 AU) | 22.6 h | 45 h |
| Proxima Centauri (4.24 ly) | 4.24 years | 8.5 years |
| Milky Way diameter (1e5 ly) | 1e5 years | 2e5 years |

(A light path of 1 ly is exactly 1 year; the same-data-center row is taken at fiber 2/3c, the rest are order-of-magnitude values extrapolated from vacuum light speed and distance, not measurements.)

- **Critical path = diameter**: commit latency, election timeout, and failure-recovery time are all simultaneously equal to propagation time. At galactic scale there is no such thing as "detecting that the Leader died" — you would have to wait 2e5 years to find out; the online protocol degenerates into a batch protocol, and **its latency becomes its period**.
- **Membership stability requires lifetime/RTT ≫ 1**: in the same data center this ratio is 1e8 (a process lives for hours, RTT is 100 µs); to obtain the same ratio at galactic scale, member lifetime would have to be 2e13 years. This is the genuinely impossible item — not that the algorithm does not hold, but that "membership" does not hold along the time dimension: one cannot distinguish "the node is dead" from "the reply is still in flight", and a consensus protocol must make that distinction bounded.
- **Throughput is pinned down by RTT**: the throughput ceiling of a single shared resource = the leader's flight window / RTT. Same data center 1.5e10 ops/s, Earth–Jupiter 193 ops/s, Milky Way 5 ops/year (with a window of 1e6). This one cannot be improved — **the throughput of global consensus is independent of the number of domains, whereas the throughput of federation grows linearly with the number of domains**.

The conclusion is layered, not "Raft doesn't work": a single consensus group covering the whole scale is mechanically correct but systemically useless (the theorem lives, the system dies); pulling the right to agree back inside the light cone, with each domain running its own local consensus group, is feasible; across domains there remains only one-way events and post-hoc alignment. Arriving at the third step yields exactly federation — **federation is not a fallback, it is the only form derived from the light cone**.

### 2.3 The Storage Engine Is the Second Bottleneck

etcd's underlying layer is a pure-Go single-machine MVCC KV (BoltDB), with its default quota hard-locked at 2-8 GB, beyond which performance avalanches even with Compact/Defrag, because of the single-entry ordering of the Raft log. A single-machine embedded engine (the RocksDB-family LSM-Tree) is 1-2 orders of magnitude higher in write IOPS and capacity ceiling in the purely vertical direction — "the consensus layer's storage bottleneck" and "the cost of the consensus protocol itself" are two independent problems, both pointing to the same conclusion: strong-consensus groups keep data volumes small.

### 2.4 The Ceiling of the Ordering Layer: the Only Observable Order Is Causal Order

Beyond latency there is a harder boundary: it lies not in "how long until it returns", but in "what order is observable".

- **The only frame-independent order is causal order.** In special relativity simultaneity is relative, but light-cone order is not — all observers agree on "whether event A falls inside the past light cone of event B". Lamport's 1978 *Time, Clocks, and the Ordering of Events* derived the logical clock precisely from the step of "there is no global time". So "in the end only causal consistency can be achieved" is not an engineering concession; **the ceiling comes from an invariant**.
- **Global order is not observed, it is bought.** The price = one round-trip RTT + a time uncertainty ε that must be explicitly acknowledged: first ε must be squeezed small (GPS + atomic clocks), then the protocol must wait out that ε (TrueTime's commit-wait). It can be bought, but it is always a negotiated product, not an observed fact. The accurate formulation therefore has two layers: what does not exist is **observable global simultaneity**, and **global order without paying a price**; what does exist is a global-order assertion carrying an ε.
- **Don't treat relativity as a universal metaphor.** Its role in this chain is structural (simultaneity is relative ⇒ global order can only be negotiated), not one of magnitude: a jet covering 1e4 km has a simultaneity deviation of only 33 ns, whereas data-center NTP jitter is on the order of 1e-4 s, a factor of 3000; relativity's real engineering bill is in GPS — a daily actual payment of +38.5 µs (special −7.2 / general +45.7). Enlarging to the Milky Way does not change this division of labor either: stellar differential rotation gives a v/c of only 7.3e-4, and the cross-galaxy simultaneity deviation is about 27 days, drowned by the 1e5-year propagation delay by 9 orders of magnitude — **what amplifies with scale is propagation delay, not relativistic effects**. The reason Earth can "pretend" to be globally consistent is that it is small enough for the light-speed term to fall below protocol overhead (in the same data center, 67 µs is a mere rounding error next to millisecond-scale queueing and fsync).
- **Engineering landing point: causal consistency is achieved by "eliminating the subject that requires arbitration", not by "propagating causal clocks".** Within a partition, a single writer issues numbers, seq is monotonic, and the cursor never rolls back — these three commitments can be validated locally without anyone's agreement, and this is precisely why they still hold at any scale. Vector clocks are not simplified away; the **subject is eliminated**: if the write right is unique then concurrent writes do not exist, and the O(N) vector with its trimming and GC disappears along with them. Aura's event plane is exactly this form (one number issuer per partition, `seq u64` monotonic), and **the key space contains no clock field at all** — keys are a concatenation of fixed-width big-endian fields, and time appears only in TTL predicates, never in a sort position.

### 2.5 The Reachability Axis: Partitioning Is the Norm

The physical ceiling has a second axis, unlike latency: not "how long until it arrives", but "whether it will arrive at all".

- **Raft is slowed on the latency axis and does no writes at all on the reachability axis.** If a majority is unreachable, writes are refused — this is where the CP choice lands: it chooses unavailability. War, navigation, and natural disasters turn this state from an anomaly into the norm.
- **The failure model changes with it.** In these scenarios the authority is not "temporarily unreachable" but **permanently gone**: the majority will never reassemble. A consensus protocol's liveness assumption is that the member set is stable and eventually reachable; permanent disappearance is an input outside its model.
- **One contrast to close it out: Raft treats partitioning as an anomaly, federation treats partitioning as the default.** The substantive problem of metadata synchronization also lies here — it pays the full complexity of consensus in exchange for **hiding the partition**: the registry looks consistent while the data is unreachable by construction; synchronizing metadata is worse than not synchronizing it, because it manufactures the appearance of "one system" (Aura repository [ADR-0013](https://github.com/orbsh/aura/blob/main/docs/adr/0013-federation-over-metadata-consensus.md)).
- **Local-first needs no switch.** The moment communication breaks, local-first takes effect automatically — it is not a configuration item of some mode, but the natural expression of the topology.

## 3. Large-Node Federation: Monolithic Dividend Internally, Peer-to-Peer Federation Externally

### 3.1 The Monolithic Vertical Dividend Is the Option Masked by Algorithmic Complexity

A modern top-spec single machine (128 cores/256 threads, TB-scale ECC memory, PCIe 5.0 NVMe at millions of IOPS) has the physical throughput to crush a Raft cluster of weak nodes busy with quorum voting. The daily peak core writes at the provincial/municipal level (medical insurance, government services, clearing) are typically a few thousand to tens of thousands per second — a heavy monolith on a single machine or in same-city active-active takes them head-on, at microsecond-level reads and writes.

Using Raft for intra-region horizontal scaling is a textbook mismatch of "trading extremely high algorithmic complexity for a meager gain". What should be done within a region is: **stack hardware + stack software efficiency, and dissolve algorithmic complexity**.

The vertical ceiling must also be stated accurately: the bottleneck is not core count but the **commit path** — WAL/fsync and memory bandwidth. The point-write throughput of a top-spec single machine is on the order of 1e5 TPS (with group commit); beyond that there are only two roads: give up per-commit durability, or batch (trading latency for throughput).

Another frequently confused question: the reason distribution is needed is overwhelmingly not throughput, but the three things vertical scaling **cannot fix** — failure domains, geography and compliance, ownership boundaries. So vertical and federation are not substitutes but a division of labor: **vertical provides capability within a domain, federation provides the boundary between domains**.

### 3.2 Inside the Territory: Two Forms

- **Extreme monolith**: a top-spec single machine + a high-performance database (multi-core-optimized PostgreSQL / an embedded RocksDB variant); persisting to local RAM/NVMe is success, at microsecond-level latency.
- **Same-city high availability**: a 3-node Raft group within a same-city <1 ms network, guarding only against data-center-level force majeure and fully shielding consensus details from the outside. This is Raft's rightful territory in [Consensus Protocol](consensus-protocol-en.md): small data volumes, where the cost of majority confirmation at same-city RTT is negligible.

### 3.3 Between Territories: Peer-to-Peer Diplomacy

Cross-region nodes have absolutely equal status: they do not share a consensus log, do not elect a global Leader, do not pursue a single view. Cross-region business (a user from Province A seeking medical reimbursement in Province B) goes through federation-standard interfaces, each side maintaining its own independent local view and reconciling afterwards. The trust boundary comes into play here — nodes mutually distrust one another, so identity is bound to the owning node and capability declarations go through a registry, rather than sharing a single permission center.

This is precisely the organizational-scale mapping of the topology judgment in [Aura Architecture](aura-architecture-en.md): stall state within a field site is written locally (not through Raft), and only metadata goes through consensus; Effector outbound registration, user namespace isolation — the same decomposition of "a small and strong control plane, a local and fast data plane".

## 4. Federated Flexible Transactions: TCC and Two Self-Healing Patches

After giving up whole-network strong consensus, cross-federation business cannot use distributed locks to lock the data on both sides — the risk of double-spending funds is borne by a business-layer two-phase flexible transaction (TCC). Note that this agrees with the criterion of [Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md): TCC deals with **coordination and compensation**, not mutual exclusion; Try/Confirm/Cancel is a state machine, not a lock.

### 4.1 The Three-Phase State Machine

Taking a cross-federation transfer of 100 yuan as an example:

```
[ Federation A (Beijing)]                                                            [ Federation B (Guangzhou)]
      |                                                                                 |
      |-- (1) Try: freeze 100 yuan locally -------------------------------------------->|
      |   (freeze the funds table; the money has not left the account but is unusable)  |-- check account validity (frozen/closed)
      |<------------------------------------------- Try-OK (B records a local hold)-----|
      |                                                                                 |
      |-- (2) Confirm: formally debit locally ----------------------------------------->|  release the hold, credit +100
      |   (asynchronously retry until confirmed)                                        |
      |                                                                                 |
      | [ Try failed/timed out ]                                                        |
      |-- (3) Cancel: unfreeze locally ------------------------------------------------>|  cancel the hold
```

- **Try**: A freezes locally (double-spending is impossible within A's view) and sends a pre-check and credit reservation to B over gRPC on mTLS/WireGuard; B verifies the payee's status, writes a local hold, and returns Try-OK.
- **Confirm**: after A receives the bilateral success confirmation it persists an irreversible ledger entry locally and notifies B asynchronously; because Try has already completed a strict pre-check, Confirm must logically succeed — network jitter requires no rollback, it retries through a local reliable message (Outbox-pattern retry queue) until B confirms.
- **Cancel**: B returns that it cannot receive the funds, or times out; A unfreezes locally, and the user's statement shows only a single "transfer did not happen" error, with no dirty ledger entry from "debit first, then reverse".

The key user-experience point lies in the design of Try: **freeze the funds rather than debit them**, so that failure leaves no trace — compensating rollback (Saga) cannot achieve this; Saga leaves behind two ledger entries, one forward and one reverse.

### 4.2 Two Self-Healing Patches

**Idempotency**: B has already credited the account but the receipt was lost in flight, so A's Outbox retry will resend the Confirm. The patch: in the Try phase A generates a globally unique XID (timestamp + node ID + sequence), and B builds locally a deduplication state-machine table with XID as the unique index — a repeated XID triggers a unique-key conflict, and B directly returns "already succeeded", never adding the money twice. This is a positive application of the criterion in [Distributed Lock Anti-Pattern §1](distributed-lock-anti-pattern-en.md): **idempotency/deduplication relies on an idempotency key + a unique constraint, not a lock**.

**Dangling re-query (TTL)**: A is destroyed wholesale before sending the Confirm, and B's hold is locked forever. The patch: B's hold table embeds a TTL (e.g. 5 minutes); after the timeout a daemon thread does not blindly cancel but queries back to A's pre-designated remote disaster-recovery node for the terminal state of that XID — if the reply is "not completed" it Cancels and releases the hold; if the reply is "already Confirmed" it forcibly advances locally to Confirm and completes the credit. **The solution to a dangling case is to re-query the terminal state, not to auto-rollback on timeout** — the latter, in the scenario where the transaction "has actually already Confirmed", makes the funds vanish into thin air.

The philosophy common to the two patches: each side relies only on **local state + a queryable declaration from the peer**, not on cross-federation real-time mutual exclusion. This is isomorphic to the core argument of [Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md) — replacing blocking (locks) with coordination (state machine + idempotency + re-query).

### 4.3 The Minimal Composition of a Cross-Domain Document

A document is not "querying the peer for its current state" but a **self-proving history** — this is its structural difference from "checking the balance". Historically, bills of lading, drafts, and cross-border clearing all relied on this one point to circulate across domains; the shipping schedule was their speed of light: the document carries its own evidence and travels itself, without requiring the peer to be online.

- **Minimal metadata**: partition + seq interval + payload + node signature (proving "this machine of mine issued numbers up to a certain position in a certain partition"). Under the single-issuer premise the seq interval has a determinate meaning and is a verifiable monotonic commitment. The timestamp inside the XID carries only uniqueness and expiry determination, not ordering.
- **No version vector**: concurrent writes do not exist under the single-writer premise, and the vector with its trimming and GC disappears along with them (see §2.4).
- **No hash chain**: a hash chain solves "preventing post-hoc tampering with history", which is another threat model. When only guarding against lag and loss, a node signature suffices; casually carrying one along merely stacks the costs of both the signature and the chain without solving any new problem.

## 5. Convergence of the Control Plane and the Data Plane

The data plane across ownership boundaries shares no storage and no consensus log (metadata is independent per node, the control plane is single-writer). Trust between federated nodes is established through **well-known protocol authentication**: identities are mutually recognized via public key/certificate, and data sovereignty is bound to the owning node — when a user logs into another node, that node has none of their historical data, and there is no global login synchronization.

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               Federated nodes (each independent)                                │
│  Local: monolithic data plane + meta instance (control plane single-writer, no consensus)       │
│  Between: well-known protocol authentication + flexible transactions (TCC/Saga reconciliation)  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

- **Internally**: business data within ownership boundaries uses the monolithic vertical dividend to dissolve algorithmic complexity; metadata is single-written by the control plane and cached by nodes, requiring no consensus — a single logical writer leaves majority voting with nothing to vote on.
- **Externally**: across ownership boundaries, TCC/Saga embraces multiple views and eventual alignment; identity trust is established via the well-known protocol.

This dividing line is an extension of the core argument of [Consensus Protocol](consensus-protocol-en.md): **Raft's territory = small capacity + strong consistency** — yet under a "single-writer control plane" metadata does not even need consensus for this small territory; Raft/Paxos is reserved for data-level strong-consistency replication (TiKV-class) and genuinely multi-writer metadata (an independent directory service). The data plane's consistency scheme is determined by Dimension A (the trust boundary) — single-machine transactions within the boundary, flexible transactions across it, CRDT in personal-sovereignty scenarios.

## 6. The Place of Local-First

The dividing line between the fifth class of models (CRDT local-first) and the first four is not scale but **the attribution of data sovereignty**: in personal-productivity and collaborative-editing scenarios, the device itself is the authoritative node and the network is merely a synchronization channel. Through the mathematical properties of the monotonic lattice, CRDT achieves out-of-order merging with no central arbitration — the price is that the data model must be expressed as a mergeable lattice (counters, sets, sequences), and arbitrary business constraints (such as a balance not going negative) cannot be placed inside a CRDT.

It is therefore not in competition with federation, but another value of Dimension A: data belonging to a personal device → CRDT; data belonging to an organization → federation; metadata belonging to the system → Raft. The three coexist in the same system by data category (e.g. collaborative documents via CRDT, user accounts via federation, the service directory via Raft).

## 7. The Protocol Layer of Liquid Organizations: the Nostr Mismatch and well-known Cold Bootstrapping

The organizational form of the AI era is liquid — super-individuals aggregate temporarily by project and disband on completion (see the federation section of [AI Era Organization and Individual](ai-era-organization-individual-en.md)). Beyond the economic layer (token settlement, voting-based pricing, AI-host coordination), there is another question that must be answered: **how do these temporary organizations actually network physically**. The protocol choice is determined by a pair of contradictions in liquid organizations:

- **Internally it must be liquid**: members come and go at high frequency, so there can be no heavy approval workflow.
- **Externally there must be a boundary**: knowledge bases, ledgers, and unpublished results cannot be thrown open like a public square.

### The Nostr Mismatch

A pure-public-relay protocol like Nostr is the solution where Dimension A takes the value "public, without sovereignty": all events circulate through public relays, and any client can pull them. This fits public socializing by nature, and by nature does not fit a bounded production organization — even if the content is encrypted, **the metadata still leaks** (who is collaborating with whom, collaboration frequency, data magnitude). Carrying a bounded organization on pure public relays is a category mismatch, not a question of encryption strength.

### well-known Cold Bootstrapping + Federated Hot Channel

The layering that holds is: **discovery goes through static distribution, collaboration goes through federated connections** — the protocol-layer projection of the layered-cost principle (cheap channels do the wide-area work, expensive channels serve only the post-convergence connections):

```
Stage 1 · Initialization  The initiator publishes /.well-known/<org>.json on its own domain:
                          a list of member public keys + endpoints, signed by the private key → the organization's first virtual boundary
Stage 2 · Discovery       Participants pull that file and synchronize the full member directory —
                          static HTTP distribution solves P2P's hardest bootstrapping problem (how to find each other)
Stage 3 · Collaboration   Members establish federated connections according to the list; high-frequency data flows within the boundary,
                          and the outside world sees only a single domain
Stage 4 · Mobility        Members joining/leaving = updating the list → each node re-pulls → connections are rebuilt
```

The essence of this layering: **using Web2's most mature static-distribution technology to provide addressing for Web3 identities (public/private keys)**. Nostr's relays have no place here — the "discovery" function the relay undertakes is replaced by a static file at near-zero cost, while the "message flow" function the relay undertakes is precisely what a liquid organization wants to reject.

### The Split Between Cold Bootstrapping and the Member List

Let us honestly mark the boundaries of this scheme: **cold bootstrapping (finding each other for the first time) and the evolution of the member list are two separate problems**.

- Cold bootstrapping is a low-frequency one-time operation, perfectly served by a static well-known file — the initiator as a single point happens to be exactly the authoritative source.
- The member list is **stateful data**: once the organization's lifetime exceeds a single session, it needs versioning (arbitration of concurrent updates), revocation (invalidating the credentials of departed members), and key rotation. A static JSON carries none of these — when the initiator goes down, the discovery mechanism dies with it, and adding a "multi-signature bulletin board" patch is essentially reinventing the relay.

So the honest conclusion is not "well-known is a perfect closed loop" but rather: **well-known handles only cold bootstrapping; the member list, as a long-term asset, belongs to some form of stateful registry** — what it looks like (single-writer, multi-signature, or lightweight consensus) is an open question, depending on the scale and lifespan of the liquid organization. The correspondence between protocol choice and organizational form: **a short-lived organization is fine with a static list; a long-lived one will inevitably grow a registry**.

### Relation to the Five Models

On Dimension A, a liquid organization is "temporarily bounded sovereignty" — lighter than federation (lifetime counted per project), heavier than personal sovereignty (with an explicit boundary and member roster). The protocol combination is correspondingly hybrid: a well-known static list defines the boundary, members trust one another in a federated manner, and collaborative editing goes through CRDT. The five models are not mutually exclusive options but a palette assembled according to data category and organizational lifespan.

## Cross-References

- **[Consensus Protocol](consensus-protocol-en.md)**: Raft mechanism details, the write-amplification account, the etcd case study, a decision matrix for when to use/not use Raft — the mechanistic basis for this article's control-plane conclusions.
- **[AI Era Organization and Individual](ai-era-organization-individual-en.md)**: the economic layer of liquid organizations (federated OPC) — token settlement, voting-based pricing, AI-host coordination; §7 of this article supplements its protocol layer.
- **[Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md)**: the source of the criterion that TCC/idempotency/re-query are coordination rather than mutual exclusion; a positive unfolding of "locks mask the absence of coordination" in federated transactions.
- **[Aura Architecture](aura-architecture-en.md)**: the engine-level landing of "a small and strong control plane, a local and fast data plane" — stall state written locally, metadata through consensus, user namespace isolation.
- **[KV Storage Engine](kv-storage-engine-en.md)**: the storage choices at the consensus and coordination levels (Fjall/SlateDB dual engine).
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: why cross-federation caching and locks are handed to the KV and consensus layers.
