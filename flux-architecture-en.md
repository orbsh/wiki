# Flux Architecture: The Core of Reactivity Is Data Flow

> **Languages:** [English](flux-architecture-en.md) (primary) · [中文](flux-architecture.md)

**Core thesis**: Traditional web architecture is a request-driven passive mode—browser refresh, polling, pull from origin. Flux architecture inverts this: when data changes, it is actively pushed to the edge and to clients, and user requests only hit data that is already ready. Stream computing and message queues are not auxiliary components but core infrastructure. "Flux" means flow—from production to consumption, data travels through a continuous pipeline, with no intermediate stalls and no pulling back.

> Flux architecture is a design at the communication-pattern layer. Fused with [Aura Architecture](aura-architecture-en.md) (a storage-compute-integrated engine), it forms [Fluxora Architecture](projects/fluxora-architecture-en.md)—a complete event-driven system.

## The Cost of Passive Mode

The default interaction of traditional architecture is the **browser refresh**: the user clicks refresh → HTTP request → origin query → response. This brings three problems:

1. **Repeated requests**: data is pulled repeatedly even when unchanged, wasting bandwidth and origin resources
2. **Uncontrollable latency**: every request traverses the full path (DNS → TCP → TLS → origin → database), and the latency the user perceives is the sum of that path
3. **Complexity of cache invalidation**: to mitigate repeated requests, Redis is introduced as an intermediate cache, which in turn brings a chain of problems—TTL management, cache stampede, avalanche degradation

Redis caching is essentially a patch for passive mode—it does not change the "request-driven" essence, it merely adds a layer of in-memory buffering on the pull-from-origin path. The real solution is to change the mode itself.

## Three-Layer Push Strategy

Flux architecture is layered by data characteristics, each layer using a different push mechanism:

### Static Layer: index.json Resource Tree → CDN

Data that can be made static (product catalogs, configuration, documents) is precomputed by stream computing into a JSON resource tree and pushed to the CDN edge. The client uses a version-number interface to decide whether it needs to pull a new version—`if (remote_version > local_version) { download_from_cdn(); }`.

This is consistent with the thinking of modern Lakehouse architecture: Lakehouse manages **data versions and snapshots** through a Catalog layer (a metadata service), rather than relying on TTL caching at the storage layer. The index version-number interface is a minimal implementation of the same idea—elevating version governance from the storage layer to the control plane, replacing an entire cache-invalidation mechanism with a single interface. See [Lakehouse Research](lakehouse-research-en.md).

Another similar example is NixOS: an immutable index (such as a NixOS derivation) describes the complete dependency tree, and the client pulls on demand. index.json is not queried on every request; it is pulled only after the version number changes—greatly reducing unnecessary traffic.

**Cache invalidation**: the CDN does no active invalidation (it cannot notify edge nodes worldwide). The solution is minimal—one interface returns the current version number; the version number lives in shared memory, read in nanoseconds, and written by direct assignment on the stream-computing side. The client decides for itself whether an update is needed, and the CDN only hosts static files. No Redis TTL management, no cache-stampede protection, no avalanche degradation—highly dynamic data has no need for caching in the first place, and the complexity that caching introduces far outweighs the benefit.

The version number is two-tiered: every resource referenced by the index JSON (product details, images, prices) carries a version number, and the index itself carries a version number—the client first compares the index version, then compares resource versions on demand, so that only the parts that actually changed are pulled. The index is large, and placing it at the CDN edge can significantly speed things up; the CDN only hosts static files, and all version checking is done on the client.

```
Stream computing generates the index JSON
  → every resource reference carries a version number
    → the index itself carries a version number
      → the index is pushed to the CDN (a large file; the CDN only hosts static files)
        → the client calls the interface to get the current version number
          → if the version does not match, pull the new index from the CDN
```

**Streaming precomputation: the production side of index.json**. The above is the consumption side of passive pulling, but index.json itself is not maintained by hand—it is **actively pushed** to the edge by a stream-computing engine that continuously watches database changes—and that is the real precondition for not relying on TTL-triggered pull-from-origin:

1. The raw data in the database changes (for example, a product price update).
2. The stream engine (RisingWave/Flink) detects the change and aggregates/transforms it into a JSON resource tree.
3. The precomputed JSON is pushed to CDN edge nodes (Cloudflare Workers, Fastly) via an API.
4. Users get the latest data from the CDN and never hit the origin.

The key difference: **active push vs. passive pull**. Traditional solutions rely on pulling from origin after the TTL expires, which is still essentially the "request-driven" passive mode, and the time at which edge nodes get their data depends on the length of the TTL; a stream engine pushing means the edge gets the new version immediately after the data changes. The cost is that stream computing runs continuously (it watches CDC even when nothing changes); the payoff is **predictable latency** (constant at the CDN edge) rather than variable latency (Redis + origin), and the complete elimination of Redis for static data.

**Trade-off**: this production-side pipeline requires architectural refactoring—the traditional "application server + Redis cache" model must be redesigned as "stream pipeline + CDN edge". For legacy systems this is a migration of several months; for new projects it is an architecture that is correct from day one. For the selection and role of stream-computing engines (from auxiliary component to core infrastructure), see "Stream Computing and Message Queues: From Auxiliary to Core" below.

### Dynamic Layer: WebSocket/WebTransport Active Push

Data that cannot be made static (real-time inventory, user state, collaborative editing) goes over a WebSocket/WebTransport channel, pushed actively to the client when the data changes. The client does not need to refresh; the data arrives on its own.

**The real difference from HTTP**: modern HTTP uses keep-alive by default, so at the connection-reuse level there is not much difference from WS. But WS's core advantage is its **connection semantics**—HTTP is request-response (initiated by the client), whereas WS is a bidirectional stream (the server can push at any time). HTTP without keep-alive has to handshake every time, which is more expensive; even with keep-alive enabled, every HTTP/1.1 request still sends the full protocol headers—frozen cookies, each header field transmitted over and over; the header overhead is not small and stacks up implicitly with each request (the browser carries it automatically, and the user is unaware); WS, after a single handshake, reuses the connection continuously, and subsequent frames carry only a tiny frame header, so the actual overhead is far smaller.

In extreme cases (say, no refresh for a whole day), passive mode seems to have "no cost if you do not request", but the complexity of modern browsers (ad blockers, background requests, service workers) makes "no requests" an ideal assumption. A WS channel, by contrast, always has one controllable connection with predictable traffic.

**HTTP's success is a distortion of the mental model**: the very reason HTTP monopolizes is that it did **not** exist for performance—what it optimizes is the **programming experience and the consumption scenario** (request-response, stateless, short connection); easy for developers to write, easy for browsers to consume, so it won by being "easiest to develop" rather than "fastest". By contrast, persistent-connection protocols (WS and the like) are designed for **specialized scenarios and optimal performance**. This is a counterintuitive conclusion: the protocol that is "most special and least efficient" in that domain ended up occupying the largest market because it catered to most people's programming mental model. The consequence is that HTTP's enormous success **distorted the default mental model of the whole field**—people take "request-response / fetch everything at once" as a given, and even when the underlying layer has long since switched to WS/WebTransport, they habitually continue HTTP's "closure-ization" (transmitting all the data in one go, even at great waste), while overlooking that once a WS connection is established it can carry an **atomized instruction sequence** (each incremental operation, at nearly zero extra cost). The natural use of WS is "a continuous, atomized stream of incremental instructions", not "transmitting a big snapshot"—the latter is a residue of HTTP thinking.

**The cost of WS**: on weak networks, WS long connections are unstable—dropping when switching networks, being disconnected by timeouts on intermediary devices, and connection-state management adding memory overhead on the server. WebTransport solves these problems—based on QUIC, it natively supports connection migration (no drop when the IP changes), multiplexing (one connection carrying multiple streams), and 0-RTT recovery. On mobile and in high-latency scenarios, WebTransport is the evolution direction for WS.

