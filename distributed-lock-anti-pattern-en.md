# Distributed Locks: Anti-Pattern Analysis

> **Languages:** [English](distributed-lock-anti-pattern-en.md) (primary) · [中文](distributed-lock-anti-pattern.md)

## Core Argument

In the vast majority of scenarios, a distributed lock is an **anti-pattern**. A lock amplifies "blocking" — something that should have been avoided in the first place — by several orders of magnitude: it introduces latency, clock dependency, and failure windows, yet usually only solves a problem that a lock-free approach could have solved.

To understand distributed locks, you first have to strip away a terminological illusion: **much of what you think is a "distributed lock" is not a lock at all**.

## 1. Terminological Clarification: What Is Not a Lock

| Scenario | Common Misconception | Truth |
|:--|:--|:--|
| Leader election | "Needs a distributed lock" | **lease / ownership**, not a lock (etcd/Consul) |
| Idempotency / deduplication | "Must ensure it is processed only once" | **idempotency key + unique constraint**, not a lock |
| Deduplication of expensive computation | "Prevent concurrent rebuilds" | **single-flight (request coalescing)**, not a lock |
| Cross-process exclusive write | "Must lock the resource" | DB row lock / advisory lock / optimistic concurrency, not a distributed lock |

**Counting substrate: lock / semaphore / counter / rate limiter are primitives with a common root.** They are not things standing side by side; rather, they are one **counting substrate** differentiated by purpose — doing **permission control** (mutex N=1, semaphore N, rate limiting by time window) must be **accurate**, and miscounting by one can let two machines through at the same time; doing **state statistics** is then split by requirement into **exact** (transaction volume must balance the books) and **approximate** (UV/deduplication inherently need no precision). Although rate limiting belongs to the permission-control family and could be exact (an exact token bucket), because loss is tolerable + gateways need low latency, it **chooses** a local, client-side approximation — within the same family, whether the certainty cost is worth it differs completely.

The direct corollary of this substrate: **if even "counting" often needs no precision, how much less so "mutual exclusion"?** The scenarios that genuinely require exact mutual exclusion, once the approximate members of this family are excluded, become even rarer — which is what leads to the alternatives below.

**Genuine distributed mutual exclusion is extremely rare.** Once the common misconceptions above are excluded, only the narrow gap that requires "multiple processes to perform exclusive writes on a distributed resource with no natural idempotency key" is still related to "distributed locks" — and even here, optimistic concurrency (a version token) can often sidestep it.

## 2. The Boundary of File Locks: Reasonable at the System Layer, Foolish at the Platform Layer

The flip side of the judgment that "distributed locks are an anti-pattern": **when is a "lock" actually reasonable?** The dividing line lies in the **medium** and the **deployment shape**.

### System Layer: File-System-Based Coordination Is Reasonable

In **system-layer** interactions (such as the package manager `package-lock.json`, PID files, `/var/run` lock files), file-system-based coordination is **reasonable** — because the file system itself is a **persistent shared medium that exists across processes and across restarts**, and system-level coordination relying on it is only natural.

> Note: a true `flock`/`fcntl` advisory lock is bound by the kernel to the process / the opened fd, and is released as soon as the process exits; what is meant here is a mechanism that expresses state through **file existence** and can continue to rely on the same medium after a restart.

### Programming Platform Layer (Single Replica): An In-Process Mutex Suffices, File Locks Are Foolish

At the **programming platform layer**, in the **single-replica + platform-guaranteed restart** scenario (e.g. a single K8s Pod brought up by the orchestration layer), the application itself **need not handle crashes** — an in-process mutex is enough, and to then apply file-lock/process-lock-style thinking is **foolish**. This habit of low-effort coordination is most conspicuous in the PHP ecosystem (the representative of the shared-hosting situation: multiple processes sharing a block file system, with no resident coordination service).

### Truly Distributed Multiple Replicas: Consensus/Lease Is Needed, Not a Lock

**Truly distributed multiple replicas** are another matter — exclusive writes over a resource shared by several active replicas still have to be guaranteed **across nodes**; it is just that the correct method is **consensus/lease**, not a Redis lock. See §7.

## 3. Anti-Pattern Case: Misuse of Review Locking

**A common anti-pattern**: locking items down during list review, forbidding new additions.

```text
❌ lock(list) during review → add is forbidden while the review runs
   - Blocks the business unnecessarily (additions get stuck)
   - Turns "review", a read-only/snapshot operation, into a global exclusive lock
```

**Correct: the watermark pattern**. The review looks only at the current snapshot; additions made during the review are filtered by the review time point and carried into subsequent batches:

```text
✅ review watermark = T0
   - Review items before T0
   - Items added after T0 → filter by entry time > T0, move into the next review round
   - No lock: additions never block
```

In a real-time architecture like Flux, you can even add items directly inside the review view, but that approach disturbs the current review focus — so the watermark is still the better answer. **Either way, locking down directly is never the optimal strategy.**

## 4. Distributed Transactions Need Lock-Freedom Even More

Distributed transactions have several protocols (two-phase commit, Saga, TCC, local message table / transactional Outbox, event sourcing); what they deal with is **consistency**, not mutual exclusion. These patterns themselves focus on coordination and compensation and **have no use for distributed locks**.

Squeezing a distributed lock into a distributed transaction is using "blocking" to paper over missing coordination — the direction is simply wrong.

## 5. Already So on a Single Machine; Distribution Is an Amplifier

Even in single-machine/single-process programming, **most scenarios should avoid locks** (prefer lock-free data structures, optimistic concurrency, message passing). Distributed locks **amplify** this inefficiency **by several orders of magnitude**:

- Single-machine lock: nanosecond-scale blocking, local, no network.
- Distributed lock: a single acquisition involves network RTT + consensus round trips, at the microsecond-to-millisecond scale, and you still have to handle clock synchronization, GC pauses, network partitions, and lock expiry.

**It introduces far more problems than it solves.**

## 6. Where Redis/Redlock Stands Here

Redis is often defended as "suited to distributed locks". That defense collapses on two levels:

1. **The right answer for a lock should have been consensus/lease** (etcd, Consul) or single-writer metadata (aura's framing), which are naturally strongly consistent.
2. Redlock does not even qualify as that "suboptimal answer" — it does not satisfy the correctness guarantee of strict mutual exclusion. See [Redis: the "Network RAM" Trap §6.1](redis-critique-en.md) for details.

## 7. Where "Consensus" Rather Than a "Lock" Is Truly Needed

When you genuinely need "strongly consistent exclusive writes by multiple processes on a distributed resource", the right tool is the **lease / leadership provided by the consensus layer**, not a distributed lock. Consensus guarantees consistency, the lease provides ownership, and the two combined naturally handle node failure and transfer of trust — exactly the job of the "Consensus and Coordination Layer", see [KV Storage Engine](kv-storage-engine-en.md).

## Cross-References

- [Redis: the "Network RAM" Trap §6.1 Distributed Locks / Redlock Debate](redis-critique-en.md)
- [Consensus Protocol](consensus-protocol-en.md)
- [KV Storage Engine — Consensus and Coordination Layer](kv-storage-engine-en.md)
- [Distributed Collaboration Topology](distributed-collaboration-topology-en.md) — TCC/idempotency/query-back unfolded in federated transactions (a positive case of coordination replacing mutual exclusion)