# Fluxora Architecture

> **Languages:** [English](fluxora-architecture-en.md) (primary) · [中文](fluxora-architecture.md)

**Path**: `~/world/fluxora/`  
**Type**: Event-driven, AI-native UI framework (Rust/Dioxus)  
**Status**: Maintenance only (periodic dependency updates). The strategic focus has shifted to graph memory.

## Core Concepts

Fluxora is not a traditional web framework. It is an **event-driven UI bus** that decouples business logic from presentation through the Kafka/Iggy message queue. The UI is described by a declarative JSON DSL (`Brick`) rather than by code.

## Architecture Topology

```
┌──────────┐   WS    ┌──────────┐  outgo   ┌─────────────────┐
│   UI     │◄───────►│ Gateway  │─────────►│ Business         │
│ (Dioxus) │         │ (Axum)   │          │ (Chat, CRM, AI…)│
└──────────┘         └────┬─────┘          └────────┬────────┘
                          │   income                │
                          └─────────────────────────┘
```

- **UI (Dioxus/WASM)**: renders the `Brick` JSON tree. Components are bound to event names—no API URL, no HTTP verb.
- **Gateway (Axum)**: WebSocket router. Dispatches UI events to Kafka `outgo`; delivers backend `income` to the correct WS session. Handles the greet hook, webhook interception, and template rendering.
- **Business services**: independent Kafka consumers. Each service (chat, crm, echo, analysis…) handles its own logic. Services are **transparent** to one another—analysis can intercept the chat stream without chat knowing.

## Project Structure

```
fluxora/
├── crates/brick/         # Core JSON DSL—typed Brick enum, attributes, bindings
├── crates/brick_macro/   # Derive macros for BrickOps, ClassifyBrick, render hints
├── crates/message/       # Unified message protocol, Event trait, Kafka/Iggy, Codec
├── crates/content/       # Content actions: Create, Set, Join, Tmpl, Empty
├── crates/gateway/       # WS router + minijinja + webhooks + session management
├── crates/ui/            # Dioxus/WASM frontend—Frame renderer, WS store, merge
├── crates/ui_macro/      # UI component derive macros
├── crates/chat/          # Demo business service (channel-based chat)
└── crates/agent/         # Agent service skeleton
```

## Core Principles

