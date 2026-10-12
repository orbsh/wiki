# Aura: A Modern Distributed Booth Engine with Storage-Compute Integration

> **Languages:** [English](aura-architecture-en.md) (primary) · [中文](aura-architecture.md)

**Status**: Architecture design complete
**Date**: 2026-06-20
**Core philosophy**: Framework fusion, compute sinking, zero-copy, deterministic boundaries

> Aura is a modern distributed booth engine with storage-compute integration. In Aura there is no SQL, no external cache locks, no multi-database synchronization. The database is completely invisible, and state is managed automatically by the booth. You only modify the context in memory; the Aura Rust runtime automatically handles high concurrency and multi-machine backup.

## 0. Architectural Doctrine: Foundation Framework Fusion

A clear main line in current engineering practice: **gateways, service orchestration, web frameworks, storage, message queues—the foundation frameworks—are converging**. Rivet makes the booth runtime and storage co-resident on the same machine; Windmill fuses task orchestration with the FaaS execution layer; Dapr attempts to unify inter-service communication and state management with a Sidecar—although it is overly fragmented, the aspiration is the same. Aura is an engineering practice of this trend: compiling all five of the above capabilities into a single Rust process.

### The Motivation for Moving Away from Microservices

The microservices architecture solved the scalability and team-collaboration problems of the monolith, but it introduced a new set of systemic costs:

| Problem | Root cause |
|------|------|
| **Network RTT accumulation** | Each inter-service call has 1–3ms of latency; the N+1 cross-service queries in a complex business chain (user → order → product → inventory) amplify latency linearly |
| **Distributed transactions** | Cross-service data consistency requires compensation mechanisms such as Saga / TCC, which are complex to implement and hard to debug |
| **Operational sprawl** | Service discovery, load balancing, circuit breaking and degradation, distributed tracing—each requires its own infrastructure (K8s + Istio + Jaeger + ...) |
| **Data movement** | Services transmit serialized data (JSON/Protobuf) over RPC; the application side repeatedly deserializes and reassembles it, with CPU and memory overhead far exceeding the business logic itself |
| **Sidecar overhead** | Service meshes (Dapr, Istio) attach a proxy layer to each service as a Sidecar; the aspiration is reasonable but too fragmented, and the operational cost is out of proportion to the latency benefit |

These problems are not implementation defects of microservices but structural costs of distributed architecture—as long as inter-service communication goes over the network, the above costs are unavoidable. The essence of moving away from microservices is: **replace network communication with in-process communication, while retaining the decoupling capability of microservices**.

### Scaling Unit: Address vs Program (A Methodological Framework)

Moving away from microservices rises to the abstraction level of "how a scalable application organizes its extension points", and reduces to a single judgment: **is the scaling unit an "address" or a "program"**.

- **Address (RPC / out-of-process)**: what is configured at the core is the **interface endpoint** (URL / socket / protocol), and at runtime it calls somewhere remote. A Webhook is merely its remote body; a local socket / stdio is its near-end body. It is cross-process, each side in its own language, loosely coupled, but every call pays a network/process round trip. If it is asynchronous and the request/response do not correspond one-to-one (such as event subscription pub/sub), then it lacks a **deterministic contract**—it is merely the asynchronous organizational form of RPC, cannot carry the responsibility of an overall extension point, and is not an independent paradigm.
- **Program (embedded dynamic code)**: what is configured at the core is **logic**, executed in place within the process. It splits into a spectrum by "expressiveness × loading":
  - **General-purpose scripting**: fully expressive languages such as Steel / Python (PyO3);
  - **Restricted DSL / configuration-as-program**: the core = the interpreter of that restricted language, the configuration = source code in that language (rule tables, routing policies);
  - **Compiled artifacts**: so/dll → **WASM** (cross-platform + sandbox; so/dll is neither cross-platform nor isolated).

**The convergence of coupling and safety**: RPC is loose (extensions outside the protocol, crash-isolated), embedding is tight (in-process); but **WASM uses a sandbox to reclaim the cost of "tightness"**—the higher tier of embedding instead achieves both light coupling and safety. This is another reason WASM replaces so/dll.

**Aura's choice is precisely this verdict**: it opposes "address / network RPC" (the motivation for moving away from microservices in the previous section), adopts "program / in-process multilingual embedding" (Steel / Python / Wasm, zero IPC, single binary), and uses the WASM sandbox to take on "untrusted third-party code". For the language-level deepening of the embedded spectrum, see [Embedded Script Language Selection](embedded-script-languages-en.md).

### Contemporary Solutions to the Monolith's Original Problems

The monolith was not eliminated because "monolith" itself was wrong, but because the engineering means of the time could not solve three problems inside the monolith. These three problems have mature solutions today:

| The monolith's original problem | Contemporary solution | Correspondence in Aura |
|-------------|---------|--------------|
| **Language lock-in**: the whole system is forced to use a single language and cannot pick the optimal tool for different scenarios | Embedded multilingual runtime (in-process VM, zero IPC) | Steel / Python / Wasm—three languages embedded via PyO3 and Wasmtime; booths choose as needed |
| **Team coupling**: a change in one place may affect the whole, and multiple teams developing in parallel conflict frequently | Decoupling by event contracts (declarative interfaces, validated at compile time / load time) | `interface_schema()` declares receives/emits; the Realm enforces whitelist validation |
| **No independent scaling**: the whole is deployed together, and a hot module cannot be scaled separately | Virtual Actor activation on demand, automatic Scale-to-Zero when idle | instance key routing + Fjall disk hibernation + Tokio event loop |

These three solutions happen to correspond to the three pain points that microservices originally tried to solve. The difference is that microservices solve them by "splitting processes", while Aura solves them by "in-process isolation"—the latter has no network overhead.

### Aura: A Standardized Fusion Scheme

Moving away from microservices is not a return to the traditional monolith but a move toward a new standardized form: **a single process containing complete distributed capabilities**. Most current de-microservicing practices are ad-hoc—each team pieces together different components according to its own scenario (embedded KV + a self-built event bus + hand-written state management), lacking a unified architectural pattern.

Aura standardizes this fusion into a reusable engine:

| Capability | Traditional patchwork | Aura's standardized scheme |
|------|------------|----------------|
| Service orchestration | A self-built booth system + hand-written state machine | The realm model (emit/on/on_join/on_batch) + automatic routing by `interface_schema()` |
| State management | Redis / Memcached / self-built HashMap | Fjall LSM-Tree (WAL power-loss protection + Scale-to-Zero) |
| Cross-node coordination | Each service implements it on its own or does not | Metadata independent per node (the meta table inside the data-plane okm instance, control-plane single-writer, no consensus—ADR-0025); booth data goes Fjall+Lakehouse or SlateDB+S3; federated nodes authenticate via a well-known protocol |
| Multilingual support | Each service has its own runtime (microservice mode) | In-process embedding (Steel/Python/Wasm), zero serialization |
| Deployment | Docker + K8s + Helm + service mesh | Single binary, `scp` is deployment; multi-machine networking goes through federation (well-known protocol authentication), no shared metadata infrastructure needed |

The significance of standardization is that developers no longer need to redesign "how to put multiple frameworks together" for each project, but instead directly use a proven fusion engine and focus their energy on business logic.

### The Boundary of Fusion: Not a Distributed Monolith

Fusion does not mean having multiple services share one database (a distributed monolith). A distributed monolith simultaneously bears the deployment complexity of microservices and the coupling intensity of the monolith—changing one service's schema drags in the entire line, yet you do not enjoy the monolith's simplicity. Aura's approach is **physically a monolith, logically distributed**:

- **Physical level**: single binary, in-process communication, zero network RTT—taking the monolith's performance and simplicity
- **Logical level**: each booth type occupies a type ns (an okm key segment, declaring collections, ADR-0026); an instance is a document within the ns, and they communicate loosely coupled through realm events (emit/on); at runtime they are activated and evicted by instance key—taking microservices' decoupling and elasticity

| Dimension | Microservices | Distributed monolith | **Aura** | Monolith |
|------|--------|-----------|---------|------|
| Communication | Network RPC | Shared-database JOIN | **In-process emit/on** | In-process function call |
| Coupling | Loose | Tight (shared schema) | **Loose (event contract)** | Tight |
| Deployment | Multiple binaries | Multiple binaries | **Single binary** | Single binary |
| Language | Per-service choice | Per-service choice | **Per-booth choice** | Globally unified |
| State isolation | Naturally isolated | **No isolation (shared DB)** | **Independent Fjall partition** | Shared memory |

Aura fuses the decoupling capability of microservices with the performance advantages of the monolith, while avoiding the inherent defects of both—and all the costs of the distributed-monolith anti-pattern.


## 1. Compute Sinking and Core Architecture Topology

In a modern general-purpose development environment, blindly introducing an external volatile in-memory cache has been proven to bring serious engineering debt and cognitive burden. This architecture, based on first principles, establishes the core principles of "compute hugging storage" and "atomic monolithic transactions", and completely rules out the inefficient pattern of Application-Side Joins (application-side assembly).

### 1.1 Failure Analysis of Traditional "Multiple Network Round Trips (N+1 Queries)"

When handling complex domain models, an external cache forces developers to play the role of an inefficient "pseudo query engine" with code on the application side.

- **Wasted data movement**: to assemble a business object with multiple layers of nested relationships (for example: user → order → product → inventory), the application server must frequently initiate multiple network round trips (RTT) over the wire.
- **Compute overhead**: the application server burns a large number of CPU cycles repeatedly serializing and deserializing the massive streams of RESP, JSON, or Protobuf data transmitted over, merging fragments in application memory.

### 1.2 Design Mechanism of This Architecture

- **Core formula**: true total latency = network RTT + database internal processing time

