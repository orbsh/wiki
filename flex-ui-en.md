# Flex UI: An Event-Driven Declarative Framework for Transport UI

> **Languages:** [English](flex-ui-en.md) (primary) · [中文](flex-ui.md)

## Positioning

Flex UI is a set of **frontend/UI framework design principles**, complementary to general data-transport architectures:

- **[Flux Architecture](flux-architecture-en.md)** describes the **general principles** — how existing data flows, how the server pushes, how the client consumes incrementally by id.
- **Flex UI applies the same set of principles to "the transport UI itself"**: the UI is not the product of a render script / request-response, but is **transported over the WS as an incremental instruction stream, as a data structure**. It is a framework-level thing — describing the whole shape of "UI driven by data", not the implementation of any specific engine.

There are two generations of implementation. Fluxora was the first concrete implementation (event-driven UI bus, Rust/Dioxus, see [Fluxora Architecture](projects/fluxora-architecture-en.md), no longer maintained); the rendering layer is now carried independently by fluxen (path `~/world/fluxen/`, Rust/Leptos CSR): DSL, streaming merge, CBOR/JSON codec, Transport trait. Flex UI is the principle layer abstracted from them; implementation details (codecs, Kafka, specific crates) stay in the project documents and are not repeated here.

## Core Concept: UI as a Data Structure

The UI is not a "page + script", but **a typed component tree (Accrete), represented as JSON (or a more compact binary) and transported over an event stream**. Key corollaries:

- The UI is **serializable data**, not a program that cannot be transported.
- Components are **bound to event names**, not API URLs, and there are no HTTP verbs. Input = emit an event (write), Display = listen for events (read). The gateway automatically maps event names to transport topics.
- What is transported **is incremental instructions, not a complete snapshot** (consistent with the consumer-side principles of flux-architecture).

**Naming**: the DSL name is Accrete (formerly Brick). It takes "accretion" — the element stream continuously attaches and grows into an interface; the name is directly the core semantics itself (streaming cumulative updates), not a "building block"-style static assembly metaphor. It occupies one axis alongside the project name fluxen: fluxen governs the field (flux = stream, mechanism), accrete governs the growth of matter (accretion, result); they are cause and effect rather than synonyms of each other (repo docs/decisions/0004).

## Why the UI Is Described by Data, Rather Than Code/HTML

| Dimension | AI/business-generated HTML | AI/business-generated JSON (Flex UI) |
|------|------------------|----------------------------|
| Token consumption | High (lots of markup boilerplate) | Low (JSON is an order of magnitude smaller than HTML) |
| Output stability | Different every time (style drift) | Deterministic (styles locked down by the framework) |
| Data validation | No schema, relies on visual inspection | JsonSchema compile-time validation |
| Producer responsibility | Content + structure + style (all-in) | Only content and structure (style is left to the framework) |

**Essence**: take "creative decisions" (style) out of the producer's (AI/business) hands, and let it do only what it is good at (content and structure); rendering is left to deterministic code. HTML aimed at the consuming web is rendered on the fly by the browser, has no schema, and its style may drift on every render; Flex UI instead uses a verifiable, cacheable JSON data stream to turn "delivering a UI" into "delivering structured data".

## Declarative DSL: Accrete

- **Typed enum**, zero-ambiguity serialization (tagged); the producer outputs `{"type":"button",...}` → automatically deserialized into the corresponding component type. It is the single closed vocabulary of all component types (see "Component System: Closed Set").
- **Static subtree**: `children: Option<Vec<Accrete>>` declares a **fixed child-node tree** nested declaratively, recursively expanded at render time. (Early fluxora used the `sub:` field name, now retired; the wire format recognizes only `children`.)
- **List item template**: `item: Option<Vec<Accrete>>` is the **set of candidate templates** for list-type components (Fold/Rack) — used together with `selector` to pick a template for each item of the list data and render it; it is **not** the list data itself (the data comes from the `list` registry via a bind source), but rather "which template to choose for a list item". Be careful not to mistake it for a "list item".
- **Human-editable text frontend**: the wire format is JSON/CBOR; beyond that there are also KDL (true v2 syntax, aimed at layout push) and YAML (aimed at full Content actions). Carrier and substance are separated — swapping the carrier does not change the data model.