1. **AI-native, not AI-dependent**: the AI generates structured JSON (the Brick DSL) validated by a Schema. Fixed business uses templates for performance; the AI is used for dynamic/exploratory UI.
2. **Event sourcing**: services are decoupled through Kafka/Iggy (the `outgo`/`income` queues). Services transparently intercept streams without each other's knowledge.
3. **Declarative binding**: components are bound to event names, not API URLs. Input=POST, Display=GET semantics. The Gateway maps event names to Kafka topics.
4. **Protocol-agnostic**: a three-layer codec strategy (see below). CBOR is the default binary codec; bincode has been removed. Postcard can serve as a lightweight alternative, but must be paired with the Externally Tagged enum pattern (see [ADR-001 supplement](fluxora-decisions-en.md#adr-001-cbor-over-bincode)).

---

## Why Let AI Generate JSON Instead of HTML

In traditional web development, the AI outputs HTML/CSS/JS markup directly. This has three fatal problems:

1. **Bloat** — a page's HTML/JS/CSS is far longer than the body content; the AI has to generate a large amount of boilerplate markup (tag closing, class naming, nesting depth), at a high token cost
2. **Unstable output** — for the same prompt, the style details differ every time (class names, spacing, nesting structure); superficially similar but with significant differences in detail. HTML is a visual artifact, so data correctness can only be checked by headless tests or the naked eye
3. **Hard to verify** — HTML has no schema, so whether the AI's output is right cannot be checked at compile time

The JSON-render idea is the reverse: **the AI generates only structured JSON (data + component type declarations) and makes no styling decisions**. The framework is responsible for rendering the JSON into a fixed-style UI.

| Dimension | AI-generated HTML | AI-generated JSON (Fluxora Brick) |
|------|-------------|-------------------------------|
| Token cost | High (lots of markup boilerplate) | Low (JSON is an order of magnitude smaller than HTML) |
| Output stability | Different every time (style drift) | Deterministic (styles locked down by the framework) |
| Data validation | No schema, visual inspection | JsonSchema compile-time validation |
| AI responsibility | Content + structure + style (the whole package) | Content and structure only (style left to the framework) |

**In essence, this takes "creative decisions" (style) out of the AI's hands and lets the AI do only what it is good at (content and structure).** Anthropic has recently been pushing HTML over Markdown as an AI output format; that direction is backwards—the heavier HTML becomes, the less stable the AI's output. JSON-render keeps the AI's output as light and as structured as possible and hands the rendering work to deterministic code.

---

## The `brick` Crate (UI DSL)

**Location**: `crates/brick/src/lib.rs`

### The Brick Enum

- `#[serde(tag = "type")]` — zero-ambiguity JSON serialization. The AI outputs `{"type": "button", ...}` → automatically deserialized to `Brick::Button`.
- A recursive tree via `sub: Option<Vec<Brick>>`.
- List items via `item: Option<Vec<Brick>>` (used by `Rack`, `Fold`).
- Template instantiation via `Render { name, data }`.
- All structs derive: `Serialize, Deserialize, JsonSchema, BrickOps`.

### The Bind System

```json
{
  "bind": {
    "input": { "event": "chat.send", "type": "text" },
    "output": { "source": "chat.msg" }
  }
}
```

- **`source`**: listens to an event stream (the GET equivalent).
- **`event`**: emits an event (the POST equivalent).
- **`target` / `field` / `submit`**: advanced binding modes.

No API URL. The Gateway automatically maps event names to Kafka topics.

### Streaming Merge (`crates/brick/src/merge.rs`)

Three merge strategies are used for localized path updates:

- **`Replace`**: overwrite the value.
- **`Concat`**: append (text concatenation, number addition, array push, object merge).
- **`Delete`**: remove (string replacement, number subtraction, object key removal).

**Key**: merge is **relative to the data source a component is bound to**, not global. Components are matched and merged by `id`. Multiple services stream to the same page without conflict.

### Macros (`brick_macro`)

- `BrickOps`: automatically generates `get_type()`, `borrow_sub()`, `get_bind()`, and the like.
- `ClassifyBrick` / `ClassifyAttrs`: classification support.
- `render_brick`: render hints.

---

## Component System Design Trade-offs

### A Closed Component Set: Why Extension Is Not Allowed

The component set is **bounded and closed**, and free extension is deliberately not allowed. Three constraints decide this together:

1. **AI reliability** — the component set is the "vocabulary" from which the AI generates JSON. If the vocabulary grows without bound, the AI cannot reliably output correct components. A closed set = the AI can learn it exhaustively
2. **WASM size** — every component increases bundle size. The number of components directly affects user load speed
3. **The enum as gatekeeper** — the `Brick` enum is the explicit boundary of the closed set. Adding a component must pass through the enum gate by hand, and that friction is deliberate

The boundary may be adjusted (components replaced), but the total will not change much.

### Why an Enum Rather than Runtime Registration

The `inventory`/`linkme` runtime registration approach (each component self-registers, a central registry looks up by name) does not apply:

1. **WASM compatibility** — `inventory` relies on linker behavior to collect registration items, and under the WASM target registrations may be silently lost
2. **Type erasure** — runtime registration requires unifying on a function-pointer signature, which destroys the compile-time type safety of Dioxus `rsx!`
3. **Loss of compile-time safety** — a wrong component function signature or a missed registration surfaces only at runtime

### Why Not Replace the Enum with a Macro DSL

A unified declarative-macro definition (one macro call generating both the enum and the dispatch) does not apply:

1. **Centralized audit** — the enum is a catalog of all components; at review the whole picture is visible at a glance. A macro DSL scatters the definitions into the invocation syntax, losing centralized control
2. **Module exclusion** — disabling a component = deleting one line from the enum. The macro approach needs an extra mechanism to control inclusion

### The Trade-offs of `gen_dispatch!`

A procedural macro automatically generates the match dispatch from the Brick enum AST. Every choice has an explicit reason:

| Choice | Alternative | Reason not adopted |
|------|---------|-----------|
| `include_bytes!` to track the file | `build.rs` + `cargo:rerun-if-changed` | Avoids a separate process, which would break incremental compilation |
| A hand-written `syn` AST walk | `darling` / `strum` | The attributes are just a few key=value pairs, zero extra dependencies |
| `Span::call_site()` errors | `Span::mixed_site()` | No external users, so there is no debugging scenario |
| Keeping the Brick enum | `inventory` runtime registration / trait object | One line of registration in exchange for zero-cost serde + compile-time exhaustiveness + WASM compatibility |

### An Open Component System Is Not Feasible Under Rust/WASM

If the requirement were open components (adding a component without changing the central file), the options are:

- **`inventory` runtime registration** — linker behavior is unreliable under WASM
- **`typetag` trait object** — depends on `inventory`, the same WASM problem + vtable overhead
- **A declarative macro DSL** — still centralized definition in essence, not truly open

Rust's type system and the WASM compilation model naturally push you toward a closed set. JavaScript can be open because of runtime dynamism. Rust determines everything at compile time, and the infrastructure for open registration is unreliable under WASM. A closed enum is not a compromise; it is the natural solution under the constraint.

---

## Backend: Event Sourcing + Message Queue

### Message Protocol (`crates/message/`)

- `Envelope<Created>`: `{ receiver: Vec<Session>, message: ChatMessage }`
- `ChatMessage<Created>`: `{ sender, created, content: Value }`
- The `Event<C>` trait: `fn event(&self) -> Option<&str>`
- Both **Kafka** (`rdkafka`) and **Iggy** (`iggy`) backends are supported.

### Content Actions (`crates/content/`)

- `Create`: sets the root layout Brick.
- `Set`: sets a named data Brick (keyed by event name).
- `Join`: merges into a list (supports the Replace/Concat/Delete methods).
- `Tmpl`: registers a template (name + minijinja string).
- `Empty`: no-op.

### Gateway (`crates/gateway/`)

- **WebSocket handler**: session management, the greet hook, and event routing to `outgo_tx`.
- **Template engine**: `minijinja` for Gateway-layer template rendering.
- **Webhooks**: intercept specific events (such as login) for special handling. They can call an external endpoint or apply a local template.
- **Session dispatch**: `send_to_ws()` listens on `income_rx` and routes by the `receiver` field.

### UI Store (`crates/ui/src/libs/store.rs`)

- `Status`: holds `layout: Signal<Brick>`, `data: Signal<HashMap<String, Brick>>`, `list: Signal<HashMap<String, Vec<Brick>>>`.
- `dispatch()`: handles incoming `Message<Brick>` actions (Create, Set, Join, Tmpl).
- Template rendering goes through the global `TMPL: LazyLock<RwLock<Environment>>`.
- Streaming merge: the `Join` action uses the `BrickOp` trait (Replace/Concat/Delete) with `id`-based matching.

---

## Codec Architecture

### Why Not `dyn Codec`?

A Rust trait with generic methods **is not object-safe**:

```rust
// This will fail:
pub trait Codec {
    fn encode<T: Serialize>(&self, value: &T) -> Result<Vec<u8>>;  // generic T!
}
pub fn build_codec() -> Box<dyn Codec> { ... }  // Error: the trait is not dyn-compatible
```

### Solution: Enum Dispatch (`ActiveCodec`)

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, serde::Deserialize)]
pub enum CodecType {
    Json,
    #[default]
    Cbor,
}