**An honest assessment of internal latency**: the push chain of Flux architecture is `CDC event → stream computing → message queue → WS gateway → client`. Stream computing and message queues both add internal latency, which may be greater than a direct HTTP request. But the key difference is in the **last hop**—WS → client is an active server push, which feels fast to the user (data arrives immediately after it changes, without waiting for the user to initiate a request). Internal latency, like a microservice call chain, is the overhead of architectural cohesion; the real advantage is on the client side, in how it feels. What the message queue's latency buys is natural decoupling—producer and consumer need not be aware of each other.

### Hybrid Strategy

Real systems are usually hybrid: what can be made static goes through index.json + CDN (low-frequency changes, acceptable staleness), what is dynamic goes through WS push (high-frequency changes, requires real-time). A hybrid strategy brings greater freedom—there is no need to make all data static (too-fast version rotation wastes CDN traffic), nor to put all data through WS (low-frequency data is not worth maintaining a long connection).

The key criterion: **data change frequency × number of clients**. High frequency × many clients = WS push; low frequency × many clients = CDN static; high frequency × single client = in-process cache; low frequency × single client = direct read, no cache needed.

## Stream Computing and Message Queues: From Auxiliary to Core

In traditional architecture, stream computing (Flink/RisingWave) and message queues (Kafka/NATS) are optional auxiliary components—used for real-time data warehousing or asynchronous tasks. In Flux architecture they become **core infrastructure**:

- **Stream-computing engine** (primarily [RisingWave](https://risingwave.dev)): watches database changes, precomputes aggregates, pushes to the CDN (static layer) or through the message queue to the WS gateway (dynamic layer). It decides what data should be pushed, when to push it, and in what shape. Note: stream computing always incurs running costs (even when not pushing, watching CDC consumes resources); this is the inherent cost of Flux architecture—passive mode costs nothing when the user does not request, while Flux architecture trades continuous stream computing for an instant response when the user does request.
- **Message queue** (Kafka/NATS/Iggy): the pipeline connecting the stream-computing engine and the WS gateway. Stream computing emits change events, the message queue guarantees in-order delivery to the gateway, and the gateway pushes onward to connected clients. Slightly higher latency, but it buys natural decoupling—producer and consumer need not be aware of each other.

Remove stream computing and message queues, and Flux architecture degrades to traditional passive mode—without an event source there is no trigger for a push.

### Stream-Computing Engine Selection

Selection is not a single choice—different engines serve different business lines, and they may coexist.

**RisingWave (primary choice)**: a streaming database, storage and compute in one. Materialized views are queryable tables; `SELECT * FROM mv_xxx` lets you see intermediate results directly, and the debugging experience is the same as with an ordinary database. The operations chain is short: the Flink approach needs three components—"Flink compute + external storage for intermediate state + Kafka as the pipeline"—while the RW approach needs only two: "RW compute+storage + message queue as the delivery pipeline".

**Flink (heavyweight supplement)**: a pure compute engine—its intermediate state is opaque (TaskManager memory or RocksDB), not stored and not queryable, making debugging almost a black box. But Flink's ecosystem maturity and ultra-large-scale processing capability are something RW has not yet reached. Even if you choose lighter options like Aura/Arroyo/Fluvio, when a business line expands to cross-system aggregate analytics, global reporting, and ultra-large-scale real-time data warehousing, Flink may still be brought in as a supplement. This is a different business line—not a replacement in selection, but tiering by scale.

**Fluvio (Rust-native lightweight option)**: a pure Rust stream-processing platform, lighter, suited to embedded scenarios sensitive to binary size. Consistent with the technical aesthetic of Aura architecture (Rust + in-process), but its ecosystem and maturity are not on par with RW.

**Arroyo (Rust-native stream computing)**: written in pure Rust, supports Rust UDFs, serverless architecture. Its UDF capability aligns with Aura's polyglot philosophy. From the perspective of traditional architecture, Arroyo's weakness is insufficient connectors (no native Postgres CDC, requiring Debezium; immature MySQL support; no Pulsar support). But on Aura/Fluxora architecture these weaknesses do not exist—the event source is an in-process Fjall write (no external database CDC connector needed), and the message queue is Kafka (no Pulsar needed). Under this architecture, Arroyo is a direct replacement candidate for RW.

### Degree of Integration → Simplification at the Business Layer

Once RW integrates storage, what it brings is not just operational simplification but **simplification of the business-layer architecture**. The business layer of a traditional architecture does four things: query the database + query the Redis cache + manage cache invalidation + serialize/deserialize. Once RW integrates storage, the business layer subscribes directly to materialized view changes and needs no intermediate cache layer—cache invalidation, cache stampede, and avalanche simply do not exist at the root.

This route has two precedents:
- **[Fluxora Architecture](projects/fluxora-architecture-en.md)**: the UI subscribes directly to the event stream; no API URL, no HTTP verbs. The business layer degrades into a Kafka consumer that only handles logic and does no data shuttling.
- **Meteor's DDP protocol**: the client subscribes directly to changes in a database query. RW's materialized views + sink are the same idea—what the client subscribes to is "the change stream of a query result", not "the response to a query request".

When integration is high enough, the business layer can even **reach straight down to the database layer** the way Meteor does—RW is itself a database, and materialized view changes serve directly as the push source, with no intermediate business service translating. This contrasts sharply with the traditional three-tier architecture (web container → business layer → database), in which "the business layer shuttles data".

**The dialectic of cost**: the continuous running cost of stream computing seems higher than that of passive mode, but it eliminates Redis, a far more expensive cache layer—Redis's memory cost is an order of magnitude higher than stream computing's CPU cost (memory at $16.56/GB/month vs. CPU billed by core-hour). Overall it comes out ahead. More importantly, **once the business grows complex enough, stream computing is inevitable**—not a question of "whether", but of "adopting it proactively or being forced into it". If you do not use stream computing for precomputation and change detection, you have to hand-write cache-invalidation logic, scheduled tasks, and cross-service state synchronization in the application layer—reimplementing in more expensive, less reliable ways what stream computing is meant to do, and with none of the fault tolerance, replay, and exactly-once semantics of a stream-computing engine.

**Response speed and architectural reliability**: Flux architecture crushes passive mode on these two dimensions. Response speed—millisecond-level push to the client after a data change, without waiting for the user to initiate a request. Architectural reliability—the stream-computing engine has checkpoint/replay mechanisms, the message queue has persistence and at-least-once delivery, and the entire chain from CDC to client has fault-tolerance guarantees. Passive mode's Redis cache is a single point of failure, and cache stampede and avalanche are standing risks.

## The Alternative Path: Aura's In-Process Event Processing

The architecture above centers on RisingWave—a stream-computing engine that watches CDC, maintains materialized views, and emits change events. But there is a more radical path: do not rely on an external stream-processing engine, and instead build the event-processing logic directly on top of an embedded store (Fjall).

[Aura Architecture](aura-architecture-en.md) takes this path. Core ideas:

- **Distributed storage is the event source**: Fjall is itself an in-process, strongly durable store (WAL); every write is an event, and the WAL is the event stream, with no local consensus overhead. There is no need for an external CDC pipeline watching another database's changes.
- **Embedded scripts are the event handlers**: the framework has built-in runtimes for multiple languages, and each booth picks one scripting language. There is no hard restriction on what a language is used for; the following are only suggestions: Steel Lisp suits deterministic policy logic (permissions, routing, state transitions), Python (PyO3) suits business logic, and WASM suits heavy plugins that bring in other languages (sandbox isolation). Language integration is likewise not hard-restricted; any language that can be integrated via PyO3 or WASM works. Making RPC calls to external services is neither built in nor prevented—users can make them themselves on the Python side with libraries such as httpx, but the framework provides no infrastructure for it. See [Aura Architecture](aura-architecture-en.md).
- **Zero network hops**: the RisingWave route's chain is `PostgreSQL CDC → RW compute → message queue → WS gateway → client` (four hops). The Aura route is `Fjall write → Steel Lisp processing → WS gateway → client` (three hops), and the first three hops are all in-process, with no network overhead.

**Division of labor with the RW route**:

| Dimension | RisingWave route | Aura route |
|:--|:--|:--|
| **Event source** | External database CDC | A Fjall write is the event (built in) |
| **Event processing** | RW materialized views (SQL, declarative) | Multi-language layering (Lisp policy + Python business + WASM plugins) |
| **Intermediate state** | RW storage (queryable) | In the booth process (0 network) |
| **Network hops** | Four hops (including two cross-process) | Three hops (only the last crosses the network) |
| **Applicable scale** | Large scale (multiple data sources, cross-team) | Within a single system (booths are autonomous) |
| **Expressiveness** | SQL (declarative, limited but auditable) | Lisp (Turing-complete, flexible but requiring constraints) |

**Constraints of the Aura route**: the event-processing logic runs inside a Steel Lisp sandbox and has high determinism requirements—the same input must produce the same output (similar to the determinism constraints of a smart contract). This restricts non-deterministic operations such as network calls, random numbers, and timestamps inside a script. The RW route has no such constraint, because SQL materialized views are themselves declarative.

**Design principle: all events, no commands**: what is persisted and replayed in the architecture is only events (facts that have happened), never commands (operations requested). Commands cannot be safely replayed—replaying "transfer 100 yuan" three times is a disaster; events can be safely replayed—replaying "account balance becomes X" a hundred times gives the same result. The Raft consensus log works exactly this way: what is stored in the log is committed entries (facts confirmed by a majority), not "please execute" commands. This is consistent with the Actor model's let-it-crash philosophy—after a booth crashes, its state is restored by replaying from the persisted event log, unfinished operations are retried by the client on timeout, and the server recovers from the last consistent state. There is no need to persist commands, and no need for the rollback machinery of distributed transactions.

**Possible hybrid**: the two routes are not mutually exclusive. Aura handles high-frequency in-process state events (shopping carts, sessions, real-time inventory), and RW handles cross-system aggregate analytics (global reporting, cross-service materialized views). The WAL produced by Aura can also serve as an RW CDC source—Aura takes the fast path, RW the slow path.

## Fluxora: The Fusion of Flux + Aura

[Fluxora Architecture](projects/fluxora-architecture-en.md) is the product of fusing Flux architecture with the Aura engine:

- **Flux layer (communication and UI)**: a built-in UI component system with JSON assembly; components bind to event names, with no API URL and no HTTP verbs. The UI is not a "request-render" loop but a "subscribe-receive-render" event stream. WebSocket is the only communication channel, and all data changes are pushed to the client over WS. Static assets (images, fonts, scripts) are referenced directly by the UI and are outside Fluxora's scope of management.
- **Aura layer (storage-compute engine)**: booth runtime + Fjall storage + a standalone metadata instance (single writer, no consensus), completing state management and event processing in-process. See [Aura Architecture](aura-architecture-en.md).

Fluxora rejects the HTTP request-response model from the outset. It is not a WS channel added onto a traditional architecture; rather, WS is the only channel, and everything from the protocol layer to the UI layer is fully event-driven.

**Fluxora vs. a dumb gateway**: traditional WS push solutions (such as nchan) are dumb gateways—they only relay messages and have no custom logic. The problem: a message queue cannot be attached directly to a push gateway; an extra layer of consumers is needed in between to do distribution logic, which in turn cancels out the reliability guarantees of the message queue. Fluxora's gateway is that layer of consumer—it implements message distribution, channel routing, and event-type parsing at the gateway layer. On Aura architecture, this path can be more radical: a data change triggers Steel Lisp processing directly from a Fjall write and is pushed to the client through the WS gateway, similar to Meteor's DDP protocol—database straight through to UI, with no separate business-service layer in between. The precondition is that Aura's in-process event processing is powerful enough to take on the role of the Fluxora gateway.

### Client Consumption Side: Incremental Instruction Stream + On-Demand Dictionary

Server-side push solves "how it arrives"; client-side consumption solves "how state is maintained after arrival". What follows describes Fluxora's frontend paradigm for consuming WS incremental instructions—validated in practice by mudra's multi-panel real-time sync (`docs/WS-SYNC.md`).

**Commands are increments; the frontend replays them**. What the server broadcasts is not a snapshot but a **command sequence** carrying database primary keys (such as `['tag_add', 2, 3]`—"add tag 2 to page 3"). Each frontend treats the commands as increments of some id→object state machine, applying them in place to local state by id rather than redrawing the whole thing. The semantics of the event is doing, not "here is the data again, take a look".

**On-demand dictionary (id-indexed)**. The client maintains an `{id: payload}` dictionary, filled on demand: rendering components receive only an id and fetch the data from the dictionary; when the dictionary lacks that id, a read command is issued (single or batched `IN`) to fetch and fill it in. Tags only grow and change rarely, so they can be cached in browser localStorage.

**Render walks only the upward chain; siblings are lazy**. To render this node, you only need to walk up the `parent_id` chain to fetch ancestors (a linked-list traversal, O(depth) and very shallow, no caching needed). Sibling information (all children under some parent) is not attached wholesale to the dictionary—it is not needed most of the time—it is **requested lazily** once, only for the interaction of expanding a menu, and cached in a session-level Map (`parent_id → children`), so that clicking again in the same session hits directly. Nodes that are never used always cost zero transmission.

**Initialize by pulling only what is needed**. Request the page list (including the ids it references), take the **union** of all referenced ids and pull them in one batch; the union naturally contains every ancestor of every reference, and the backend brings out the ancestor chains in one pass with a single recursive CTE (`WITH RECURSIVE`), so the dictionary is complete in one go. A full forest (the tag tree) is not pulled—only the levels that are actually used.

**Key point**: Fluxora's consumption side rejects "fetch a full snapshot at once and then refresh on demand" (a residue of HTTP thinking), replacing it with **command increments + an on-demand dictionary**—transmitting only the data that is used, only when it is used, with the cost of a hit approaching zero and unused data never arriving.

**The same principles apply to transmitting the UI itself**: what is described here are general data-flow principles; [Flex UI](flex-ui-en.md) applies the same set of principles to **transmitting the UI structure** (the component tree as an incremental instruction stream over a recursive struct). It is a framework-level form rather than the implementation of a particular engine. The two share a common root—"command increments + on-demand dictionary" is the general paradigm for "transmitting any recursive struct", and business data and the UI tree are merely two kinds of payload.

## From MQ to Aura: Architectural Evolution

### MQ as a Transitional Solution

Fluxora's early design used Kafka as the internal communication pipeline: multiple business services acted as Kafka consumers, coordinating event flow through the MQ. This was functionally workable, but it exposed structural costs.

### The Structural Costs of MQ

**Unstable latency**: Kafka's end-to-end latency is affected by the number of partitions, consumer-group rebalances, disk I/O, and other factors, and it fluctuates widely. For scenarios that need deterministic latency (such as real-time push), this is unacceptable.

**Heavyweight, hard to control**: a Kafka cluster itself needs ZooKeeper (or KRaft), multiple brokers, partition management, and replica synchronization. Operational complexity grows linearly with scale. During partition rebalancing, consumers pause consumption, causing latency spikes.

**Microservice fragmentation**: the whole thing is a microservice model—many services sit on top of the MQ, each deployed and operated independently. This brings all the downsides of microservices (accumulated network RTT, distributed transactions, operational bloat) without the benefits (independent scaling, technological heterogeneity)—because all the services are essentially doing the same thing: consume MQ events, process business logic, write state back.

**Business-logic fragmentation**: interacting through events scatters business logic across multiple consumer services. A complete business process is split into several event handlers, and debugging requires tracing across services. It gets none of microservices' "independent deployment" advantage, yet bears the cost of "scattered logic".

**Unrestricted languages lead to fragmentation**: on the surface "no language restriction" is freedom, but in practice the commonly used languages are just a few (Python, Go, Java). Not restricting actually leads to multiple languages mixed within one system, increasing coordination cost and cognitive burden.

### Aura's Solution

Aura eliminates the above problems at the architectural level:

**Serverless, one platform**: upload scripts directly (Steel/Python/WASM); the Aura runtime automatically manages booth lifecycles, state persistence, and scale-to-zero. No need to deploy a Kafka cluster, no need to deploy a pile of microservices. Operations converge onto a single platform.

**Integration of fragmented scripts**: Aura's Actor model consolidates fragmented event handlers into organized booth instances. Each booth has its own state partition (a Fjall partition) and communicates loosely coupled through field events (emit/on). Control granularity is finer than with an MQ—booth activation, eviction, and state migration all fall within the runtime's scope of control.

**Language restriction**: three languages, Steel / Python / WASM, covering three dimensions: deterministic policy (Lisp), business logic (Python), and sandbox isolation (WASM). If that is not enough, they can be extended, but general-purpose languages such as Java/Go are refused outright—that they can compile to WASM is another matter, but native integration would introduce runtime overhead and fragmentation. Restricting languages = reducing cognitive complexity = increasing reliability.

## Architecture Comparison

| Dimension | Passive mode (traditional) | Flux architecture |
|:--|:--|:--|
| **How data arrives** | Client pull | Server push |
| **Traffic when nothing changes** | Repeated requests (wasteful) | Zero push traffic (but stream computing runs continuously) |
| **Latency** | Sum of the full path | Internal path is slightly longer but feels fast (the last hop arrives instantly) |
| **Cache layer** | Redis intermediate cache (a patch) | CDN + WS (native) |
| **Invalidation mechanism** | TTL + manual purge (complex) | Version-number interface + event-driven (simple) |
| **Role of stream computing** | Optional auxiliary | Core infrastructure |
| **Role of the message queue** | Optional auxiliary | Core infrastructure → in-process in Aura (evolution direction) |
| **Client experience** | Needs refresh/polling | Data arrives on its own |
| **Reliability** | Redis single point of failure, risk of cache stampede/avalanche | Fault tolerance along the whole chain (checkpoint/replay/persistence) |
| **Adaptability to complex business** | Hand-written cache invalidation / scheduled tasks / state sync | Native support from stream computing, complexity internalized in the engine layer |
| **Degree of integration** | Web container + business layer + Redis + database (four layers) | RW (compute+storage) + message queue + WS gateway (two layers) → a single Aura platform |
| **Debugging** | Redis cache hits/misses hard to trace | RW materialized views can SELECT intermediate results directly |
| **Role of the business layer** | Data shuttling (query DB + query cache + manage invalidation + serialize) | Logic processing (subscribe to materialized view changes, no cache layer) |
| **Deployment model** | Microservices + MQ + cache + database | A single platform (Aura) |
| **Language strategy** | Unrestricted (fragmentation) | Steel / Python / WASM (controlled) |

## Relationship to the Redis Critique

This document is a full treatment split out from §4 of the [Redis: the "Network RAM" Trap](redis-critique-en.md). The Redis critique answers "why not use Redis"; this document answers "if not Redis, then what"—Flux architecture is a structural replacement for Redis, not a patch.

---

## Cross-References

- **[Aura Architecture](aura-architecture-en.md)**: storage-compute-integrated engine, the compute and storage layer of Flux architecture. Together = Fluxora.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: a complete critique of the passive-mode patch. Flux architecture replaces Redis at the pattern level.
- **[Long-Lived Connection Engineering](long-lived-connection-engineering-en.md)**: the physical constraints of WS/WT long connections when Flux architecture is deployed (backpressure, fragmentation, connection skew) and their solutions.
- **[mudra implementation](https://github.com/orbsh/mudra/blob/main/docs/WS-SYNC.md)**: the concrete implementation of multi-panel real-time sync—command increments, on-demand dictionary, lazy siblings—with the landing code and verification.
- **[Lakehouse Research](lakehouse-research-en.md)**: the index version-number interface shares a common root with the Lakehouse Catalog—elevating version governance from the storage layer to the control plane.