### Recursive Node Tree: Rendering and Instantiation

**Recursive rendering**: rendering an Accrete tree is depth-first expansion — check the current node's `children` (child node list): if present, recursively wrap each child into a child view and hand them, together with the node itself, to type dispatch; if not, hand it directly to type dispatch. The whole JSON tree is thus flattened level by level into an actual component tree, with the structure determined by data (`children` recursion) rather than by code.

**Type dispatch (`gen_dispatch!`)**: after flattening, the current node is **dispatched to the corresponding concrete component** (case/rack/button/input…) according to Accrete's type enum. Dispatch is generated automatically by the procedural macro `gen_dispatch!` from the `Accrete` enum AST as a match (rather than hand-written) — it faces a **closed enum**, so it can exhaust all components at compile time and hard-dispatch each kind to its own render function, with no runtime registration and no trait object overhead (see "Component System Design Trade-offs").

**Template instantiation (`Template {name, data}`) is completed before entering the registry**: the template mechanism fills data into an already registered template and produces a **new Accrete**. `expand(&env)` is called at the **dispatch stage, before writing into state slots** (see `dispatch_msg` in `ctx.rs`: every content action has `expand` run **before** the layout/data/list write) — on encountering a `Template` variant it takes the template by `name`, renders `data` into text, **deserializes back into Accrete to replace itself**, and **finishes recursively expanding the whole subtree before the data is persisted**, so that no `Template` variant remains in the incoming tree. Rendering therefore only ever sees already-expanded Accrete; **template instantiation is "preprocessing on receiving data, before it enters the registry", not runtime expansion at render time**. Templates can nest — if an Accrete produced by a template still contains a `Template`, it is expanded in the same pass.

The relationship and timing of the three: **recursive expansion determines the tree's shape (flattening at render time), type dispatch determines node form (dispatch by type at render time), and `Template` is expanded as preprocessing the moment data arrives (before entering the registry)** — shape and form are decided at render time, while parameters (template expansion) are ready at the data entry point.

### Bind System: Component ↔ Event Stream

The value of a bind slot is a **tagged variant** (discriminated by the `kind` tag, variant fields flattened):

```json
{
  "bind": {
    "input":  { "kind": "event", "event": "chat.send", "type": "text" },
    "output": { "kind": "source", "source": "chat.msg" }
  }
}
```

- **source**: listen to an event stream (read).
- **event**: emit an event (write).
- **target / field / submit**: advanced binding modes; the `field` variant may carry `payload` (such as extra data for a form field).
- **default**: fallback value slot (the `default` field gives a component a static initial value).

(fluxora's early bind was untagged — the variant was guessed from the key name; now retired; in the current protocol `kind` is required, and decoding an unknown kind raises an error.)

No API URL. The gateway maps event names to transport topics; components and backend services are decoupled through events, mutually transparent.

### Binding Data Source Mechanism (Consumer-Side Implementation)

A component's own `bind` defines how it gets data, depending on whether it is a **single-item component or a list component** — the frontend internally parses `bind` in the schema and queries a **globally registered data view**:

- **Single-item component**: parse the bind's source variant to obtain `source` (event name), take it out of the global `data` registry (the value is **a single Accrete**), and render it as the component's content. The current UI tree itself is an independent `layout` state, standing side by side with `data`/`list` as one of three — the top-level tree is in `layout`, and data fetched dynamically by source is in `data`/`list`.
- **List component** (rack etc.): `source` queries the independent `list` registry (each entry is a **Vec**) and renders item by item.

The registry's **notification granularity is a per-key lazy signal slot**: the outer map stores only handles, and values are published through inner slots — writing key K disturbs only K's subscribers, and does not broadcast to the whole table. This is the principle-level "incremental" extended inside the frontend: even from the frame to inside the renderer, no full-table notification is done.

**The data registry and the list registry must be kept separate** — they could be unified, and the simplest way would be **collection-first** — treat everything as lists, and a component that does not need a list simply holds a single entry. But **the list semantics of list components differ from those of single-item components**: fetching a list from `data`, or a single item from `list`, is semantically wrong either way. Keeping the two tables separate is deliberate. Moreover, many components genuinely do not have list semantics. Hence `data`/`list` remain separate tables.

**Send mechanism**: a component's write operation is to take the `event` defined in `bind` and `send` it out together with the **current data** (the component's local value) (`ctx.send(event, id, content)`). Pressing Enter in an `input` immediately does `send(event, None, val)` to emit the event.