#[derive(Debug, Clone)]
pub enum ActiveCodec {
    Json,
    Cbor,
}

impl ActiveCodec {
    pub fn new(t: CodecType) -> Self {
        match t {
            CodecType::Json => Self::Json,
            CodecType::Cbor => Self::Cbor,
        }
    }

    pub fn as_type(&self) -> CodecType {
        match self {
            Self::Json => CodecType::Json,
            Self::Cbor => CodecType::Cbor,
        }
    }

    pub fn encode<T: Serialize>(&self, value: &T) -> Result<Vec<u8>, CodecError> {
        match self {
            Self::Json => serde_json::to_vec(value).map_err(|e| CodecError::Encode(e.to_string())),
            Self::Cbor => {
                let mut out = Vec::new();
                ciborium::ser::into_writer(value, &mut out)
                    .map_err(|e| CodecError::Encode(e.to_string()))?;
                Ok(out)
            }
        }
    }

    pub fn decode<T: DeserializeOwned>(&self, bytes: &[u8]) -> Result<T, CodecError> {
        match self {
            Self::Json => serde_json::from_slice(bytes).map_err(|e| CodecError::Decode(e.to_string())),
            Self::Cbor => {
                ciborium::de::from_reader(&mut std::io::Cursor::new(bytes))
                    .map_err(|e| CodecError::Decode(e.to_string()))
            }
        }
    }
}
```

Zero vtable overhead, fully Clone-able for use in async tasks.

### Why Bincode Is Rejected

- **Root cause**: Bincode **cannot deserialize `#[serde(tag = "...")]` enums**—it calls `deserialize_any`, which bincode does not support. This affects all deserialization of the `Brick`/`Content` types, not just the wire protocol.
- serde 2.x incompatibility (v1.x broken, v2.x API unstable)
- No type self-description (the Gateway cannot partially parse routing metadata)
- No cross-language support (bincode has no JS library)