Because the physical RTT inside a modern LAN/VPC is often as high as 1.0ms to 3.0ms, the pure in-memory advantage of an external cache—a few tenths of a millisecond—is meaningless in the face of network latency.

- **Compute Near Data**: by directly squeezing the embedded KV's LSM-Tree Block Cache and the in-process zero-copy path, it achieves a single request, a single return, straight to the essence.

→ See [Redis: the "Network RAM" Trap](redis-critique-en.md), [MySQL Critique](mysql-critique-en.md), [Unified Data Layer](unified-data-layer-en.md)



To remove the runtime overhead brought by the JavaScript/TypeScript runtime (Rivet), as well as the execution loss and environment pollution brought by Python, this architecture adopts a two-layer decoupled topology of **Rust (high-performance shell) + Steel (deterministic core)**.

```
[Network request (REST/WS)]
│
▼
[Tokio async event dispatch] ──► [Wake the Rust booth with the matching ID] ──(read the latest Context from local Fjall disk)
│
▼
[Booth flushed to disk and asleep (Scale to Zero)] ◄──(single WAL write to Fjall) ◄── [Steel Lisp VM kernel running]
```

The natively implemented Rust booth engine uses Tokio MPSC channels to build a low-overhead Host exoskeleton, avoiding multi-threaded data stomping. Each booth is closely bound to memory state (no shared locks), hibernates when idle (Scale to Zero), and on being woken by a request instantly spins up a lightweight Steel Lisp VM as its highest strategy core. Host and Steel Lisp are bridged directly through memory pointers, zero-copy mapping memory state into Lisp variables. For the complete zero-shared-lock booth implementation, see [§2.2 Polyglot Gateway](#22-polyglot-gateway-pure-rust-hybrid-booth-implementation), where `exec_steel_lisp` is the single-Steel version of the implementation.


## 2. Architecture Upgrade: Multilingual Hybrid Runtime (Polyglot Embedded Harness)

This architecture has evolved from a single Steel Lisp strategy layer into a multimodal hybrid runtime console. By establishing a unified "Polyglot Bridge" inside the Rust main booth, it dispatches the most suitable "handyman" for different business scenarios, while maintaining the core design of fast startup and Scale to Zero.

### 2.1 Multilingual Hybrid: A User Selection Guide

The framework provides two embedded scripting languages + one sandbox runtime. **Each booth chooses only one scripting language**, but **there is no hard restriction on what a language is used for**—the "applicable scenarios" in the table below are only suggestions, and developers can freely allocate them according to actual needs. The framework does no intelligent dispatch.

| Technical choice | Type | Underlying VM / isolation mechanism | Core value | Applicable scenarios |
|---------|------|-------------------|---------|---------|
| **Steel (Lisp/Scheme)** | Scripting language | Bytecode VM + hygienic macros | 100% unambiguous, naturally sandboxed (dangerous system I/O not included by default) | High-risk strategy defenses, permission routing, complex metaprogramming DSLs |
| **Python (PyO3 embedded)** | Scripting language | Embedded CPython interpreter (GIL captured at memory level) | Zero-overhead reuse of the world's largest AI/Data ecosystem | Restructuring legacy AI code assets, Tokenizer matrix computation, Numpy/HuggingFace calls |
| **Wasm (Wasmtime)** | Sandbox runtime | Cranelift JIT compiler / hardware sandbox | Near-native CPU performance, hardware-level isolation | Hosting untrusted third-party code; storage-type Rust services (k10r/gravity and the like, whose storage is carried by the host via frame upcalls). Once wasm-gc + WASI mature, it can be extended into a unified runtime (see [WASM Unified Runtime](wasm-unified-runtime-en.md)) |
| **Rust main shell** | Host | Tokio async coroutines + MPSC state channels | Low-overhead startup, 27MB memory, core Scale-to-Zero design | Distributed gateways, system I/O, event dispatch |
| **Persistence** | Storage layer | Fjall/SlateDB (data/meta two instances separated, metadata single-writer no consensus) | LSM-Tree + Raft consensus | No external cache middleware, single-transaction atomic solidification |

**Key clarifications**:

- **Scripting languages (Steel/Python) are the carriers of a booth's business logic**; each booth chooses one.
- **Wasm is an isolation runtime**, and its positioning has undergone one expansion:
  - Its initial positioning was "hosting untrusted third-party code"—the writing language is usually Rust, but the host does not care and only consumes the `.wasm` binary.
  - As Rust services like k10r and gravity enter the realm in the form of wasm booths, its positioning expanded to **heavily isolated service hosting, and the sole release form for Rust services**: a service is compiled to `.wasm` and uploaded at runtime (`set(lang="wasm", bytes)`), not compiled into the host.
  - The consequence of compiling into the host: each application forks a copy of aura, adding a service means repackaging, and the platform degenerates into a framework.
- **The engine does not provide in-process Rust booths**: the framework's own mechanisms (the evictor and the like) are ordinary logic within the realm, and wrapping them into a booth for a roundabout trip is pointless. There is only one way for Rust code to become a booth: compile it to wasm and upload it—not a dylib, not compile-time, not a closure.
- **Storage does not enter the sandbox either**: the OKM schema that ships with a service is compiled into wasm as-is, the `VirtualStorage` implementation is replaced with frame upcalls (okm-wire frames), and the host-side NestStorage executor carries the physical storage under the app ns prefix allocated by the registry (Krystallizer ADR-0007 storage carrying split; static derive, okm-dynamic not needed).
- **The wasm carrier ABI has landed**:
  - Handlers are exported by event name (multiple entry points).
  - Values cross linear memory as CBOR: the host writes arguments via the guest's `aura_alloc`, calls `(ptr, len) -> i64`, and the return packs `(ptr:u32)<<32|len:u32`.
  - The ctx bridge host imports are registered in the `aura_host` namespace with a unified signature.
  - An explicit `interface_schema` export takes priority; otherwise it is inferred from the export manifest.
  - JSON appears only at the host-side `ResidentSession` boundary; the storage layer and the wasm bridge have zero JSON text.
- **There is no hard rule on language use**: Steel can write business logic, Python can write strategy, as long as the developer considers it appropriate.

**Persistence choice supplement**: booth messages have strong **temporal locality**—hot spots concentrate on the most recently written state, SlateDB lands in the memtable / cache as soon as it is written, and cold reads from S3 are triggered only by occasional point queries, which is acceptable. Therefore **SlateDB can replace the "Fjall + Lakehouse" pair** (single source of truth, no flush pipeline); only when the hot path has a hard requirement for in-process ns-μs and reads have no temporal locality does it fall back to Fjall. If strongly consistent distributed replication is needed, do not build it yourself—use FoundationDB / TiKV directly. See [KV Storage Engine](kv-storage-engine-en.md) §Two Distribution Paths / Strongly Consistent Distributed and [Unified Data Layer](unified-data-layer-en.md).

**The design rationale for language restriction**: the framework is restricted to three languages—Python/Steel/Wasm—not because it "cannot support more", but to **restrict the AI's generation space**. The quality of AI-generated code is inversely proportional to language complexity—the fewer the languages and the clearer the boundaries, the more reliable the AI-generated code. The component set is closed and finite (an enum gatekeeper + adding components by manually extending the enum), consistent with Aura's booth enumeration design: the AI needs a complete vocabulary to correctly generate JSON, and the number of languages affects judgment accuracy. Not restricting languages looks flexible but in fact pushes the burden of choice onto the AI and the developer—the AI has to pick the optimum among infinite options, and the developer has to debug compatibility among infinite combinations. Restricting languages = reducing cognitive complexity = improving generation quality.

**The evaluation standard for engines vs applications**: Aura is an engine and platform, not an application. Evaluating an application asks "what problem did you solve" ("just use PostgreSQL" is a reasonable application-layer answer); evaluating an engine asks "what problems can you enable people to solve" (an engine's value lies in the number and complexity of the upper-layer applications it empowers). Evaluating an engine by application standards—"you don't need to build an engine, a horse cart is enough"—is a confusion of levels. An engine's rival is not PostgreSQL but other engines (Akka, Orleans, Tempo). An engine's customer is not the end user but the developer who builds applications with the engine.

**The rationale for choosing Steel**: Steel does not have "special status"; it is one of the three languages. But S-expressions have unique engineering value: parentheses `()` physically lock down all scopes and operation boundaries, whereas the "whitespace = semantics" design of whitespace-sensitive languages (Koto, Python) creates unpredictability in team collaboration and Git merges. Steel's zero ambiguity makes AI-generated code more reliable—Lisp's homoiconicity makes Transformers naturally good at generating it, and S-expressions are the syntax the AI is most likely to get right. But Steel has low data density (little code on GitHub), so when the AI writes it bare it hallucinates by invoking intuitions from standard R7RS Scheme—it needs to be constrained with Few-Shot in `CLAUDE.md`.

→ See [Embedded Script Language Selection](embedded-script-languages-en.md)

**The simplification path once wasm-gc matures**: Rust → Wasm is already available now (wasm-gc not needed), and async scenarios can be covered by Rust → Wasm. Once wasm-gc lands, languages such as Kotlin/C# will flow into the Wasm ecosystem, further enriching language choices. Steel and PyO3 are unaffected—Steel's S-expression deterministic boundaries cannot be replaced by Wasm; Python is too dynamic and compiling to Wasm is extremely hard or even infeasible, but that is not needed—PyO3's in-process zero-copy call overhead is inherently low, and the interactive mode of executing immediately without compiling is Python's inherent value. See [WASM Unified Runtime](wasm-unified-runtime-en.md).

**No hard language restriction**: the framework connects external languages through two channels, PyO3 and WASM; in theory any language that can compile to WASM or be called through the C ABI can enter the booth process. The framework has three built-in runtimes—Steel/Python/WASM—but does not prevent users from connecting other languages (such as Lua, Julia) through PyO3's C extension mechanism.

**External calls**: the framework provides synchronous calls through `ctx.invoke()` + a unified call registry (see the aura repository [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §ctx.invoke unified call primitive). The Host governs the call lifecycle (timeout, auditing, observability), and Fluxora handles HTTP requests. Booths do not call external systems directly through PyO3—that bypasses Host governance. In-process calls (PyO3/WASM) always take priority over cross-process calls.

**Streaming calls (iterate)**: the fourth primitive alongside emit/on/invoke (ADR-0034): `ctx.iterate(target, handler, args)` returns a cursor and pulls typed envelopes one by one (ADR-0036 unification: `done` is always boolean, non-terminal carries `item`, terminal may carry `value`—invoke is a 1-stream that terminates on the first round, and `ctx.invoke` is its value-taking sugar)—termination, backpressure, and consumer liveness are built into the pull structure, not conventions. The producer form branches by language: python writes a `yield` generator (the framework parks and drives it; exhaustion means done); steel/wasm/bgi (nushell's PTY carrier has been retired, and nu booths take the bgi dual-fifo form) have no host that can drive a generator, so the handler writes envelopes explicitly (a Rust-wasm guest maps `Iterator` within module state). The consumer form likewise branches by language: python scripts use the native `ctx_iterate(...)` generator (break auto-disposes), Rust uses a cursor, and the rest pull until done; each pull is a hot call riding the existing call machinery (streams are not persistent and not replayable—for at-least-once, use events). Motivating scenario: a provider booth (python `httpx.stream` consuming SSE → yield per token) is consumed by a sibling wasm booth, implementing ADR-0031 "external access belongs to booth code".

**Layering for AI Agent/Harness scenarios**:

An Agent's logic is divided into two layers—the orchestration layer and the tool layer—implemented in different languages:

- **Agent main loop (Wasm/Rust)**: receive → LLM call → tool call → return. Stable, performance-first. A Wasm booth calls `ctx.invoke()` through a host function; the Host performs the async operation while the Wasm is suspended, transparently to the Wasm.
- **Tool integration** has two modes:
  - **Skill-based**: all tools go uniformly through the `perform()` interface—structured, composable, discoverable. Built-in operations (file I/O, network requests, HTTP clients) and business logic are all encapsulated as Skills, and the Agent calls them through a unified interface. Emergent Skills are automatically clustered through high-weight subgraphs in graph-memory, and the Agent discovers and calls them at runtime.
  - **Non-Skill-based**: direct host functions (Rust/Wasm) + scripts (Python). Lightweight scenarios that do not need the overhead of the Skill framework. General stable operations go through host functions, and business-specific logic is registered via invoke.toml.

The Agent main loop runs in a single booth instance (instance key = session_id). Tool calls within the same session are serial (the booth's single-threaded semantics), and different sessions run in parallel. After handling a request, the Agent enters `on_sleep`, the state is flushed to Fjall, and memory drops to zero (Scale-to-Zero). When the next message arrives, `on_wake` activates and restores it. Streaming output is implemented by frequently emitting `agent.stream` events, which Fluxora converts into SSE/WebSocket pushes.

The main loop itself can be further shrunk into a pure function—the session is the data: the booth fetches the complete session from the memory system by `session_id` → runs a turn → stores the session; session persistence and restoration all sink into the memory system, and the booth is stateless between two calls, with `on_sleep`/`on_wake` degenerating into two actions, fetch and store. The loop no longer holds session state and does not rewrite history; all function calls are ordinary records to it. The trimming of the tool-call format happens at the serialization view layer of the memory system, and the storage layer is always complete.

The complete component architecture of the stateless Agent is written up separately: the loop component (Gravity), the entry point (Prism), the executor (Effector, in both embedded and remote executor forms), the skill emergence loop, and the transport verdict—see [Stateless Agent Architecture](stateless-agent-architecture-en.md); for the memory-side design, see Krystallizer ADR-0006 (stateless Agent integration).

Tools and Skills are not static files but subgraphs in graph-memory with a high tool index—usage data automatically clusters to emerge Skill boundaries; at runtime the Agent discovers relevant Skill edges through vector search and injects them into the context ordered by weight.

#### 2.1.1 Multi-Skill Orchestration: Pipeline as Code

**Problem**: the standard pattern of current Agent frameworks (Hermes, Agno, etc.) is LLM real-time orchestration—every Skill call is an independent inference, and context is passed along in the conversation. By the 3rd or 4th Skill call, the LLM may have already "forgotten" the key fields returned by the 1st Skill. With the same input, different runs may take different paths. The orchestration process is a black box, and errors are invisible.

**Constraint**: Aura forbids the LLM from generating code by string concatenation (§2.1 the design rationale for language restriction: restrict the AI's generation space, improve generation quality). But multi-Skill orchestration requires deterministic execution.

**Solution**: a three-layer separation—the LLM outputs structured intent, and a Rust Pipeline Runner executes deterministically.

```
User request → LLM (planning, outputs a YAML pipeline definition)
              → Pipeline Runner (Rust, deterministic execution: read YAML → call Skill → pass data)
```

What the LLM generates is YAML, not code:

```yaml
# The pipeline definition output by the LLM (structured intent)
pipeline:
  - skill: order-fetch
    params: { status: 1 }
    output: orders
  - transform:
      type: filter
      source: orders
      expression: "amount > 100"
      output: large_orders
  - skill: pricing-calculate
    input: large_orders
    output: pricing
  - skill: payment-create
    input: pricing
```

The Pipeline Runner (Rust) executes directly, without going through bash:

```rust
// pipeline_runner.rs — deterministic executor
async fn run_pipeline(yaml: &str) -> Result<Value> {
    let pipeline: Pipeline = serde_yaml::from_str(yaml)?;
    let mut ctx: HashMap<String, Value> = HashMap::new();

    for step in &pipeline.steps {
        match step {
            Step::Skill { name, params, output } => {
                // A Skill receives JSON on stdin and outputs JSON on stdout
                // The Runner only does plumbing: stdout(A) → stdin(B), no deserialization
                let input = resolve_refs(params, &ctx);  // replace $ref references
                let result = execute_skill(name, &input).await?;
                ctx.insert(output.clone(), result);
            }
            Step::Transform { source, expr, output } => {
                // Only transform steps require the Runner to parse JSON
                let data = ctx.get(source).unwrap();
                let transformed = apply_transform(data, expr)?;
                ctx.insert(output.clone(), transformed);
            }
        }
    }
    Ok(ctx)
}
```

**Key design**: for Skill-to-Skill the Runner only does plumbing (stdout → stdin), no deserialization. The Skill script already handles JSON parsing internally. Only Transform steps require the Runner to read and write JSON.

**How the three-layer separation satisfies the constraints**:

| Constraint | How it is satisfied |
|:---|:---|
| Aura forbids the LLM from generating code | The LLM outputs YAML, not Rust/bash |
| Orchestration needs deterministic execution | The Runner is deterministic: input YAML, fixed execution path out |
| Observability is needed | YAML is auditable; the Runner log records each step's input/output/Exit Code |
| Intervention is needed | Change the YAML or change the Runner, not the LLM |
| Existing Skills must be reused | The Runner calls the Skill CLI through stdin/stdout JSON, compatible with existing Typer interfaces |

**Comparison with real-time orchestration**:

| Dimension | Real-time orchestration (status quo) | Pipeline Runner |
|:---|:---|:---|
| Number of LLM invocations | One inference per step | One at planning time |
| Context | Continuously growing, may drift | Complete at planning time, no LLM during execution |
| Determinism | Probabilistic; the same input may take different paths | Deterministic; the same YAML gives the same execution |
| Cost of errors | Every execution may err | Paid once at generation, correct forever after a fix |
| Observability | Black box | YAML + Runner logs all visible |

**Prerequisite**: each Skill's interface declaration needs explicit input/output schemas (structured parameter definitions) so that the Runner can validate the data flow in the pipeline. The current Skill's perform() interface already has parameter type declarations; the output schema needs to be added.

### 2.2 Polyglot Gateway: Pure Rust Hybrid Booth Implementation

Embed all four embedded language runtimes into the same Tokio async booth instance to create a multimodal hybrid execution environment. The entire call process is streamed and interwoven entirely in process memory (In-Memory):

```rust
// src/main.rs
use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

// 1. Bring in the various new and traditional embedded language runtimes
use steel::steel_vm::engine::Engine as SteelEngine;           // Lisp
use wasmtime::{Engine as WasmEngine, Store, Module, Instance}; // Wasm Core
use pyo3::prelude::*;                                          // Python 3

// Define the multilingual task scheduling protocol
#[derive(Debug)]
pub enum PolyglotTask {
    RunLisp { script: String, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
    RunWasm { bytecode: Vec<u8>, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
    RunPython { code: String, func: String, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
}

pub struct MasterAgentBooth {
    agent_id: String,
    receiver: mpsc::Receiver<PolyglotTask>,
    memory_context: ciborium::Value, // dynamic structured state, a CBOR Value tree
}

impl MasterAgentBooth {
    pub fn new(agent_id: String, receiver: mpsc::Receiver<PolyglotTask>) -> Self {
        let mem = ciborium::Value::Map(vec![
            ("agent_status".into(), "active".into()),
            ("engine_version".into(), "2026.06".into()),
        ]);
        Self { agent_id, receiver, memory_context: mem }
    }

    pub async fn run_loop(mut self) {
        while let Some(task) = self.receiver.recv().await {
            match task {
                PolyglotTask::RunLisp { script, input, respond } => {
                    let res = self.exec_steel_lisp(&script, &input);
                    let _ = respond.send(res);
                }
                PolyglotTask::RunPython { code, func, input, respond } => {
                    let res = self.exec_embedded_python(&code, &func, &input);
                    let _ = respond.send(res);
                }
                // Wasm can also be spun up, executed, and released in memory following this pattern
                _ => {}
            }
        }
    }

    // =================================────────────────====================
    // Engine A: Steel Lisp (the strategy brain, using parenthesis boundaries to prevent errors)
    // =================================────────────────====================
    fn exec_steel_lisp(&self, script: &str, input: &[u8]) -> Vec<u8> {
        let mut vm = SteelEngine::new();
        // Take agent_status out of the CBOR Value
        let status = self.memory_context.get("agent_status")
            .and_then(|v| v.as_text()).unwrap_or("idle");
        vm.register_value("current-status",
            steel::steel_vm::as_values::AsRefSteelVal::as_ref_steel_val(status).unwrap());
        // input is CBOR-encoded dynamic data; the Host decodes it and injects it
        let input_val: ciborium::Value = ciborium::from_reader(input).unwrap();
        vm.register_value("input-raw", /* ... */);
        match vm.run(script) {
            Ok(out) => {
                let result = out.last();
                // return the CBOR-encoded output
                let mut buf = Vec::new();
                ciborium::serialize(&result.to_string(), &mut buf).unwrap();
                buf
            }
            Err(e) => {
                let mut buf = Vec::new();
                ciborium::serialize(&format!("Lisp Error: {:?}", e), &mut buf).unwrap();
                buf
            }
        }
    }

    // =================================────────────────====================
    // Engine B: PyO3 Python (reuse the AI/data ecosystem; embedded memory throughput with no process friction)
    // =================================────────====================
    fn exec_embedded_python(&self, code: &str, function_name: &str, input: &[u8]) -> Vec<u8> {
        Python::with_gil(|py| {
            let status = self.memory_context.get("agent_status")
                .and_then(|v| v.as_text()).unwrap_or("idle");

            let activators = PyModule::from_code_bound(py, code, "agent_script.py", "agent_script");
            match activators {
                Ok(module) => {
                    // input is CBOR-encoded dynamic data
                    let input_val: ciborium::Value = ciborium::from_reader(input).unwrap();
                    // The Host converts the CBOR Value into a Python dict and passes it in
                    let py_input = cbor_to_pyobj(py, &input_val);
                    let args = (status, py_input);
                    if let Ok(func) = module.getattr(function_name) {
                        if let Ok(py_res) = func.call1(args) {
                            // convert the return value back to CBOR encoding
                            let result_val = pyobj_to_cbor(py, &py_res);
                            let mut buf = Vec::new();
                            ciborium::serialize(&result_val, &mut buf).unwrap();
                            return buf;
                        }
                    }
                    let mut buf = Vec::new();
                    ciborium::serialize(&"Python execution failed", &mut buf).unwrap();
                    buf
                }
                Err(e) => {
                    let mut buf = Vec::new();
                    ciborium::serialize(&format!("Python VM Error: {:?}", e), &mut buf).unwrap();
                    buf
                }
            }
        })
    }
}
```

### 2.3 Completely Breaking the NoSQL Hangover: The "Monolithic Relational Last Stand" Under Multilingual Hybrid

Many trend-following architects, on seeing that this scheme supports multiple embedded languages, immediately react: "So do I have to build a complex state-synchronization layer, or introduce an external in-memory database, for these languages to share data?"

**No. This is precisely where the architecture is most clear-headed and most lean:**

- **Unified memory space**: the Steel VM, Wasmtime, and Python's PyO3 are all compiled into the memory address space of the same Rust process, either as dynamic links or as C-Binding statics. Booth state lives in the type declaration's collections (ADR-0026 §3); an instance is a document within the type ns, and the document's dynamic segment is read and written field by field; the languages bridge through the ctx.store.emit instruction—after parsing the instruction the Host reads and writes the type's ns, converts to each language's native types, and injects them into the VM. In-process calls, no cross-process IPC, no network serialization overhead.

**State is stored separately by collection/document**: booth state is not a single large CBOR blob but is organized by the collections declared by the type:

```
type ns (allocated at registration) /
  history/{instance_key}   = [10000 records]
  profile/{instance_key}   = {...}
  settings/{instance_key}  = {...}
```

The handler emits read/write instructions on demand, without loading collections it does not need:

```python
def handle(ctx, user_message=None):
    if user_message:
        # Only read history in this type's collections, without touching other collections
        recent = ctx.store.emit(get_document("history", ctx.self_id.key))

        recent["records"].append(new_record)
        ctx.store.emit(put_document("history", ctx.self_id.key, recent))

        # Already flushed to disk (WAL + memtable)
```


- **The dynamism of the data payload**: a booth's input/output is `ciborium::Value` (an in-memory value tree) in-process, with zero serialization. At cross-language boundaries (Steel/Python/Wasm VM) it is serialized to CBOR `Vec<u8>`—the schema is not fixed and supports nested objects, arrays, numbers, and booleans. Steel Lisp, Python (the cbor2 library), and the Rust Host can all encode and decode it. Unlike Postcard (which needs compile-time Rust types), CBOR is self-describing and suited to cross-language dynamic data exchange.

**CBOR improvement direction**: the current CBOR key name has overhead in high-frequency small-message scenarios—every field carries a full string key name. Improvement directions: eliminate the key-name overhead (key-name indexing or column-family-style storage), with a dynamic columnar layout (similar to an LSM-Tree's column-family design), but without dragging down query performance. This is an engineering optimization, not an architectural change—the choice of CBOR as the cross-language data-exchange format is unchanged; only the encoding efficiency improves. Note: this is not heading in the direction of "schema constraints" (Rust struct state is already type-safe), but compressing the encoded size while preserving dynamism.

- **Atomicity of state flushing**: in one round of interaction, the multiple languages write the WAL to disk through the `ctx.store.emit` instruction (ADR-0026 §3: state lives in the collections declared by this type, and an instance is a document within the type ns). Fjall's LSM-Tree naturally supports high-frequency small writes. Booth state is written to local Fjall (not over the network path); metadata (booth registration, routing, this node's configuration) is written to the meta table inside the data-plane okm instance (booth Def, ADR-0025), with a single-writer control plane + node cache—**user data (including login state) is bound to its owning node and is not globally synchronized**: if a user logs in on another node, that node does not have their historical data, and across nodes only identity is authenticated by the well-known protocol.

**There is no multi-round RTT consumption over the wire, no volatility risk of an external cache, no distributed bugs from multi-database synchronization.**

Rust builds the exoskeleton, Lisp locks the boundaries, Python plugs into the ecosystem, Wasm isolates the black boxes, and the persistence layer uses Fjall/SlateDB (data/meta two instances separated).


## 3. Storage Architecture

The storage layer makes the engine pluggable through traits—the storage engine (KV read/write) and the distribution layer (multi-node coordination) combine orthogonally. Key points:

- **Dual-track implementation**: `FjallEngine` (local NVMe, synchronous I/O wrapped via spawn_blocking, sub-millisecond latency for private deployments) and `SlateEngine` (SlateDB → S3, inherently async, cloud-native stateless compute)
- **A single API**: `ctx.store.emit` (booth state, the collections declared by this type, not over the network path—ADR-0026 §3); the registry/shard map and the like are internal engine structures and do not expose a read/write surface to booths (ADR-0025)
- **Starting single-node ≠ a distributed fate**: Fjall starts single-node and data is not replicated; SlateDB+S3 is the cloud-native mode; metadata introduces no consensus—each node is independent, the control plane is single-writer; cross-node evolution follows the federation form (well-known protocol authentication)
- Booth state has strong temporal locality (hot spots concentrate on the most recent writes); a SlateDB memtable hit makes cold reads from S3 acceptable, and it can replace the "Fjall + Lakehouse" pair

For complete design details (trait signatures, the AuraCollection binding container, the distribution layer, configuration evaluation, multimodal routing), see the **aura repository [`docs/design/storage.md`](https://github.com/orbsh/aura/blob/main/docs/design/storage.md)**.


## 4. Serialization Protocol Analysis and Comparison

→ See [Serialization Protocol Comparison](serialization-protocol-comparison-en.md)

### Aura's Layered Serialization Strategy (updated 2026-06-20)

| Layer | Serialization format | Rationale |
|------|-----------|------|
| **Booth state persistence (Fjall)** | CBOR | Dynamic structured data, self-describing, cross-language (Steel/Python/Rust) encode/decode |
| **Booth input/output payload** | `ciborium::Value` (in-process) / CBOR bytes (across boundaries) | In-process it is an in-memory value tree, zero serialization; when persisting/replicating/externally delivering it is serialized to CBOR bytes |
| **Raft metadata (LogId, etc.)** | Postcard | Used only by external strongly consistent schemes (TiKV, etc.); this architecture has no consensus layer by default |
| **Analytical query path** | Arrow IPC | Columnar alignment; after reading from Fjall, Polars zero-copy casts to a DataFrame |
| **KV value read-only point query (candidate)** | rkyv | Pure Rust mmap zero-copy, O(1) point query; only for the in-process hot path, while cross-language/egress still goes CBOR/Arrow (see the serialization document, scheme B3 landing boundary) |
| **Cloud long-term memory Lakehouse** | Lance | Embedded vector and inverted indexes, native support for streaming retrieval from remote S3 |
| **Massive cold-history hibernation** | Parquet | High-compression cold storage |
| **UI ↔ Gateway** | CBOR | Cross-language (WASM), self-describing |

**Core principle**: technology should not be red tape that ties developers' hands. Only by seeing the boundaries of the physical hardware clearly and removing needless IDL nesting can the Aura engine run efficiently between multi-core CPUs and NVMe.

---


## 5. Realm Model: Inter-Booth Interaction and the External World

The realm (Event Realm) is the event space inside the engine: booths do not address each other directly but interact through `emit(name, data)` / `on(name, fn)`—the emitter does not care who handles it, and the handler does not care who emitted it. The event name is the reference, and the instance key is the instance locator. Key points:

- **The ctx boundary** (ADR-0043): ctx is the host-provided global namespace object for the script's names (bound once at load time, alive for the session)—`ctx.invoke / ctx.emit / ctx.store / ctx.iterate / ctx.timer.* / ctx.queue.* / ctx.schema / ctx.on / ctx.receives`; the payload goes in the handler parameter position (the natural reading of gRPC's `func Foo(ctx, req)`), and `ctx.return` does not exist (the entry point returns natively). `ctx.on` is the registration primitive (`@on`/`on` are its sugar), **legal only during activation** (a phase gate—calling it in the handler phase = an error value); the `__sleep`/`__wake` lifecycle hooks are folded into its reserved names (entering the schema hooks array, not going through MQ routing). On the Rust side, `Handler = Fn(&Ctx, Value)`—ctx is a long-lived global borrow, not a value the handler holds or threads through
- **interface_schema()**: derived from the `@on` decorator (receives: event name → key field; wildcards go into wildcard_receives)—emits are not declared, not collected, not validated (ADR-0012): the receiver set is a runtime fact, and an emit with no subscriber falls into the dead-event ring, which is the observable audit surface
- **Instantiation and sharding**: the same instance key is serial (state consistency), different keys are parallel; **data follows its owning node (the federation verdict)**—the instance key scope is internal to the node, with no global placement and no cross-node resharding
- **Cross-Partition queries**: projection booths (continuous aggregation, recommended) / Arrow HTAP (ad-hoc columnar scans)
- **Unified call model**: `ctx.invoke()` is the only governed call surface; a cold call (reaching a human/external system) waits without entering park, writes a suspension event to the event stream, and `resolve_call` re-enters
- **Delivery semantics**: in-process events are causally ordered; there is no consensus layer—cross-node follows the federation form, and inter-realm messages are explicitly addressed (no global event ordering); for the physical basis and scale quantification, see [Distributed Collaboration Topology](distributed-collaboration-topology-en.md) §2
- **Persistent event queue (ADR-0014)**: the storage form of event delivery is an okm embedded persistent partition (`[mq-data][event][part_id][time]` + `[mq-cursor][event][part_id][booth]`), not an in-memory channel—emit flushes to disk immediately (passive saving and the active saving of booth state are dual tracks of the same engine), backlog during instance eviction is delivered as usual after reactivation (scale-to-zero does not drop triggers), slow-consumer backlog is visible and can "skip to latest" (a skip-to-now fallback valve), and N subscribers = one partition + N cursors (the N copies of a per-booth mailbox structurally disappear). **Retention = the minimum watermark of active subscribers** (the denominator comes from the routing registry; stale cursors do not pin the watermark; the backlog depth is a real-time count from an okm reduce over the mq-data prefix); the queue is a buffer, not storage—at-least-once holds only while the subscriber stays registered, and booth state (type collections) is the durable truth. "No queue component" means no external heavyweight queue; an embedded persistent partition is the natural form of passive event saving. The current implementation is a broadcast transitional form, with the persistence rewrite in 4.5c step 2b

For complete design details (the routing-table structure, the four trigger modes on_join/on_batch/on_debounce, the Collector, wildcard matching, delivery code, host implementation, MQ decomposition), see the **aura repository [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md)**.

## 6. Developer Experience (DX): From Inspiration by Rivet Actors to Aura's Improvements

> Inspired by Rivet Actors' Actor-model developer experience, but Aura has made fundamental improvements in type safety, cold-start performance, state persistence, and multilingual support.

### 6.1 Rivet Actors' DX Baseline and Aura's Improvements

Rivet Actors provides an excellent booth development experience: a TypeScript SDK, automatic HTTP endpoint generation, built-in state management, WebSocket connection management, and a local development server. But its core limitations are:

| Dimension | Rivet Actors | Aura's improvements |
|------|-------------|------------|
| **Language** | TypeScript/JavaScript (V8 isolation) | Rust core + Steel Lisp/Python/Wasm embedding (in-process, no IPC) |
| **System startup** | Hundreds of milliseconds (Node.js process + V8 initialization) | Millisecond level (Tokio runtime + Fjall open) |
| **Booth wake-up** | A few milliseconds (V8 VM activation) | Microsecond level (instant creation of the Steel bytecode VM; Python PyO3 ~1ms) |
| **State storage** | SQLite (co-resident on the same machine, but a single-machine bottleneck) | Fjall LSM-Tree (embedded; metadata independent single-writer per node, data into the lake/S3) |
| **Type safety** | TypeScript (runtime types, weak at compile time) | Rust compile-time strong types + Steel Lisp's S-expression zero ambiguity |
| **Multilingual** | Only JS/TS | Rust/Steel/Python/Wasm four languages mixed in-process |
| **State persistence** | SQLite file | Fjall KV + CBOR serialization |

### 6.2 Booth Definition and Lifecycle Booths are implemented via a `set(lang, script)` submission (see the aura repository [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §booth definition interface). A booth is **multi-entry**: each handler declares the events it listens to with the `@on(event, key=...)` decorator (the python decorator / the steel `on` function / the wasm export convention), and the event name is the handler's addressing name; `interface_schema` is derived from the decorators (receives is no longer declared by hand; emits are not declared or collected, ADR-0012), events that declare a key are delivered to a `(event, partition)` queue, and those without a key go to a single event queue—events belong to no booth, and one event queue can have multiple subscriber booths (one-to-many is structural). At upload time the Host introspects once and persists the metadata to the meta store; the execution path never calls `interface_schema`.

Lifecycle:

1. **Upload (set)**: an independent lifecycle that may never execute—the Host introspects `interface_schema()` (or derives it from the `@on` decorators) once, persisting the metadata (receives/lifecycle) together with the script to the meta store, and derives the delivery routing from receives
2. **Wake-up (`__wake`)**: when an event arrives, the Host activates the instance (the definition is already persisted, and state is read/written on demand from this type's ns collections via ctx.store); the unconsumed backlog events for that instance in the persistent queue continue to be delivered by cursor. `__wake`/`__sleep` are `ctx.on`'s reserved-name hooks (ADR-0043 §3 folding)—a script writes `ctx.on("__wake", h)`, Rust uses the construction-time `.hook("__wake", h)`, and the realm dispatches by HookBody form (Compiled direct call / Declared through the resident session) at activation/eviction
3. **Handling**: the handler is addressed and executed by event name; inside the handler it can `emit()`, and state is persisted immediately; `interface_schema` is never called
4. **Sleep (on_sleep)**: after being idle, state is flushed to Fjall and memory drops to zero (Scale-to-Zero; per-type idle_ttl determines the residency window)
5. **Version change**: a new `set` introspects once more and updates the persisted metadata and routing; the old metadata is governed beforehand

Scheduled tasks use declarative annotations and do not need an external CronJob:

```python
@cron("0 2 * * *")
def daily_sync(ctx):
    orders = ctx.store.emit(get_document("orders", ctx.self_id.key))
    sync_to_remote(orders["pending_orders"])
```

**Key differences from Rivet**:
- Rivet's booth state is a JS object serialized to SQLite; Aura's state is a document (nTLV encoded) in the type collections, landed in Fjall
- Rivet's cron is triggered by an external scheduler; Aura's cron is a declarative annotation inside the booth
- Rivet requires manually calling `booth.setState()`; Aura's set/append writes the WAL to disk immediately
- Rivet's booth logic can only be JS; Aura's `set()` supports switching languages at runtime

### 6.3 Event Discovery: The interface_schema() Convention

A script declares the events it receives with the `@on` decorator; at upload the Host introspects once (or derives from the decorators) to build the event routing table, persists the metadata to the meta store, and the execution path never introspects. See the aura repository [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §interface_schema.

**Hot reload**: script modified → reloaded → `interface_schema()` recomputed → routing table updated. No need to recompile the Rust host.

**The case of Rust booths**: booths written in Rust follow the same `interface_schema()` convention, but the macro generates it automatically at compile time, with no need to write it by hand:

```rust
// The #[aura::schema] macro scans at compile time and automatically generates the interface_schema() export
#[aura::schema]
#[derive(Serialize)]
struct Order { id: String, item: String }

// Event handling is registered through the macro
#[aura::on("order_created")]
async fn handle_order_created(ctx: Context, data: Order) -> Result<()> { ... }
```

**The only constraint**: a script must export `interface_schema()` and return JSON matching the structure. Violation means loading is refused. The script is the documentation—seeing `interface_schema()` tells you what events this booth receives and what events it emits.

### 6.4 Local Development Experience

Rivet has a `rivet worker dev` local development server. Aura's local development is lighter:

```bash
# Single-file startup—no Docker, no etcd, no database needed
$ aura dev order_booth.py

# Output:
# [Aura] booth order_booth started
# [Aura] realm: default
# [Aura] state: Fjall local mode (./data/booth/)
# [Aura] hot reload: watches file changes, edits take effect immediately
```

**Compared with Rivet**: Rivet's local development needs a Node.js runtime + a V8 isolation layer. Aura is a single binary, ready on download, with no runtime dependencies.

### 6.5 Testing Experience

```rust
#[cfg(test)]
mod tests {
    use aura::test::RealmHarness;

    #[aura::test]
    async fn test_order_flow() {
        // The test framework automatically creates a temporary Fjall store and realm, and cleans up automatically after the test
        let mut realm = RealmHarness::new("default").await;
        realm.set("python", "order_booth.py").await;

        // emit the event and verify the response
        realm.emit("order_created", {"item": "widget"}).await;
        let events = realm.collect("order_completed").await;
        assert_eq!(events.len(), 1);

        // State persistence verification—state is still there after a restart
        realm.restart().await;
        realm.emit("order_created", {"item": "gadget"}).await;
        let events = realm.collect("order_completed").await;
        assert_eq!(events.len(), 2);
    }
}
```

**Differences from Rivet**: Rivet tests need to start a local development server or mock SQLite. Aura's tests are purely in-process—the booth runs directly in the test process, Fjall uses a temporary directory, and there is no network and no external dependencies.

### 6.6 Deployment Experience

```bash
# Build—single binary, no runtime dependencies
$ aura build --release
# Output: target/release/order_service (12MB)

# Deploy to the target machine—no Docker, no K8S needed
$ scp target/release/order_service user@server:/opt/aura/
$ ssh user@server "aura serve order_service --port 8080"

# Multi-machine deployment—federation form: each node is deployed independently, mutually authenticated via the well-known protocol (data is not globally synchronized; it follows its owning node)
$ aura serve order_service --config /etc/aura/aura.kdl
```

**Comparison with K8S deployment**:

| Step | K8S deployment | Aura deployment |
|------|---------|----------|
| Build | Dockerfile → docker build → push registry | `aura build --release` |
| Config | Deployment YAML + Service YAML + Ingress YAML | `aura serve --port 8080` |
| State storage | PVC + StorageClass + PV | Fjall embedded (automatic) |
| Multi-machine metadata sync | StatefulSet + etcd + headless Service | No synchronization needed—metadata is independent per node (control-plane single-writer), mutual authentication between federations via the well-known protocol |
| Scaling | HPA + Metrics Server + CPU/memory thresholds | Booths automatically Scale-to-Zero |
| Certificates | cert-manager + ClusterIssuer + Certificate CRD | Built-in Let's Encrypt (optional) |

### 6.7 What Aura Has That Rivet Does Not

**1. Compile-time state safety**
Rivet's `booth.setState()` is a runtime API—passing the wrong type, forgetting to call it, or old data failing to deserialize after a state-structure change are all exposed only at runtime. Aura's state is a Rust struct, and the compiler intercepts all type errors at compile time.

**2. Multilingual scripting + a sandbox runtime**
Rivet's booth logic can only be JS. Each of Aura's booths can freely choose its scripting language—Python (AI/data), Steel (metaprogramming)—with different booths using different languages while discovering the event contract uniformly through `interface_schema()`. In addition, the Wasm sandbox runtime hosts untrusted third-party code with hardware-level isolation, complementing the scripting languages.

**3. Deterministic sleep/wake**
Rivet's booth sleep relies on a V8 heap snapshot, and waking requires deserializing the entire heap. Aura's booth sleep serializes the Rust struct via CBOR and writes it to Fjall; waking deserializes directly from disk into memory—it does not depend on the V8 heap format and is not affected by GC pauses.

**4. Distributed state replication**
Rivet's state is single-machine SQLite, and multiple replicas need external synchronization. Aura's metadata is independent per node (control-plane single-writer, no consensus), booth state goes to SlateDB+S3 or local Fjall, with no external components needed; federated nodes authenticate via the well-known protocol, and data follows its owning node.

### 6.8 Summary of DX Design Principles

1. **Single binary, zero dependencies**: developers need not install Docker/K8S/Node.js/Python; downloading the Aura binary lets them develop, test, and deploy
2. **State as code**: booth state is a document in the type collections (ADR-0026 §3), written to the WAL to disk immediately through the `ctx.store.emit` instruction. Rust booths enjoy compile-time type safety, and script booths enjoy JSON Schema validation
3. **Declarative lifecycle**: `on_wake`/`on_sleep`/`cron` annotations declare booth behavior, with no external scheduler needed
4. **Local is production**: local development uses a temporary Fjall directory and production uses a persistent Fjall directory, with 100% identical behavior—no "it runs locally but blows up online" problem
5. **Progressive complexity**: single-node → multi-machine networking goes through the federation form (each node independent + well-known protocol authentication; on the data side, local lake or SlateDB+S3), with no need to rewrite code or introduce new components

### 6.9 auractl: The CLI Management Tool

Aura provides a CLI tool `auractl` with two top-level subcommands:

**`auractl realm`** — realm management operations:

```bash
# Submit/update a booth definition
auractl realm set cart_booth python cart_booth.py

# List all booth definitions
auractl realm list

# View a booth's interface_schema
auractl realm schema cart_booth

# Emit an event to the realm (for debugging)
auractl realm emit add_to_cart '{"user_id": "A", "item": {"id": "X"}}'

# View active booth instances
auractl realm instances

# View dead-letter events
auractl realm deadletters

# View pending calls
auractl realm pending-calls
```

**`auractl fjall`** — Fjall data maintenance:

```bash
# View all partitions
auractl fjall partitions

# View a booth's state
auractl fjall get cart_booth.cart cart_booth:user_A

# Scan a partition's keys
auractl fjall scan booth_defs --prefix cart

# Compact the LSM-Tree (manually trigger compaction)
auractl fjall compact

# View storage statistics
auractl fjall stats

# Export booth definitions (backup)
auractl fjall export booth_defs --output cart_booths.cbor

# Roll back a booth definition to the previous version
auractl fjall rollback cart_booth --version 2
```

`auractl` reads and writes the local Fjall instance directly (single-node mode) or operates remotely through the Host API (distributed mode). The two subcommands have separated responsibilities: `realm` manages runtime state and events, and `fjall` manages the underlying storage and versioned data.

## 7. Panoramic Deconstruction: Competitive Analysis of Big-Tech Infrastructure and Emerging Agent Platforms

Before putting this architecture into real commercial Web services, we need to step outside the geek perspective and reconcile it, by strict industrial-grade standards, at a high dimension against several of the most representative agent runtime platforms currently on the market.

### 7.1 Cross-Generation Technology Matrix Quick-Reference Table

| Dimension / Platform | Google AX + Substrate | Rivet Actors | Windmill | This Architecture (Rust+Steel+PG/Fjall) |
|------------|----------------------|--------------|----------|------------------------|
| **Underlying Core Technology Stack** | Go / K8s control plane / container sandbox | Rust + V8 Isolates + FoundationDB | Rust (core) + Python/TS (execution) + Postgres | Pure Rust + embedded Steel VM + Fjall/SlateDB |
| **Language Overhead and Size** | Heavy (container Pod level) | Moderate (V8 process isolation) | Moderate (the multiple runtimes it depends on are fairly bulky) | Extremely light (single binary, ~27MB memory per session) |
| **Language Ecosystem Friendliness** | Model-agnostic, all-language support | Leans toward JavaScript/TypeScript | Heavily leans toward Python | Removes JS pollution, excellent for Rust/Lisp natively |
| **Cold Start / Recovery Latency** | ~200 ms (Pod memory snapshot thaw) | Milliseconds (instant V8 VM activation) | Tens of milliseconds (FaaS worker process scheduling) | Startup is Tokio runtime initialization (milliseconds); booth wakeup at microsecond level (embedded VM created instantaneously) |
| **State Retention and Anti-Amnesia** | Event log replay (WAL / Replay) | Compute and SQLite storage co-located on the same machine | Distributed async work queue (leans toward stateless short-lived tasks) | Fjall/SlateDB embedded storage (metadata single-writer per node, no consensus) |
| **Whitespace Sensitivity / Boundary Ambiguity** | Depends on the specific language running inside the container | Depends on JS/TS closure conventions | Suffers from a Python indentation and implicit type-conversion discontinuity | Zero ambiguity (S-expression parentheses give deterministic boundaries) |

> For a detailed DX-level comparison with Rivet, see [§6.1](#61-rivet-actors-dx-baseline-and-auras-improvements).

### 7.2 In-Depth Diagnostic Analysis of Core Competitors

#### ① The Google Camp (AX + Agent Substrate) — Industrial-Grade Heavy Armored Vehicle

- **Architectural Essence**: Google aims to take over large-scale enterprise agents by restructuring the Kubernetes (K8s) low-level scheduling. AX is responsible for achieving "in-place resurrection after disconnect" of long-running tasks through event log (Event Logging) replay, while Substrate is responsible for using Pod-level incremental memory snapshots to automatically freeze agents in suspended/waiting states (Scale to Zero), thereby achieving a 97% improvement in hardware utilization.
- **Technical Pain Points**: Because it takes the Pod container as the smallest isolation unit, the architecture is extremely large and heavily dependent on the K8s ecosystem. For small and medium-sized agile Web services, its deployment and operations cost and its cold-start response latency (~200ms-level latency is still too high in real-time Web scenarios) are extremely uneconomical.

#### ② The Rivet Camp (Rivet Actors) — Game-Grade High-Concurrency Lightweight Fighter

- **Architectural Essence**: Evolved from high-concurrency multiplayer online game infrastructure. It abandons containers and adopts a low-level runtime developed in pure Rust with embedded V8 Isolates VM islands. Each agent or long-lived session is, in memory, a highly agile booth with an in-memory SQLite bound to its side, achieving millisecond-level cold start and sleep recovery.
- **Technical Pain Points**: It bundles the JS ecosystem. Although it claims its low level squeezes performance out of Rust, its official SDKs and exposed application-layer APIs currently lean almost entirely toward JavaScript/TypeScript (Bun / Node.js). If you dislike JS runtime overhead, the looseness of dynamic typing, and V8's implicit memory consumption, Rivet becomes a huge obstacle to your technical fastidiousness.

#### ③ Windmill — A Python Arsenal with FaaS Genes

- **Architectural Essence**: A Code-first, high-concurrency distributed asynchronous task orchestration and lightweight FaaS platform. Its biggest highlight is zero intrusiveness—it directly uses Python's type hints (Type Hints) to automatically generate standard JSON Schema in the background, so that ordinary Python scripts can serve directly as Tool callbacks and be perfectly recognized by large models.
- **Technical Pain Points**: In essence it is not a resident agent platform. Its low level is designed for "short-lived jobs" and scheduled scripts. A Python task releases resources as soon as it finishes, lacking the event state machine that holds the line as in AX, and lacking the memory-level co-located state (Co-located State) of Rivet. When faced with dense "high-frequency long-running multi-agent interactions (Long-running Swarm loops)", the friction of stateless process spin-up appears far too heavy.

#### ④ Dapr — The Allure and Cost of the Sidecar Abstraction Layer

- **Architectural Essence**: A CNCF incubated project that, through the Sidecar pattern, attaches "building blocks" (State Management, Pub/Sub, Service Invocation, etc.) to each service, shielding underlying implementation differences behind RESTful/gRPC interfaces. Switching Redis → Kafka → NATS requires no change to business code.
- **Technical Pain Points**: Every call takes one hop through the Sidecar, compounding latency. The abstraction layer is too heavy—each building block is a layer of REST API wrapping, whereas de facto standard protocols already exist in each domain (Kafka's protocol, Postgres's protocol, S3's protocol); abstracting one more layer on top of a standard is a regression. Dapr's core value is cross-cloud migration and vendor lock-in avoidance, but in systems where the technology stack is controllable, using native protocols directly is always superior to an abstraction layer. Aura's approach is in-process integration: no Sidecar, no extra hops.


## 8. Single-Machine Start and Distributed Evolution

### 8.1 Core Insight: Starting Style ≠ Ultimate Data Destiny

Starting with a lightweight in-memory state management component in the main program never delays or prevents the system from later connecting to and using any vertical-domain specialized external database.

Just as in ordinary development we hand-write a `Mutex<HashMap>` in the main process or spin up a Tokio state channel as an execution-period cache and business state machine; this does not prevent us, when persistence is needed, from using a single-trip transaction connection (Single-Trip) to write the data into PostgreSQL along the way, or archive it into LanceDB on S3.

If, following this highest pragmatic philosophy of "starting style does not equal ultimate data destiny", we re-examine the reasonableness of "any project starting directly with Fjall", the core technical watershed is no longer "can another data engine be used", but how heavy the **"architectural mental load and lock-in cost"** you inject into your binary from day one is.

### 8.2 Comparison Table of Three Starting Modes

| Dimension | Mutex<HashMap> | External Consensus Start | **Tokio Booth + Fjall** ✨ |
|------|---------------|--------------|---------------------------|
| **Coding Friction** | Extremely low | Extremely high (state machine abstraction) | **Low** (simple KV interface) |
| **Power-Loss Protection** | ❌ None | ✅ WAL + RaftLog | **✅ WAL** |
| **Scale-to-Zero** | ❌ None (100k booths blow up memory) | ✅ LSM-Tree | **✅ LSM-Tree** |
| **Persistence** | ❌ None (state lost after restart) | ✅ (metadata Raft + data Fjall/S3) | **✅ Fjall on disk** |
| **Future Scalability** | ✅ Unlimited | ⚠️ Whole-domain consensus binding (under federated semantics this is a directional regression) | **✅ Unlimited** |
| **Mental Load** | Extremely low | Medium (consensus dependency) | **Low** |
| **Recommended Scenario** | Prototype validation | When data-level strongly consistent replication is definitely needed (use TiKV etc. directly) | **General starting pattern** ✨ |

**Mode One: Mutex<HashMap> start (zero-dependency experience)**—when the project has just typed its first line of code, state is a native Rust type, and no serialization macros need to be written. Fatal flaws: no power-loss protection, no Scale-to-Zero, no persistence.

**Mode Two: External consensus start (metadata coordination shackles)**—introducing consensus (Openraft or etcd) for cross-node coordination, even if the use is narrowed to metadata (booth registration/routing/configuration) without replicating booth data, still binds the coordination logic to Raft's consensus dependency. This architecture has explicitly decided **not to introduce consensus**: metadata is independent per node (the control plane is single-writer, and the `ctx.metadata` read-only face keeps the writer permanently single), and multiple machines take the federated form—there is no path to consensus inside the federation, and multi-control-plane deployment is a directional regression rather than a scaling point. Consensus is borne only when "data-level strongly consistent replication" genuinely appears, by a ready-made external solution (TiKV etc.).

**Mode Three: Tokio booth + single-machine Fjall start (the highest compromise in engineering)✨**—replacing HashMap with Fjall adds almost no coding friction, yet brings: ✅ local bare-metal-level physical protection against power-loss crashes (WAL), ✅ automatic memory zeroing when idle (Scale-to-Zero), ✅ read/write speeds up to the limit of the physical hardware, ✅ microsecond-level positioning via Bloom filters. If the project grows and needs a multi-machine form, since it is already a booth + Fjall architecture, it can evolve directly into the federated mode (nodes independent + well-known protocol authentication, data still local + lake/S3), with no need to introduce consensus. **The starting style does not lock in the ultimate destiny.**

### 8.3 Starting-Style Code Examples

#### Mutex<HashMap> Start (Zero Friction)

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use tokio::sync::mpsc;

struct BoothState {
    data: Mutex<HashMap<String, Vec<u8>>>,
}

impl BoothState {
    fn new() -> Self {
        Self {
            data: Mutex::new(HashMap::new()),
        }
    }

    fn upsert(&self, key: &str, value: Vec<u8>) {
        self.data.lock().unwrap().insert(key.to_string(), value);
    }

    fn get(&self, key: &str) -> Option<Vec<u8>> {
        self.data.lock().unwrap().get(key).cloned()
    }
}
```

#### Tokio Booth + Fjall Start (Best Compromise)✨

```rust
use fjall::{Config, Keyspace, PartitionCreateOptions};
use std::sync::Arc;

struct BoothState {
    keyspace: Keyspace,
    partition: Arc<fjall::PartitionHandle>,
}

impl BoothState {
    fn new(path: &str) -> anyhow::Result<Self> {
        let keyspace = Config::new(path).open()?;
        let partition = keyspace.open_partition(
            "booth_state",
            PartitionCreateOptions::default(),
        )?;

        Ok(Self {
            keyspace,
            partition: Arc::new(partition),
        })
    }

    fn upsert(&self, key: &str, value: Vec<u8>) -> anyhow::Result<()> {
        self.partition.insert(key.as_bytes(), value)?;
        self.keyspace.persist(fjall::PersistMode::SyncAll)?;
        Ok(())
    }

    fn get(&self, key: &str) -> anyhow::Result<Option<Vec<u8>>> {
        Ok(self.partition.get(key.as_bytes())?.map(|v| v.to_vec()))
    }
}
```

#### Metadata Evolution: Independent Per Node, Federated Authentication

```rust
// Multi-machine form: metadata is independent per node (the meta table inside each data-plane okm instance, control plane single-writer), data written directly to local Fjall (then to the lake)
// No consensus across nodes—federated nodes authenticate identity via the well-known protocol, and user data follows its owning node

struct DistributedBoothState {
    meta: Arc<MetaStore>,               // Metadata: booth registration/routing (this node's instance)
    booth: Arc<BoothState>,             // Reuses the single-machine Fjall implementation above
    store: Option<ObjectStoreClient>,   // Data fallback: lake / SlateDB+S3
}

impl DistributedBoothState {
    async fn upsert(&self, key: &str, value: Vec<u8>) -> anyhow::Result<()> {
        // Data written directly to local Fjall; lake / S3 as async fallback—no consensus write path
        self.booth.upsert(key, value).await?;
        Ok(())
    }

    async fn locate(&self, booth: &str) -> anyhow::Result<NodeId> {
        // Metadata read on this node (control plane single-writer + node cache)
        self.meta.locate(booth).await
    }
}
```

### 8.4 The Victory of Technical Realism

A state management framework is only a tool; it is not a cage that imprisons a business's destiny. On a project's first day, directly introducing heavy logic such as distributed network consensus often drags down early business evolution speed, killed by over-engineering.

But if you adopt **"the lightest Tokio booth message pipeline + single-machine embedded Fjall"** as the general starting core for new projects—this provides both startup speed, memory safety and hibernation efficiency, and leaves wide open the door to a later unified evolution toward all external databases (Postgres, S3, LanceDB). Running on a single machine is perfectly fine; it is the most robust, most clear-headed, low-overhead first step for this architecture toward industrialization.

---


## 9. Industrial Applicability Diagnosis and General Scenarios

This pure-Rust compute-storage integrated architecture, forged from Fjall (local storage) + an independent metadata instance (single-writer, no consensus) + a multimodal embedded kernel (Steel/PyO3/Wasm user-driven sandbox), has extremely strong universality in industrial deployment. It is compatible not only with gateway, FaaS, task platform and game scenarios, but can even trigger architectural disruption in these scenarios.

### 9.1 Engineering Applicability Diagnosis of Four Core Scenarios

#### API Gateway Scenario: Suitable (achieving "dynamic policies, zero network overhead")

Traditional API gateways (such as Kong or APISIX), when handling advanced routing (e.g. dynamic rate limiting, AB test traffic splitting, user authentication), usually must frequently read external caches, or stuff Nginx full of obscure Lua scripts.

**Under this architecture**: Every API routing policy or every tenant is a distributed booth in the cluster. After the gateway receives a request, it needs no network hops at all; it directly pulls the latest rule for that route from process memory via Fjall, and a Steel Lisp sandbox dynamically specified by the user executes it in a second.

**Key point**: Because metadata is independent per node (control plane single-writer + node cache), after you modify this node's gateway rules they take effect immediately, with no risk of external cache single-threaded deadlock or cluster split-brain.

#### FaaS Platform Scenario: Complementary (curing the fatal flaw of "stateful Serverless")

The most fatal pain point of traditional FaaS (such as AWS Lambda or Vercel Functions) is "statelessness (Stateless)". Every time a function is spun up it must freshly connect to the database, handshake, and look up configuration (cold start lasting hundreds of milliseconds), which has given rise in the industry to a strong desire for something like Cloudflare Durable Objects (stateful persistent objects).

**Under this architecture**: You do not need to bundle a heavy, bloated V8 and JS ecosystem the way Rivet does. The user decides for themselves whether this millisecond runs FaaS with Steel or Python.

**Key point**: It provides Scale-to-Zero (memory zeroed when idle) and low system startup overhead, after which booth wakeup is microsecond-level (embedded VM created instantaneously). When a function is inactive, its state hibernates in Fjall disk in the form of compact SSTables, occupying not one byte of RAM; when a request hits, Fjall instantly slaps the Vec binary blob into memory, and the multi-language engine revives in place to execute.

#### Task Orchestration Platform (such as Windmill/Prefect): Surpassing (removing scheduling queue bloat)

Task platforms like Windmill must, at their low level, depend heavily on a high-throughput PostgreSQL or external queue to maintain task scheduling queues, concurrency locks, and retry state. When confronted with million-level micro-task concurrency, the external database connection pool is instantly exhausted.

**Under this architecture**: Each long-running streaming task (Workflows) is itself an independent booth. Every Step of a task and every step's retry state are atomically written into the local physical Fjall engine.

**Key point**: It completely eliminates the external scheduling queue. Task state lives in the compute engine's intimate process memory, requiring no cross-process IPC and no external locks. It is lighter and faster than Windmill, and naturally possesses cross-datacenter anti-amnesia and anti-sudden-death (Fault-tolerance) capability.

#### Game Server Scenario: Its Original Trade (the Rust return of game-grade high-concurrency booths)

Why did Rivet Actors use this design? Because they originally came from hosting multiplayer online game rooms (Game Rooms) and real-time player state (Player Stateful).

**Under this architecture**: You implement, in pure Rust, a native game runtime that is purer and cleaner than Rivet's.

**Key point**: One game room is one booth; all of the player's movement, health, and equipment changes (State Mutate) are cached directly at high frequency (60 times per second) in that booth's memory, persisted on every write to Fjall WAL (same-room state is not replicated across nodes; a room belongs to a fixed node—demands for strongly consistent replication are handed to external TiKV-class solutions). When the game ends, Fjall asynchronously performs one large physical flush to disk. Because JS/V8 garbage collection (GC) overhead is completely kicked out, the main program can run at the smoothness of over a thousand frames, and a single machine hosting tens of thousands of player room systems will never stutter.

### 9.2 Three Brand-New Hardcore Application Scenarios

Besides the four classic domains above, this full-stack architecture of Fjall + independent metadata + user-driven multi-language sandbox can also demonstrate applicability in the following three environments involving general, cutting-edge development:

#### Distributed Industrial IoT and Edge Computing (Edge AI & IoT Gateways)

In wind farms, connected vehicles, drone orchestration, or automated factory server rooms, hardware devices are often in harsh physical environments of "weak network, low power consumption, local disk where every inch of space is precious". You cannot stuff a huge K8s cluster or a PostgreSQL database into an edge server room.

**How to play it**: Throw this single-file binary directly onto an edge gateway (such as a Raspberry Pi or an industrial single-board computer). Because Fjall's LSM structure is extremely resistant to power-loss wear, edge nodes each stay independent and autonomous (metadata single-writer locally, well-known protocol authentication between federated parties). When high-frequency sensor data is collected, PyO3 Python runs Numpy anomaly detection directly in local memory; when a crisis is detected, it immediately switches to Steel Lisp to run a deterministic shutdown policy script. Microsecond-level response, with no need to connect to the cloud at all.

#### Modern Web 3.0 / Consortium Chains and Trusted Decentralized Ledgers (Consensus Ledgers)

Traditional blockchain low levels (such as Ethereum nodes) are architecturally unwieldy when executing smart contracts. For ledger scenarios requiring whole-network consensus, the consensus layer directly adopts a mature solution (handing majority consensus to a dedicated blockchain/ledger engine), and the contract logic is written as Steel Lisp or Wasm bytecode executed on this architecture; with a Rust core of under 2000 lines, it implements an ultra-high-performance dedicated decentralized state ledger whose response speed breaks past tens of thousands of TPS and which has cross-border multi-datacenter disaster-recovery capability.

#### Enterprise Private "Headless" AI Programmer Agent Matrix (Headless Coding Matrix)

When a company needs to deploy 1000 ubiquitous "AI programmers" that automatically review Git code, run automated tests and fix bugs in the background 24/7, each AI programmer is a resident or hibernating distributed booth. When the AI needs to crawl API documentation from the external network, it calls the Python (PyO3) async network library; when it needs to run PyTorch weights or semantic analysis against local files, it calls PyO3 Python; when it needs to execute dangerous local compilation, it is locked firmly inside a Wasm sandbox to prevent it from mistakenly deleting the real disk. All of its thinking logic, progress, and pitfall-avoidance memos need no dependence on an external queue at all, and are persisted directly in the local Fjall database.

### 9.3 The Architect's Core Advantage

The reason this architecture can demonstrate applicability simultaneously across so many divergent general and extreme scenarios is not that it has many features; on the contrary, precisely because it removes every intermediary that should not exist (The Best Part is No Part).

At the physical level it eliminates serialization and deserialization at both ends of the network cable, fitting multi-core CPUs, NVMe physical disks, and the memory space of multi-language VMs directly into the same rack position. In a general development environment, this is a concise design orientation.

---


## 10. Applicability to Conventional Web Services

Aura's core scenario is high-concurrency stateful streams (games, IM, flash sales, AI agents). For conventional Web services (CRUD, e-commerce, enterprise management systems), it is necessary to judge which sub-scenarios suit Aura and which should fall back to a traditional relational database.

### 10.1 Data Mapping

Conventional Web business data maps by characteristic onto Aura's dual-track memory:

| Business Data Type | Traditional Approach | Aura Mapping |
|---|---|---|
| High-frequency state transactions (shopping cart, inventory, Session) | Redis | Working memory (SlateDB+S3): in-booth-process KV, S3 auto-replication, 0 network RTT |
| Stream events (order history, audit logs) | Log table or Kafka | Fjall WAL local persistence (under compute-storage separation, S3 bears long-term safety) |
| Massive historical read-only analytics (reports, full-text search) | Read-write splitting, Elasticsearch | Long-term memory (LanceDB + S3): window close converts to columnar Lance, S3-managed |

### 10.2 Suitable Sub-Scenarios

- **High-concurrency state changes** (flash sales, order grabbing): MPSC queuing inside the booth process; a single machine carries concurrency that traditional architectures can only bear with multiple machines plus external caches
- **Diskless elastic deployment**: working memory Scale-to-Zero, long-term memory managed on S3, nodes can be spun up or destroyed at any time

### 10.3 Unsuitable Sub-Scenarios

- **Ad-hoc SQL queries over arbitrary dimensions**: hot data is Fjall KV blobs, cold data is S3 columnar slices; multi-table JOIN + GROUP BY cannot be executed. Note: [Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md) has partially solved this problem
- **Back-office management / ERP / CRM**: 80% of the work is tabular report join queries
- **Team threshold**: Bincode serialization and embedded script sandboxes are unfriendly to junior developers (consensus is no longer inside the architecture, so it is no longer a threshold)

### 10.4 Hybrid Architecture

When a conventional Web service contains both "high-concurrency core modules" and "conventional CRUD", adopt a hybrid:
- Strip CRUD out to an external database (e.g. if relational queries are needed)
- Use Aura for the high-concurrency state engine

Do not let technical sentiment become a stumbling block for business iteration.

---

## References

- [1] Research on the Data Consistency Avalanche Caused by Blind NoSQL Fanaticism and Multi-Database Federation (2025/2026 technical survey)
- [2] Throughput Benchmark of Modern High-Speed Multi-Core CPUs and RDBMS Shared Memory Buffers (2026/03)
- [3] InfoQ 2026/05: Google Open-Sources AX and Agent Substrate to Build a K8s Control Plane Dedicated to Agents

---


## Cross-References

This document is the ultimate landing solution for the architecture design, forming a complete decision loop with the following detailed analyses:

- **[Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md)**: Full-chain compute-storage integration of Fjall + Arrow + Polars, including the collision of the underlying byte layouts of 7 data formats.
- **[Aura + Fluxora DevOps](aura-fluxora-devops-en.md)**: The engineering practice of Script-as-Code, including GitOps workflows, CI/CD configuration, script testing and debugging.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: A detailed argument for not using Redis; §9.6 is the summary version.
- **[MySQL Critique](mysql-critique-en.md)**: MySQL's SQL anti-patterns and the sharding illusion; this architecture chooses embedded KV.
- **[Unified Data Layer](unified-data-layer-en.md)**: An alternative to modern multi-model databases; this architecture may choose SurrealDB as a relational supplement.
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: A technical comparison of Rune/Steel/Koto; this architecture firmly chooses Steel Lisp.
- **[Editor Selection Analysis](editor-selection-2026-en.md)**: An in-depth analysis of Helix vs Neovim; this architecture can use Fennel/Steel metaprogramming in Neovim.
- **[Nginx Critique](nginx-critique-en.md)**: Nginx's legacy design; this architecture may choose Envoy/Pingora as a modern gateway.
- **[Flux Architecture](flux-architecture-en.md)**: Aura §5's field model is the in-process implementation of reactive architecture—booth writes to Fjall → emit event → subscriber receives in-process, zero network hops, no dependence on an external stream processing engine.
- **[WASM Unified Runtime](wasm-unified-runtime-en.md)**: Rust → Wasm is now usable (no wasm-gc needed), covering async scenarios. After wasm-gc, Kotlin/C# will pour in, further enriching language choice. Steel and PyO3 are retained.