**Component linkage (direct synchronization between components)**: it does not go through the backend. The premise is that **the shared DSL never carries runtime handles of the rendering framework** (see "Decision: A Single Rendering Backend") — form linkage is therefore not placed in the data but in the rendering layer: while traversing the subtree, the `form` component injects its own form state through the `Ctx` clone chain (each form holds its own clone, and nesting takes effect through shadowing), and input components bound via `field` write their current value into this state; on submit, all field values are collected and `send` out in one shot. The same kind of mechanism can be used for any component pair — sharing the same view-layer slot establishes reactive linkage, and the data model itself does not move.

#### `selector`: Dispatch Key for List Item Rendering

`selector` is an optional field of every component's `attrs`; it does not render itself and is consumed only when choosing a list item template — its role is **to let one and the same list render heterogeneous component forms per item**. The mechanism has two stages:

1. **Build the template index** (consuming the rack's `item` candidate template set): every template that carries a `selector` goes into `index: {selector → Accrete}`; those without a `selector` serve as `default`.
2. **Per-item selection** (rendering each actual data item): read the **data item's own** `selector`; if it hits a corresponding template in `index`, use it; if not, fall back to `default`.

Essentially it is a **discriminated-union-style list model**: data items carry a selector to match the template table, rather than all items sharing a single template — the form is "dispatch the rendering form by data item content".

### Streaming Merge (Incremental Update)

Three merge strategies are used for **localized path updates**, rather than a full redraw:

- **Replace**: overwrite the value.
- **Concat**: append (text concatenation, array push, object merge).
- **Delete**: remove (text replacement, key removal).

**Key point**: the merge is relative to the **component's bound data source**, and the merge identity is **pinned to `id`** — there is no second key axis (key template configuration is deliberately rejected, because the merge contract and DOM identity must be of the same origin; two keys would split row identity; composite identities such as `alice:m42` are assembled by the producer, not derived by the renderer). Rows without an `id` degrade to positional rendering, which is legal but loses DOM identity across insertions. Multiple services can stream to the same page without conflicting with each other — this is the same principle as flux-architecture's "command increments + frontend replay by id", only here the "instructions" describe the UI structure rather than business data.

### Message-Level Operation Channel (ev)

A downstream frame has a top-level operation channel field `ev`; the renderer **consumes only `ev="draw"`** (the rendering category), logs and ignores the other categories, and touches no state slot. On the surface it is strictly distinguished from the event names in bind (data-slot addressing, such as `login`, `chat`): `ev` is "who this message is for", while `event` is "addressing inside a rendering operation". Extension channels thus change from "changing the message structure" to "adding a new ev value + a new consumer" — the structure is frozen, the categories are open.

## Component System Design Trade-offs

### Closed Component Set, Deliberately Not Extensible

The component set is **finite and closed**. Three constraints jointly determine this:

1. **AI reliability** — the component set is the "vocabulary" for AI-generated JSON. An infinitely swelling vocabulary makes it impossible for the AI to output reliably. A closed set lets the producer (AI) learn it exhaustively.
2. **WASM/bundle size** — every component adds loading size.
3. **The enum is the gatekeeper** — the enum is the explicit boundary of the closed set; adding a component must manually pass through the enum. This friction is intentional (disabling a component = deleting one line from the enum).

The boundary can be adjusted (components replaced), but the total does not grow significantly.

### Why an Enum Rather Than Runtime Registration

`inventory`/`linkme` runtime registration (each component self-registers, a central registry looks up by name) does not apply:

1. **WASM compatibility** — it relies on linker behavior to collect registration entries, which may be silently lost on the WASM target.
2. **Type erasure** — runtime registration requires unifying everything into a function pointer signature, destroying compile-time type safety.
3. **Loss of compile-time safety** — a wrong component signature or a missed registration only surfaces at runtime.

### Why Not Replace the Enum with a Macro DSL

1. **Centralized audit** — the enum is a component inventory; a macro DSL scatters definitions into call syntax, losing centralized control.
2. **Module exclusion** — disabling a component = deleting one line from the enum; the macro approach needs an extra mechanism to control it.

### An Open Component System Is Infeasible in Rust/WASM

Open approaches (inventory, typetag trait object, macro DSL) each have structural problems in Rust/WASM. Rust determines everything at compile time, and the infrastructure for open registration is unreliable under WASM. **The closed enum is not a compromise, it is the natural solution under the constraints.** (JavaScript can be open because of runtime dynamism.)

## State: UI Store and Actions

- It holds three parts of state: `layout` (the current UI tree), `data` (single-item data by key), and `list` (lists by key), all of them per-key signal slots (see "Binding Data Source Mechanism").
- On receiving an action message (Create, Set, Join, Tmpl) → update the corresponding state; **first pass the operation channel filter** (see "Message-Level Operation Channel").
- **Streaming merge**: the `Join` action uses a merge strategy (Replace/Concat/Delete) and performs localized updates based on `id` matching — touching only the operated node and preserving the rest (collapse, local expansion, etc. all have independent per-instance state).
- **Row-level isolation**: each row inside a rack is a memoized view — untouched rows recompute to an equal value and produce no DOM work whatsoever. The "incremental" principle of merging is carried through to row granularity.

## Rendering

- **Streaming buffer**: accumulating streaming text uses a chunked buffer (rather than pre-concatenation or lifetime references); each chunk appends in O(1) amortized, and envelopes can be discarded early.
- **Component atomicity**: directly map standard HTML/SVG elements (Table, SVG), with no excessive semantic wrapping — LLMs are heavily trained on standard elements, so keeping them atomic ensures generation accuracy.
- **Synchronous template engine**: lightweight text rendering is pure computation, no IO, microsecond-level, and needs no async offloading (async wrapping would instead add overhead).
- **Style layer**: zero framework, CSS variables (oklch) + nesting + trait composition — like the component layer, a "closed vocabulary + determinism" (see [Flex UI CSS](flex-ui-css-en.md)).

## Design-Time vs Runtime

| Mode | Trigger | Characteristic |
|------|------|------|
| **Design-time (AI-assisted)** | AI generates Accrete JSON + mock data; the developer previews it and saves it as a template | Lowers the barrier to writing components |
| **Runtime (template-driven)** | Business returns pure data → apply template → millisecond-level rendering | Zero LLM latency, deterministic, no sensitive data into the LLM |
| **Exploratory (AI-generated)** | When the structure is unpredictable (dashboards, demos), AI generates a one-off UI | For extreme scenarios only |

## Architecture Decision Records (ADR)

### Decision: A Single Rendering Backend; the Shared DSL Does Not Carry Framework Handles

**Decision**: the framework keeps only one rendering backend (currently Leptos; Dioxus has been removed), and the DSL layer **never** introduces runtime types (signals, handles) of any rendering framework.

**Rationale**: this is a structural impossibility, not an engineering trade-off. Dioxus's `Signal<T>` does not satisfy `Sync` for non-`'static` input types, while Leptos's signal slots require `T: Send + Sync` — if the DSL carried framework handles, the two sides could mathematically not both hold at the meeting point that is the data model. View-layer state (such as form fields) propagates along the `Ctx` clone chain and does not enter the data.

**Counter-trade-off**: the previous form hung an `Option<Signal<Value>>` on the brick — it surrendered the very foundation of "framework-agnostic serializable UI" for convenience. After its removal, rendering behavior (form linkage, per-id row state) is built by the rendering layer itself, with slightly more code, in exchange for the DSL being true across backends.

### Decision: The Operation Channel Is Required; Breaking Changes Trade Away Structure Freezing

**Decision**: the top-level `ev` field of a message is required; the old shape (no ev) fails to decode outright, with no compatible default.

**Rationale**: silently treating a missing field as `draw` would incorrectly render frames of other future channel categories — errors must be loud. At present no external producer depends on the old shape, so it can still be changed; missing this window would only make the cost higher.

**Counter-trade-off**: a wire contract break. What is gained is that the "who is the message for" dimension exists from the protocol's birth, and the extension point (new ev value + new consumer) no longer requires touching the message structure.

### Decision: The UI Message Channel Uses Unbounded Sends, With No Explicit Backpressure

**Decision**: the frontend message channel uses unbounded sends (`UnboundedSender` / unbounded mpsc) and does **not** implement explicit `Backpressure` / flow-control logic for the queue. A slow consumer will not block the sender.

**Core invariant (the rationale, not empirical luck)**: **the CPU is always faster than the network, by several orders of magnitude.** This layer is guaranteed by physical distance, and carries through to the bottom:
- **First layer (physical)**: any optimization technique that makes data efficient **across kilometers (network transport)** can likewise be used for transport **across a few millimeters inside a chip**, and will only be faster — the physics of transport optimization do not change, and a shorter distance is necessarily better. Therefore "consumption rate (DOM/CPU) slower than production rate (network)" has only two possibilities: **a design problem** (a badly written program) → the program is not worth discussing; **a physical problem** (the network ought to be slower) → physically impossible — millimeters are always faster than kilometers.
- **Second layer (no queueing needed)**: the only reason queues/buffers exist is to absorb bursts where "production momentarily exceeds consumption". The network is the only production source and is **never faster than** the DOM, so there is no momentary production surplus — **buffer demand is 0, no queueing node is needed, and queueing theory no longer applies**. The conclusion is not "an unbounded queue is safe because the network is slow", but "the network is always slower than consumption ⟹ no queueing": messages go straight through and are processed on arrival. **The network itself is the natural rate limit of transport; that layer of backpressure does not need to be implemented by the business.**

**Why not explicit backpressure**: when the producer is the network and the consumer (CPU/DOM) is always faster than it, even the enqueueing step need not exist, let alone flow control; explicit `Backpressure` code would reimplement the rate limiting already implicitly provided by the transport layer, and is both redundant and unreliable (hand-written flow-control logic is itself a source of bugs).

**Boundary of counterexamples**: queues are necessary elsewhere because bursts where "production momentarily exceeds consumption" do exist (such as bursty batch processing of local computation); here the production source is the network and physical distance guarantees it is always slower, so the burst surplus is always negative. If in the future some scenario does have the CPU slower than the network — that is not a defect of this framework, but a design problem of the program in that scenario (not worth discussing within this framework), or a physically impossible situation.

**How it is verified**: this decision can be settled by the invariant reasoning above alone (physical distance → no queueing); measurement/load testing is only after-the-fact **corroboration**, not **discovery**. If a proposal needs testing before it can be determined to hold, one should be more conservative — that is not needed here, as the grounds are sufficient.

### Decision: No Conditionals/Loops/Scopes in Templates; Expressive Power Deliberately Limited

**Decision**: the template engine (`Template`) provides only simple instantiation with `name`+`data`, and does **not** introduce template language features such as conditionals, loops, or scopes.

**Rationale**: the ceiling on expressive power is a **deliberate choice**, not an oversight. Introducing conditionals/loops/scopes would turn templates from "filling data" into "a new language", and: first, one of the goals is to **simplify AI generation** — adding template syntax the AI is unfamiliar with is harder than letting the AI generate HTML+JS directly, directly contradicting the original intent; second, it preserves the isomorphism of "template = pure data filling", making `Template`'s semantics predictable and cleanly recursively expandable by `expand`.

**Counter-trade-off**: the cost is that templates cannot express logic such as branching/repetition — forms that need such logic are pre-structured by the business in `data` (conditions/lists already have representations in the data), or the AI directly generates concrete Accrete rather than templates. **Boundary**: genuine dynamic capability (such as runtime logic triggered by user interaction) should not be in the template layer at all; templates serve only deterministic rendering; anything needing arbitrary logic takes another route (such as generating a concrete tree) and does not mix with templates.

### Decision: Set (Data) / Create (Structure) / Tmpl Unified in One Protocol, Not Split into Two Axes

**Decision**: in the content protocol, various actions such as `Set` (update data), `Create` (rebuild structure), and `Tmpl` (register template) are **unified** in the same `Content` enum and the same message, and are **not** split into two channels by "data vs structure".

**Rationale**: analogize to an SQL client **transporting DDL and DML at the same time** — they are both "changes to the database" and ought to share one channel; this is not called entanglement but **the operation set of one and the same state machine**. Likewise, `Create`/`Set`/`Join` etc. are all "changes to the UI state machine", and unifying them in one protocol is what is simple. Splitting into a "data axis/structure axis" is artificially manufactured separation that breaks the unified mental model of "one class of operations".

**Counter-trade-off**: some would argue that "structure and data have different lifecycles and should be separated". But from the event-sourcing/state-machine perspective, both are actions on the same state tree; separation brings no control advantage and instead adds the maintenance and routing cost of two protocols. **Boundary**: if in the future some action's semantics differ so greatly that it cannot share one `Content` enum (unlikely), reconsider splitting.

### Decision: No Logic-Layer Caching in Rendering — the Bottleneck Is on the DOM Side

**Decision**: the rendering path does **not** implement generic logic-layer caching/memoization optimization.

**Rationale**: rendering is executed in Rust (outside WASM) or compiled to native; the cost of repeated logic-layer computation has already been pushed very low by language performance; the real bottleneck is on the **DOM side** (browser layout/painting), not on the component logic side. Logic-layer caching is an optimization habit of traditional JS/interpreted web frontends (JS is slow, so it needs memo); applying it to Rust has **no basis whatsoever** — an optimization point born of traditional inertia does not apply to a system that has already traded language performance for eliminating the bottleneck.

**Counter-trade-off**: caching can save redundant computation. But redundant computation here is microsecond-level and does not help solve the DOM bottleneck; caching introduces invalidation tracking, state inconsistency, and complexity, with benefits approaching zero while the risks are real. **Boundary**: one exception is retained — the rack's row-level memo is not "logic-layer speedup" but **the minimum granularity of reactive correctness** (without memo the whole list re-renders every frame, and the DOM work lands directly on the bottleneck side); if measurement shows that some component's logic computation has become a real bottleneck (low probability), optimize it in a targeted way; do not add caching in advance for a fictitious bottleneck.

### Decision: Template Inputs Are Not Schema-Validated Inside the Framework

**Decision**: the framework does **not** perform runtime schema validation on the `data` entering a `Template`.

**Rationale**: this is a **trusted system** — data entering a template has **already been validated at the entry point** (generated on the business side, or AI-generated and then validated on the agent/gateway side); the trust boundary is at the entry point, not in the framework. Validating again on every render inside the framework is **repeated overhead**: the same piece of data has already been validated once, and validating it again on the hot path is pure waste and puts the trust boundary in the wrong place.

**Counter-trade-off**: the lack of in-framework validation means the framework does not backstop anomalous data. But backstopping is properly the job of the entry point (constraints on the generation side), not of the rendering side; in-framework validation protects the scenario of "not trusting the entry point" — which contradicts this system's trust model. **Boundary**: if an untrusted external data source (not business/AI-generated) is introduced in the future, validation should be added **at the access point**, not on the framework's rendering path.

## Relationship to flux-architecture

- flux-architecture describes the **general principles**: how data flows, how it is pushed, how the client consumes incrementally on demand by dictionary.
- Flex UI is **the same principles applied to the UI itself** — what is transported is no longer business data but the UI structure itself (Accrete), and it likewise uses "command increments + on demand (the component tree as id-indexed data)".
- In other words: **Flex UI proves that flux-architecture's consumer-side principles (incremental instruction stream, on-demand dictionary, lazy loading) are not "a scheme for transporting data" but a general paradigm for "transporting any recursive structure"** — the UI tree is just one kind of payload for it.

## Cross-References

- **[Flux Architecture](flux-architecture-en.md)**: general principles of data flow and the client consumer side (the transport foundation of this framework).
- **[Fluxora Architecture](projects/fluxora-architecture-en.md)**: the first-generation implementation (Rust/Dioxus event-driven UI bus), no longer maintained; retired shapes such as `sub:`/untagged bind/`Render` are as recorded in that document; for the current protocol see below.
- **fluxen (`~/world/fluxen/`)**: the current rendering-layer implementation; decision archive `docs/decisions/0001–0004` (transport abstraction, single rendering backend, ev channel, brick→accrete rename).