### Why CBOR Rather than MessagePack?

- CBOR is an IETF standard (RFC 8949), with better type tags and deterministic length prefixes, and it supports partial parsing/lookup.
- A mature JS ecosystem (`cbor-x`), which is crucial for future frontend integration.
- Self-describing—for WS and Kafka workloads, the slight size/speed trade-off is acceptable.

### Three-Layer Codec Strategy

| Layer | Strategy | Reason |
|----|------|------|
| **Gateway WS** | Pinned at handshake from the URL parameter `?codec=json\|cbor` | The URL parameter is read at WS upgrade. Greet is sent immediately with the correct codec. Zero RTT. |
| **Kafka queue** | Hard-coded CBOR | All queue traffic uses CBOR. Producer: `into_writer`; consumer: `from_reader`. If multilingual consumers appear, Avro/Schema Registry is recorded as a future possibility. |
| **UI (WASM)** | Default CBOR, overridable by query parameter | `Status` stores the `ActiveCodec`, and `send()` chooses Text vs Bytes frames. |

### Why Not a Single Global Configurable Codec?

A unified codec setting is simpler on paper but fails in practice:

- The **Gateway** serves heterogeneous clients. The URL parameter provides per-connection control without configuration burden.
- **Kafka** stores durable state. Always CBOR—no configuration. CBOR's self-description means debugging is still possible. If multilingual consumers appear, Avro is recorded as a future option.
- The **UI** is the user-facing layer. The default should be the most efficient option (CBOR), with a debugging escape hatch.

### Gateway: Pinning the Codec at Handshake (URL Parameter)

The codec is parsed at WS upgrade—**zero extra RTT**. The codec is **pinned at handshake**—no dynamic detection, no first-frame adaptation.

**Handshake flow iterations** (earlier approaches were iterated on and refined):

1. ~~Greet sent immediately with a fixed JSON codec~~ — a codec mismatch if the client expects CBOR
2. ~~The client sends a `client_info` first frame, then greet~~ — adds 1 RTT before the UI renders
3. **Current: the URL `?codec=json|cbor` is parsed at WS upgrade** — zero extra RTT, greet is sent immediately with the correct codec

```rust
// main.rs — the route handler reads the codec from the URL
let mut codec = codec_for_router.clone();
if let Some(codec_str) = q.get("codec").and_then(|v| v.as_str()) {
    if let Ok(ct) = codec_str.parse::<CodecType>() {
        codec = ActiveCodec::new(ct);
    }
}
ws.on_upgrade(async move |socket| {
    handle_ws(socket, tx, state, config, tmpls.clone(), codec, &a).await;
});
```

**Handshake flow (zero RTT)**:
1. The client opens a WS connection (`/channel?codec=json&token=xxx`)
2. The Gateway reads the codec from the URL → sends greet immediately (correct frame type)
3. The client receives greet → renders the base UI
4. All subsequent messages use the same codec (pinned for the connection's lifetime)

**The `encode_ws()` helper**:

```rust
// shared.rs — CodecType lives directly on Client (CodecHint was removed as redundant)
pub struct Client<T> {
    pub sender: T,
    pub codec: CodecType,  // pinned from the URL parameter at handshake
}

pub fn encode_ws<T: Serialize>(codec: CodecType, value: &T) -> Option<axum::extract::ws::Message> {
    let bytes = ActiveCodec::new(codec).encode(value).ok()?;
    Some(match codec {
        CodecType::Json => axum::extract::ws::Message::Text(String::from_utf8(bytes).unwrap().into()),
        CodecType::Cbor => axum::extract::ws::Message::Binary(bytes.into()),
    })
}
```

**The router parses the codec from the URL**:
```rust
let mut codec = config_codec.clone();
if let Some(cs) = q.get("codec").and_then(|v| v.as_str()) {
    if let Ok(ct) = cs.parse::<CodecType>() { codec = ActiveCodec::new(ct); }
}
ws.on_upgrade(async move |socket| {
    handle_ws(socket, tx, state, config, tmpls.clone(), codec, &a).await;
});
```

### Why Remove `CodecHint`

`CodecHint { Json, Cbor }` is structurally identical to `CodecType { Json, Cbor }`, with a `From` conversion. It duplicated the `ActiveCodec::encode()` logic—`CodecHint::encode()` was merely "encode bytes → wrap in a Text/Binary frame". **Pattern: do not create parallel enum types that map 1:1.** Use `CodecType` + the `encode_ws()` helper directly.

### UI (WASM)

The UI respects the `?codec=json` or `?codec=cbor` URL parameter for both sending and receiving. The `ActiveCodec` is stored in `Status` and used by `send()` to decide the frame type:

```rust
// store.rs — Status::send respects the codec
if let Ok(buf) = self.codec.encode(&outflow) {
    let msg = match &self.codec {
        ActiveCodec::Json => gloo_net::websocket::Message::Text(String::from_utf8(buf).unwrap()),
        ActiveCodec::Cbor => gloo_net::websocket::Message::Bytes(buf),
    };
    self.ws.send(msg).await;
}
```

**Note**: `gloo_net` uses `Message::Bytes`, not `Message::Binary`.

### WASM Dependency Isolation

When `ui` depends on `message`, disable default features to avoid WASM-incompatible native dependencies:

```toml
# crates/ui/Cargo.toml
message = { path = "../message", default-features = false }
```

### Kafka Queue

Kafka uses **hard-coded CBOR**—no configuration needed.

The producer encodes CBOR binary, and the consumer decodes through `ciborium::de::from_reader`.

**Migration** (2026-05-08, Bincode → CBOR):

| File | Change |
|------|---------|
| `crates/message/src/codec.rs` | Removed the `CodecType::Bincode` and `ActiveCodec::Bincode` variants |
| `Cargo.toml` (workspace) | Removed `bincode = "1.3.3"` |
| `crates/message/Cargo.toml` | Removed `bincode.workspace = true` |
| `crates/gateway/Cargo.toml` | Removed `bincode.workspace = true` |
| `crates/ui/Cargo.toml` | Added `ciborium.workspace = true` |
| `crates/ui/src/libs/store.rs` | `Status::send()` → CBOR encoding + `gloo_net::websocket::Message::Bytes` |
| `crates/message/src/kafka/mod.rs` | Producer: `into_writer`; consumer: `from_reader` replacing `serde_json::from_str` |

**The Kafka producer sends raw bytes**:
```rust
let mut buf = Vec::new();
into_writer(&value, &mut buf)?;
producer.send(FutureRecord::to(&topic).payload(&buf), ...);
```

**The Kafka consumer payload is binary, not a string**:
```rust
// Old: payload_view::<str>() + serde_json::from_str
let payload = msg.payload().unwrap();
let mut cursor = std::io::Cursor::new(payload);
let value: T = ciborium::de::from_reader::<T, _>(&mut cursor)?;
```

### Performance Notes

- Enum dispatch = compile-time match, zero vtable overhead.
- `Clone` on the codec enum is cheap (unit variants).
- CBOR is the default—close to bincode in performance, with IETF-standard type self-description and cross-language support.
- JSON is used for debugging and AI-generated content.

---

## Streaming Merge Strategy: `Vec<String>` Fragment Buffering

### Why `Vec<String>` Rather than `&[&str]` or `String`

Three approaches were evaluated for accumulating streaming text fragments:

| Approach | Problem | Status |
|------|------|------|
| `&[&str]` | Lifetime hell + **memory amplification** — the reference pins the full original fragment (envelope + payload) in memory; the envelope cannot be dropped until rendering is complete. Amplification factor = 1 / payload_ratio (e.g. 20% payload → 5× memory) | Rejected |
| `String` + `push_str` | Grows with the buffer; each token may cost an O(N) reallocation + memcpy | Rejected |
| `Vec<String>` + `push` | O(1) amortized append; only N × 8 bytes of pointer overhead; the payload can be extracted immediately and the envelope dropped | **Accepted** |

`Vec<String>` and `String` store essentially the same content—the `Vec` merely adds one pointer (8 bytes) per fragment. The allocation pattern is strictly better: amortized O(1) per token vs. `push_str`'s possible O(N) reallocation per token.

### Render Strategy

The render component will eventually be redesigned to consume `Vec<String>` directly, iterating the fragments without pre-concatenating. This even avoids the single final allocation. Deferred for now.

### Protocol Overhead

Each fragment carries the complete JSON envelope (metadata, routing), of which the text payload is a small part. Storing a full `String` per fragment means memory scales with the envelope size, not just the payload. This is an acceptable cost of the decoupled architecture. If it becomes a bottleneck, extract and store only the payload after routing.

### Key Principles

- **Mandatory deserialization**: every message (even a single token) must be deserialized to locate the correct UI tree node.
- **Frequent merge**: merge per token to achieve a smooth "typing effect". Higher CPU cost than "batch copy" but prevents browser stutter.

---

## Component Atomicity: Table and SVG

- **Direct mapping**: these map directly to standard HTML/SVG elements.
- **AI alignment**: LLMs are heavily trained on standard HTML/SVG. Keeping them atomic ensures high generation accuracy.
- **No over-abstraction**: semantic wrappers are unnecessary and hinder flexibility.

## Concurrency: the Synchronous Template Engine

- **CPU-bound**: `minijinja` rendering is pure computation, no IO. The blocking time is negligible (microseconds).
- **No need for spawn_blocking**: an async wrapper adds overhead to lightweight text processing.
- **Reserved for scale**: async offloading is needed only in high-concurrency/heavy-template scenarios.

## Design Time vs Runtime

### Design Time (AI-assisted)

- The AI generates Brick JSON + mock data → the developer previews in the UI → saves it as a template.
- The AI acts as a "super frontend engineer"—lowering the barrier to writing JSON components.

### Runtime (template-driven)

- A business service returns pure data → the Gateway applies the template → rendering at millisecond speed.
- Zero LLM latency. Deterministic output. No sensitive data is fed to the LLM.

### Exploratory (AI-generated)

- When the structure is unpredictable (dashboards, presentations), the AI generates a one-off UI.
- Reserved for extreme scenarios only.

## CSS Strategy

- Modern CSS: variables, nesting, `@layer`.
- Trait-based composition (e.g. `.accent.txt`, `.accent.border`).
- No preprocessor needed.

## Strategic Significance

- **Thick shell**: control the AI's output through Schema and types to ensure reliability.
- **Deterministic UI**: JSON generation costs fewer tokens and is more accurate than code generation.
- **Multi-agent ready**: services are transparent to each other; any number of agents can intercept and inject.
- **Current status**: complete. Maintenance mode. The focus has shifted to graph memory as the core differentiator.

## Future Migration Path: The Aura Bottom Layer

When Fluxora matures, it can choose to migrate the bottom layer from Kafka to **Aura** (a modern distributed booth engine with storage-compute integration).

### Migration Benefits

| Dimension | Kafka (current) | Aura (future) |
|------|--------------|-------------|
| **Deployment complexity** | Requires a Kafka cluster | Single binary, 14ms startup |
| **Consistency** | Eventually consistent | Strongly consistent (Raft) |
| **State storage** | None (stateless services) | Fjall (stateful booth) |
| **Analytical capability** | None | Polars in-memory federated queries |
| **Operational cost** | High (Kafka cluster management) | Low (single binary) |

### Migration Strategy

1. **Gradual migration**: Fluxora's Gateway layer can selectively connect to the Aura backend
2. **Dual-track operation**: Kafka and Aura can run in parallel, switching over gradually
3. **Zero downtime**: smooth migration through the event-sourcing log (RaftLog)

### Architectural Relationship

```
[Fluxora UI layer] (independent project)
    ↓ Optional integration
[Aura bottom layer] (independent project)
    ├── Openraft (Raft consensus)
    ├── Fjall (Arrow IPC storage)
    └── Polars (in-memory analytics)
```

**Key**: the two projects remain independent. Fluxora is a complete event-driven UI framework that can run on its own; Aura is a low-level distributed infrastructure that can be used by other projects.

## Why Event Sourcing?

- **Decoupling**: business services are independent Kafka consumers. Adding/removing one touches no others.
- **Auditability**: the complete event history supports replay, debugging, and AI analysis.
- **Multi-agent**: services transparently intercept streams and inject responses.

## Why Separate Queues?

- **Flow control**: AI streaming = high-frequency/fragmented. A dedicated queue prevents blocking low-frequency control messages (notifications, joins, etc.).

## Architecture: Thick Shell, Sovereign Control

- Fluxora embodies Orbit's "thick shell" philosophy: the system controls the AI's output through Schema validation, not the other way around.
- The AI generates within constraints; the framework ensures correctness.
- Business logic is completely independent of UI rendering—services produce data, the Gateway applies templates, the UI renders.

## Quick Commands

```bash
cargo build --workspace   # Build all crates
cargo build -p gateway    # Build the Gateway only
cargo build -p ui         # Build the UI (WASM)
```

## Gateway Configuration

```toml
# gateway.toml — codec setting defaults
# The UI can override via URL: ?codec=json or ?codec=cbor (zero RTT)
# The codec is pinned at handshake—no runtime detection or update.
# Kafka is always CBOR.
```

---

## Pitfalls

- **`dyn Codec` is not object-safe**: a generic method on a trait cannot be used with `dyn`. Always use enum dispatch (`ActiveCodec`).
- **`gloo_net::websocket::Message::Bytes`, not `Binary`**: in WASM, `gloo_net` uses `Message::Bytes(Vec<u8>)` for binary WebSocket frames, not `Message::Binary` (variant not found, compile error).
- **axum 0.8 `Message::Text` takes `Utf8Bytes`**: a `String` does not convert directly—use `.into()`. `Message::Text(s.into())` works; `Message::Text(s)` does not.
- **`SessionManager` is wrapped in `Arc`**: `&mut` cannot be obtained through an Arc. Add `get_mut(&self)` on the wrapper to obtain a `dashmap::mapref::one::RefMut` for mutable access.
- **The WASM build fails because of Kafka**: `rdkafka` needs a native C library. Use `default-features = false` for `message` in `ui/Cargo.toml`.
- **Iggy macro scope**: the `iggy_conn!` macro references types behind `#[cfg(feature = "iggy")]`. The macro invocation must also be guarded by the same cfg.
- **False-positive lints**: the project uses Rust 2024 features (let chains, async fn) but some crates may report a "Rust 2015" lint. These are pre-existing edition-configuration issues, not logic errors.
- **Documentation updates**: when updating the README, **merge** new decisions with the existing ones. Never delete the original rationale.
- **`ActiveCodec` is `Debug + Clone`, not `Copy`**: do not try to use it in a match guard without cloning.
- **The Kafka codec is hard-coded to CBOR**: there is no `queue.codec` configuration field. The Kafka producer/consumer use `ciborium::ser::into_writer` and `ciborium::de::from_reader` directly. If Avro support is needed in the future, that will be a separate design effort.
- **Bincode's root cause is `deserialize_any` + internally tagged enums**: the deepest reason bincode fails is that `#[serde(tag = "...")]` requires `deserialize_any`, which bincode does not support. This affects all deserialization of the `Brick`/`Content` types, not just the wire protocol.
- **URL codec parsing must be independent of the token**: in `ui/src/main.rs`, the `?codec=` query-parameter parser was wrongly nested inside the `?token=` branch. If the token comes from a `data-token` attribute rather than the URL, codec parsing is skipped entirely. Parse the codec as a sibling branch, not inside the token branch.
- **`CodecHint` redundancy**: it is a structural duplicate of `CodecType`, with a `From` conversion and duplicated encoding logic. Replace it with `CodecType` on `Client` + a single `encode_ws()` function. Do not create parallel enum types that map 1:1.
- **`Status::send` must respect the codec**: the UI's `send()` was hard-coded to `ciborium::ser::into_writer` + `Message::Bytes`, always sending CBOR regardless of the configured codec. Use `self.codec.encode()` and match on the codec to choose `Message::Text` vs `Message::Bytes`.
- **Greet must be sent immediately, never delayed**: Greet contains the base UI layout. Delaying it to wait for a client first frame (for codec detection) adds 1 RTT and causes a blank screen on connect. Use the URL `?codec=` parameter instead—parsed at WS upgrade, zero extra RTT.
- **`handle_ws` parameter order**: `handle_ws(socket, outgo_tx, state, config, tmpls, codec, session)` — `codec` is the **final** value determined at handshake time (from the URL parameter or the config default), not an overridable fallback. Do not add dynamic detection logic.
- **Documentation and comments must be English only**: all ADRs, README sections, design decisions, and source-code comments must be in English. No Chinese in the project docs or code comments.

---

## Cross-References

This document is the complete design of the Fluxora project, forming a complete decision loop with the following detailed analyses:

- **[Aura Architecture](aura-architecture-en.md)**: a modern distributed booth engine with storage-compute integration, the low-level infrastructure Fluxora may optionally connect to in the future.
- **[Flux Architecture](flux-architecture-en.md)**: Fluxora is a radical practice of reactive architecture—the UI and protocol layers are fully event-driven, with no HTTP request-response model.
- **[Aura + Fluxora DevOps](aura-fluxora-devops-en.md)**: the engineering practice of Script-as-Code, including GitOps workflows, CI/CD configuration, and script testing and debugging.
- **[Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md)**: Fjall + Arrow + Polars, storage-compute integration across the full chain.

**The unified first principle**: no tech worship, no swallowing the pie the open-source world advertises; look only at the real physical limits of the hardware and the team's productivity.
