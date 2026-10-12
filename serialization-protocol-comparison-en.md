# Serialization Protocol Comparison: IDL vs Code-First

> **Languages:** [English](serialization-protocol-comparison-en.md) (primary) · [中文](serialization-protocol-comparison.md)

**Nature**: A comparative analysis of multiple options (not a single decision)
**Core philosophy**: Technology should not be the red tape that shackles a developer's hands and feet

---

## Overview

In distributed system design, the choice of serialization protocol directly affects developer experience, cross-language compatibility, backward compatibility, and performance. This document analyzes the pros and cons of the mainstream options and provides a **dimension-priority** comparison framework—not "pick one," but "under which dimension does which one come first."

→ This analysis affects the Raft control flow of [Aura Architecture](aura-architecture-en.md) and the data storage layer of [Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md).
→ For the encoding techniques of hand-written wire formats (variable-length length prefix, tag/length bit-sharing, bounds checking), see the dedicated section at the start of §3.

## The Full Genealogy: Layering by "How Much the Schema Participates at Decode Time"

Serialization formats line up along a single spectrum; the core variables are **how much self-describing information the data stream carries** and **how much the receiver must know in advance**:

```
Fully self-describing ←————————————————————→  Fully static/ordered
(with field names)                            (no field names/IDs at all)
CBOR / JSON           Tagged/TLV              Cap'n/SBE/            postcard/bincode
                      Cap'n/SBE/FlatBuffers   FlatBuffers
```

| Layer | Representatives | Carried in the data stream | Receiver must know in advance | Redundancy | Backward compatibility |
|:--|:--|:--|:--|:--|:--|
| **Dynamic (self-describing)** | CBOR, JSON, MessagePack | field **name strings** | almost nothing | largest | strongest (purely by name) |
| **Semi-dynamic (Tagged/TLV)** | Cap'n Proto, SBE, FlatBuffers, PB | numeric field **ID (tag)** | requires type-schema alignment | medium (numbers smaller than names) | strong (skip unknown IDs) |
| **Static (ordered)** | postcard, bincode | no names, no IDs | exact struct definition | **0** | none (adding a field breaks it) |

**Inside the semi-dynamic layer live two philosophies**: Cap'n Proto / FlatBuffers are **pointer-style zero-copy** (reading a single field is an O(1) address jump, strong evolution, large size); SBE is **fixed and compact** (nanosecond-level latency, weak evolution). They replace CBOR's "field name strings" with "numeric field IDs," which both sidesteps the name redundancy and preserves the evolution ability to skip unknown fields by tag—this is precisely the essence of "semi-dynamic."

### The Dynamic/Static Dual Semantics of Data Types

The reason the genealogy exists is rooted in the fact that data types themselves each carry two semantics. JSON's object/array each correspond to a pair of forms:

| Written form | Dynamic semantics | Static semantics | Applications of dynamic semantics | Applications of static semantics |
|:--|:--|:--|:--|:--|
| object | map (open key set, added/removed at runtime) | struct (fixed field layout, locked at compile time) | — | — |
| array | seq (length is a property of the content) | tuple (each position is meaningful, a fixed-length field record) | list, set, string | vector (an application of tuple) |

The genealogy of serialization formats is exactly the projection of this pair of semantics: dynamic formats (JSON/CBOR) carry the names and lengths of map/seq along with the data; static formats (postcard/rkyv/SBE) brand the layout of struct/tuple into the byte stream, zeroing out the metadata. **Whether a value goes dynamic or static depends on whether its length/key set is a property of the content or a property of the type**—the same JSON text admits both readings, and choosing the wrong reading is the source of all the pain points that follow.

From this comes an asymmetric corollary: **string has only the seq side, with no static counterpart**. The identity of a character sequence contains no length constraint; a "fixed-length string" is not a type but a type disguised by a storage layout. The `CHAR(n)` in databases is the product of this forgery; *see [MySQL Critique §Strings Have No Static Form](mysql-critique-en.md) for the defect list*.

**Dimension-priority quick reference**:

| Priority dimension | First choice | Rationale |
|:--|:--|:--|
| smallest size / pure Rust / fixed schema | **postcard** | 0 bytes of metadata, the most compact Varint |
| self-describing + cross-language (WASM boundary) | **CBOR** | carries field names, the gateway can partially parse without a template |
| zero-copy read + low latency + evolution | **Cap'n Proto** | pointer-style; fetching a single field does not deserialize the whole message |
| extreme low latency / fixed schema / high frequency | **SBE** | nanosecond-level, branchless, but weak evolution |
| general evolution + cross-language (not chasing extreme size) | **Avro** | 0 Tag pure data + JSON schema |

---

## 1. The Core Pain Point: The Philosophical Fault Line Between PB's IDL and Rust's Code-First

Protocol Buffers (PB), and indeed the vast majority of traditional industrial-grade communication protocols, have their greatest "fault line" and "pain point" in the Rust ecosystem—namely, the **IDL (Interface Definition Language)-driven declaration style** versus Rust's strongly **code-cohesive philosophy (Code-First)**, led by exportable structs (Struct / Enum with Derive Macros), share a natural "generation gap" and incompatibility.

In traditional microservices, PB forces you to:
1. First write a separate `.proto` file
2. Configure obscure `build.rs` plugins (such as `prost-build` or `tonic-build`)
3. At compile time, dynamically generate a pile of external Rust source that you never wrote by hand, is extremely ugly, and is hard to jump to or refactor directly in Neovim

This design directly shatters the visual satisfaction of **"compile-time strongly-typed cohesion"** that Rust programmers are proudest of.

---

## 2. In-Depth Reconciliation of Each Option

### Solution A: prost-derive Implements "Pure Rust Declarative (Code-First PB)"

Who says writing PB must start with a `.proto`? With Rust's powerful procedural macros and the prost ecosystem, you can perfectly well define your distributed command state machine directly in native Rust syntax, and at compile time have the macro automatically translate and map it into a PB-compliant binary stream, thus completely killing the IDL.

#### The True Rust Warlock Flow: 0 External Files, 100% Declarative PB State Machine

```rust
// src/store.rs
use serde::{Serialize, Deserialize};

// 🔥 The core trump card: use the prost procedural macro to mark PB field IDs directly on Rust structs
// Completely evaporate the .proto file and build.rs! Your code is the single source of truth

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct UpdateBoothStateCommand {
    // Marked as PB field ID 1, compile time automatically applies high-compression Varint
    #[prost(string, tag = "1")]
    pub agent_id: String,
    
    // Marked as PB field ID 2, holds the embedded multi-language Arrow byte-stream Payload driven by the user's supreme will
    #[prost(bytes = "vec", tag = "2")]
    pub arrow_payload: Vec<u8>,
}

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct TerminateBoothCommand {
    #[prost(string, tag = "1")]
    pub agent_id: String,
}

// Mapped through Rust's native Enum into PB's OneOf union
#[derive(Clone, PartialEq, ::prost::Oneof, Serialize, Deserialize)]
pub enum RaftCommandPayload {
    #[prost(message, tag = "1")]
    Update(UpdateBoothStateCommand),
    
    #[prost(message, tag = "2")]
    Terminate(TerminateBoothCommand),
}

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct AuraRaftCommand {
    #[prost(uint64, tag = "1")]
    pub term: u64,
    
    #[prost(oneof = "RaftCommandPayload", tags = "2, 3")]
    pub payload: ::core::option::Option<RaftCommandPayload>,
}
```

#### The Dimensional Strike of the Code-First Approach

1. **All-green jumps in Neovim**: Because all the state structs are native Rust code, in Neovim you can freely refactor via the most familiar `gd` (Go to Definition), add or remove fields at any time, and get seamless LSP smart hints. No extra external files torturing your eyes.

2. **A grand unification of multiple formats**: On the struct we mount both `::prost::Message` (handling the high-concurrency inter-cluster Raft votes and control-network transport) and `Serialize/Deserialize` (handling locking data as Bincode/MsgPack into Fjall's physical disk partitions, extremely cleanly). Data flows between the two systems without any manual escaping or mapping functions, and the types are absolutely safely, naturally consistent in kernel space.

### Solution B: Bincode/Postcard in a Pure Rust Environment

The core value of PB (Protocol Buffers) lies in **"absolute compatibility across a cross-language ecosystem"** (for example, your main cluster is written in Rust, but tomorrow you need to let microservice nodes written in Go/Java by the team join the consensus network).

If, in your reasoning and decisions, you have already concluded: "this brand-new Web service/Agent cluster of mine, from the front-end gateway to the back-end distributed nodes, is 100% pure Rust monolithic code":

- **Stop carving flowers onto PB immediately.** Resolutely veto all PB components with a single vote.
- **Turn completely to Bincode** (or the Postcard format designed purely for minimal compactness).

Because in a pure Rust environment, Bincode needs no IDL at all, no macro markers at all, and contains no field-ID redundancy at all. Based on the Rust struct's own memory layout, it can directly generate hardware-level pure raw-byte encode/decode code at compile time. Its metadata overhead is an absolute **0%** (lighter and faster than PB with Tag IDs—it is the true bare-metal muscle).

#### Fluxora's Real-World Pitfall: Bincode's Fatal Flaw

Bincode's "bare-metal muscle" was verified as unusable in practice in the Fluxora (AI-native UI framework) project. [→ See ADR-001](projects/fluxora-decisions-en.md)

**Root cause**: Bincode **cannot deserialize `#[serde(tag = "...")]` enums**—it calls `deserialize_any`, which bincode does not support. This affects all deserialization of the `Brick`/`Content` types, not just the wire protocol.

**Collateral fatal injuries**:
- serde 2.x incompatibility (v1.x broken, v2.x API unstable)
- No type self-description (the Gateway cannot partially parse routing metadata)
- No cross-language support (bincode has no JS library)

**Conclusion**: Bincode's zero-metadata design is a double-edged sword—it is indeed extremely lightweight in pure-Rust internal RPC scenarios, but the moment serde advanced features (internally tagged enums, `deserialize_any`) or cross-language needs come into play, it collapses outright. Fluxora ultimately migrated to CBOR, while Aura chose Postcard (inheriting serde ecosystem compatibility + built-in Schema evolution).

### Solution B2: The Semi-Dynamic Layer—Cap'n Proto / SBE / FlatBuffers

Between the [dynamic self-describing (CBOR/JSON)](#the-full-genealogy-layering-by-how-much-the-schema-participates-at-decode-time) and the [static ordered (postcard/bincode)](#the-full-genealogy-layering-by-how-much-the-schema-participates-at-decode-time), these three formats address by **numeric field ID**, not by field name string, nor by pure order. They avoid CBOR's field-name-string redundancy while preserving the evolution ability to "skip unknown fields by tag."

#### Cap'n Proto—Pointer-Style Zero-Copy

- Message = a tree in memory (segment + pointers). Reading a field is an **O(1) address jump**; you can pull out a single field without deserializing the whole thing.
- Strong schema evolution: `@N` field IDs + union, unknown IDs skipped.
- **Cost**: the message contains pointer structures + alignment padding, **so the size is not compact**. Zero-copy's "grab-and-use" holds in memory, but over the network/when persisted the pointers are offsets and the advantage weakens.

#### FlatBuffers—Same School as Cap'n (offset table)

- Zero-copy, the buffer is the memory layout; mainly for game/mobile serialization caches.
- Positioning and evolution the same as Cap'n, with a more "engineering, quick-tune" style.

#### SBE (Simple Binary Encoding)—Fixed, Compact, Row-Style, Purpose-Built for High-Frequency Finance

- Fixed-width struct layout + field IDs, nanosecond-level latency, branchless, extremely compact.
- But **schema evolution is very weak** (can only append fields at the tail, fixed width), and rollout is nearly all-or-nothing.
- The schema is defined in XML and generates code (at compile time).

#### The Two Philosophies of the Semi-Dynamic Layer

| | Pointer-style (Cap'n / FlatBuffers) | Fixed compact (SBE) |
|:--|:--|:--|
| Read | O(1) address jump, zero-copy field fetch | sequential tag scan, fixed width |
| Evolution | strong (@N ID + union) | weak (append at tail, fixed width) |
| Size | large (pointers + alignment padding) | compact |
| Latency | low | nanosecond-level |
| Suited to | deserialization scenarios needing zero-copy + evolution | high-frequency fixed schema (finance) |

#### ⚠️ Also IDL-Driven: PB / Cap'n / FlatBuffers Are the Same Disease

Many people who escape from Protobuf treat Cap'n / FlatBuffers as the antidote, **and this is a misconception**—all three are the **same family** that "first writes a standalone IDL, then generates stub code," and what users dislike (external IDL, generated code that cannot be hand-edited, broken jumps, mismatch with modern derive workflows) is exactly the same for all three:

| Capability | Protobuf | Cap'n Proto | FlatBuffers |
|:--|:--|:--|:--|
| Definition style | `.proto` standalone IDL | `.capnp` standalone IDL | `.fbs` standalone IDL |
| Rust without IDL (code-first / derive) | ✅ **has prost-derive** | ❌ no official derive | ❌ no official derive |
| Rust ecosystem | **most mature** | medium | official but low-level and cumbersome |
| Zero-copy field fetch | ❌ sequential parsing | ✅ pointer-style | ✅ offset table |
| Cross-language bindings | **widest** | medium | medium |
| Size | Varint + Tag compact | inflated (pointers + alignment) | inflated |

A counterintuitive conclusion: **among the three, only the PB family gives you an exit**. `prost-derive` (Solution A) uses Rust structs + the `#[prost(tag="1")]` procedural macro to produce PB-compatible binary without writing a `.proto`—this is the only member of the IDL family that satisfies "close to Rust procedural macros." The Rust side of Cap'n / FlatBuffers has no official derive; it must be files + compile-time generation, all the way down a single path.

`prost-derive` being feasible stems from Solution A:

```rust
// No .proto needed, the Rust type is the definition, mapped at compile time to PB-compatible binary
#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct UpdateBoothStateCommand {
    #[prost(string, tag = "1")]
    pub agent_id: String,
}
```

#### The Real Classification: An IDL Must Be a Standard, Programmatically Operable Data Format

There is nothing wrong with an IDL itself—the problem is **whether the IDL is a "standard data format" and can be conveniently operated on by programs**. The right direction for escaping the IDL is not "no IDL," but looking at **whether the structure definition is standard, programmatically operable data**:

- **JSON / KDL / TOML**: **the source is the data**, self-describing, can be directly operated on with grep/jq/editor scripts, interpretable without any schema. Data as a citizen.
- **Avro (.avsc)**: the IDL is **pure JSON**—a standard data format; programs can use generic JSON tools (jq / python / any language) to **conveniently generate, validate, evolve, and do schema registry**. This is the template for a "good IDL": the structure definition is both readable and programmatically operable, and `0 Tag` pure data stays compact.
- **PB / Cap'n / FlatBuffers / SBE (.proto/.capnp/.fbs/XML)**: the IDL is **proprietary syntax**, parseable only by a dedicated toolchain (protoc/capnpc/flatc); programs cannot operate on it directly with generic data structures, and the generated code cannot be hand-edited. This is the true root of the "annoyance"—not "having an IDL," but "the IDL not being a standard format."

**"Higher readability" is not a talisman for a proprietary IDL either**—it is squeezed from both ends, being neither free of cost nor exclusive:

- **It is not "readable," it is "readable only after you've learned it"**: the high-density readability of `.proto`/`.fbs` holds only for someone who already knows the syntax; for a first-time unfamiliar reader it is less plain than JSON (`name: "age", type: "int"` is understood at a glance). And to cash it in you must first pay a learning cost, then keep maintaining it with a dedicated toolchain—it is not a one-time expense.
- **Readability is not a privilege of proprietary formats**: the actually low-readability one is JSON (quotes/braces/nesting noise—precisely .avsc's real weakness); but among standard data formats there is a tier **designed specifically for human-readable definitions**—**TOML / KDL**, standard, programmatically operable (every language has an off-the-shelf parser), and achieving readability close to `.proto`. On these two points of readability, the proprietary IDL has no foothold.
- **postcard / bincode**: the `#[derive(Serialize, Deserialize)]` procedural macro, **the type is the definition**, no external IDL, structure and Rust type unified; it is the correct answer for "close to the modern derive workflow" (sacrificing cross-language self-description, holding only inside pure Rust).

**Beyond the timing of initialization parsing lies the real divide between "standard vs proprietary."** As noted above, at initialization, reading the IDL versus JSON makes no essential difference; but the operability of that schema at **all its remaining moments of use** differs enormously between the two formats. The divide falls in three places:

1. **Operability during editing/operations**: if the schema is JSON, it can be casually queried, validated, diffed, and dynamically generated by jq/grep/python; `.fbs` can only rely on a dedicated toolchain (flatc). This is the difference in "whether the file in hand can be processed generically," occurring at any moment the schema is touched by any person/tool, not only that one time at engine initialization.

2. **Whether the schema travels with the data at runtime / whether out-of-process interpretation is required** (the key point): when cross-language, cross-process, gateway/WASM boundaries need to understand the data—
   - If the schema is standard JSON (or CBOR self-describing): the receiver can **parse independently**, without depending on the sender's code, and can also partially parse only the routing metadata, understanding at any time.
   - `.fbs` / Rust-layout-only (rkyv): requires the other side to **hold the corresponding generated code or a schema companion**; cross-language it may even be completely uninterpretable (a non-Rust system cannot read rkyv by Rust layout). If a gateway/WASM boundary wants to route by it, it must maintain a generated stub for every language, and the binding is hardcoded.
   - The gateway boundary scenario: the gateway reads only the routing field at the boundary and forwards the rest of the payload as-is (zero-copy borrow of `&[u8]`, see GatewayRouterEnvelope in the next section)—if the schema is standard JSON/CBOR, the gateway can independently extract the routing metadata and forward; if it is bound to a proprietary IDL or a Rust-only layout, the gateway either cannot understand it or is forced to fully deserialize before forwarding, and the zero-copy advantage is canceled out by the IDL choice.

3. **How the schema evolves**: a JSON schema (Avro) can evolve smoothly via registry + default (rolling upgrades do resolution at read time, see [Solution C](#solution-c-apache-avros-schema-evolution)); `.fbs` evolution is bound to the IDL compilation artifacts—changing the schema means regenerating code, which is especially painful across languages.

**Conclusion correction**: Bid farewell to "proprietary-syntax IDLs" and head down one of two roads—either `postcard` (pure Rust internal, derive, the type is the definition), or `Avro` (if you need a standard, programmatically operable IDL + cross-language + evolution, a JSON schema is the template), plus `CBOR` (cross-language/WASM, serde + self-describing). **Avro should not be grouped in the same tier as PB/Cap'n/FlatBuffers**—it defines its schema with standard JSON, conveniently operable by programs, and precisely sidesteps the problems you dislike.

**Echoing KV's DDL**: encodings like Cap'n / SBE's "field IDs + skip unknowns" are exactly the same idea as "packing a whole row with Tagged/TLV evolvable serialization" in the [KV Storage Engine](kv-storage-engine-en.md)—field-ID addressing = the KV DDL's TLV scheme. On the genealogy it is a reasonable compromise between postcard (static) and CBOR (dynamic): more evolution capability than postcard, less field-name overhead than CBOR. If you dissect down to the schema level, "standard vs proprietary" applies equally: Avro's JSON schema is programmatically operable, .proto's proprietary syntax is not.

### Solution B3: Zero-Copy Formats—the Actual Division of Labor Among rkyv / Arrow / Avro

First draw the **failure boundary** of zero-copy clearly, otherwise it is easy to mistake "no off-the-shelf implementation exists in the ecosystem" for "physically mutually exclusive":

**Zero-copy has truly only two structural failure modes:**

1. **The offset cannot be branded into the buffer**: when a data structure references something outside the buffer (a heap `Box`/`Rc`, an external object), a relative offset can only represent inside-buffer positions and cannot represent external references—such types cannot be branded into a zero-copyable form.
2. **The layout cannot be replayed byte-for-byte in the target environment**: a non-Rust artifact cannot be interpreted by the Rust-compiled layout (rkyv's Rust-only restriction is of this kind), or the byte order/alignment does not hold on the target end.

As long as the offset can be branded into the buffer (a relative pointer), neither mmap nor memmove fails, and zero-copy always holds for "fetching a single field."

**Two things easily misjudged as "mutually exclusive" that actually are not:**

- **The schema changing ≠ the offset cannot be branded in**: when the schema changes, the old buffer is invalidated—this is a **common tax of all serialization schemes** (postcard/bincode likewise re-encode, Avro relies on read-time resolution), not something unique to zero-copy. What it causes is a version/migration **cost**, not a zero-copy failure; rkyv can brand its own offset for every schema version.
- **A standard IDL ≠ the schema necessarily changes dynamically**: a **compile-time-fixed** schema defined in standard JSON (such as a fixed .avsc) is not inherently mutually exclusive with zero-copy—a fixed layout can brand offsets as usual. What is truly mutually exclusive with pointer-style zero-copy is the narrow state of "**the schema being dynamically generated/changed frequently at runtime**": branding a different offset on every run, so an already-encoded buffer cannot be stably reused.

Therefore, what zero-copy "cannot have both" is not "a standard IDL" but "**a dynamic schema**." That no off-the-shelf format simultaneously satisfies "zero-copy + row-store pointers + standard IDL" is an **ecosystem/engineering status quo** (no implementation of the two yet exists), not a physical necessity. So it remains three approximators, each compromising in a different direction:

#### rkyv—Pure Rust Zero-Copy, "the Type Is the Schema"

- **Zero-copy deserialization**: relative pointers are branded into the output (offset-based); reading is an O(1) address jump, without deserializing the whole thing.
- **Relative pointers rather than absolute addresses**: the serialized buffer can be memmoved wholesale and can be **mmap**ed, without its internal pointers becoming invalid. This is the fundamental reason it, like Cap'n, is independent of the address space, and it is why it treats mmap read-only hot queries as its first design goal.
- **schema = Rust type + 3 procedural macros** (`Archive` / `Serialize` / `Deserialize`), with no standalone IDL file—code-first to the extreme, 100% programmatically operable, more "standard" than a proprietary `.capnp`. Reading an `Archived<T>` field is O(1).
- **Hard constraints**: requires an aligned buffer (`AlignedVec`) + aligned access (paid for with padding); the layout is encoded/decoded only by **Rust compilation artifacts**—**i.e., Rust-to-Rust (including pure Rust compiled to wasm) can read/write; a non-Rust artifact (host JS, Go, including wasm compiled from them) cannot interpret it**. wasm is a compilation target, not the restriction itself—pure-Rust wasm is entirely usable. Evolution compatibility is weak (adding fields/changing classes requires managing versions or migrations yourself, not as smooth as Avro/Cap'n).
- **Positioning**: a mmap point-query tool of choice for massive read-only KV values, not in conflict with postcard (compact) or CBOR (cross-language), and can be layered.
- **Placement boundary (don't set out as a 'performance multiplier')**: zero-copy is only cashed in on the hot path of "serving repeated random point queries over an mmap archive buffer without materializing owned structures." If it is "KV → Arrow/Iceberg/Parquet batch export," the bytes are copied into Arrow/Parquet buffers as usual, and rkyv can only save the intermediate owned deserialization; the gain is less than the full "deserialization-free" amount. rkyv and postcard/CBOR are not an either/or but a division of labor by layer; and since it is a Rust-only proprietary layout + evolution that requires self-managed versions (see above), it is suited only to the "stay-in-process" half, while cross-language/egress still goes through Arrow / Avro.

```rust
#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]
struct Example { id: u32, name: String }
// (API names depend on the 0.7/0.8 version, including to_bytes / access and the rancor error system)
```

#### Arrow (IPC)—Standard Schema + Columnar Zero-Copy

- The schema travels with the data, a generic Arrow Schema object, JSON-serializable, programmatically operable, not bound to a proprietary IDL.
- Zero-copy reads are columnar-aligned (not row-store pointer-style).
- Drawback: columnar, so fetching a single row/field is not as direct as row-store pointer-style.

#### Avro (.avsc JSON)—Perfect Standard IDL, but Not Zero-Copy

- The schema is standard JSON, programmatically operable to the extreme.
- Sequential encoding, no internal pointers; reading requires decoding the whole record in schema order.

#### Approximator Comparison

| Position | Satisfies | Compromises |
|:--|:--|:--|
| rkyv | zero-copy + row store + code-first | only Rust compilation artifacts can decode it, no standard IDL file, weak evolution |
| Arrow | standard schema + zero-copy | columnar, not row-store |
| Avro | standard IDL + cross-language + evolution | not zero-copy |

**Conclusion**: Zero-copy + row-store pointers can be had together with a **compile-time-fixed standard IDL** (a fixed .avsc can brand offsets as usual); what is truly mutually exclusive is only the narrow state of "zero-copy + dynamic schema," and the ecosystem has no off-the-shelf implementation. So the approximator is still chosen by scenario—pure-Rust internal mmap hot queries → rkyv; standard-schema batch column access → Arrow; cross-language + evolution → Avro.

### Solution C: Apache Avro's Schema Evolution

Avro's most terrifying and also most fascinating physical property—when data travels over a network cable or is written into a physical log, it **contains no field names at all** (unlike CBOR), **and not even field IDs** (unlike PB/Thrift)!

How does it do it?

When Avro serializes, it strictly follows the order of the Schema you defined in JSON, laying the data one after another flush against each other into a pure physical byte stream.

```
PB layout:   [Tag ID + type] -> [value] (still a tiny bit of Tag redundancy)
Avro layout: [value 1] -> [value 2] -> [value 3] (muscle data, pure to the extreme)
```

#### A Universe-Scale Backward Compatibility That Is a Dimensional Strike (Schema Evolution)

This is the absolute trump card with which Avro dominates big-data pipelines (such as Kafka/stream computing), and the place where it is far more powerful than Bincode.

**Pain point**: When writing Rust, if you have stored data with Bincode and tomorrow your booth upgrades by adding a new field to the struct, an old node reading the new data will instantly crash (Panic).

**Avro's solution**: In Avro, as long as the new and old nodes each hold a corresponding JSON spec sheet, Avro's decoder (based on the `apache-avro` crate) will automatically perform **"Schema validation and alignment (Schema Resolution)"** in memory.

- If a new node finds that old data is missing a field, it automatically fills it in with the `default` value written in the JSON
- If an old node reads new data, it automatically ignores and instantly "skips" the extra unknown bytes

The entire distributed cluster can achieve **online, imperceptible rolling upgrades** extremely smoothly, without stopping any server.

#### A Seamless "Multi-Language Sandbox Grand Unification" Contract

Because its spec sheet is standard JSON, this means your set of user-dynamically-driven multi-language engines (Steel Lisp / PyO3 Python) can, with no modification to any underlying Rust code at all, naturally, natively, 100% understand the current communication contract!

- **Lisp linkage**: the embedded Steel Lisp VM can directly use its forte, S-expressions, to parse and dynamically compose this JSON spec sheet
- **Python linkage**: Python needs no extra C-Binding compilation at all; a native call to `json.loads()` takes it over perfectly

This fully realizes zero-obstacle cross-language communication between the application layer, the scripting layer, and the consensus network layer.

#### The Cold Physical Reality: Why Avro Is Unsuitable to Replace Arrow?

Although Avro sounds flawless, one must guard its physical boundary with extreme sobriety:

- ✅ **It is only suitable for**: Raft → Avro (taking over the signaling and command-broadcast packets of the multi-node consensus network)
- ❌ **It is absolutely unsuitable for**: Fjall → Avro (it cannot replace Apache Arrow for running hot-data analytics)

**The fatal shortcoming (the physical chasm between row and column)**:

However streamlined Avro is, its essence remains **row-oriented storage**. In memory, the data is laid out user by user, one next to another.

Once you reach the stage of needing Polars for arbitrary-dimension impromptu complex multi-table JOINs and batch dataframe merges, row-laid-out Avro instantly falls behind. Because a multi-core CPU cannot activate SIMD hardware vectorized parallel acceleration in Avro's memory. If you force Avro for analytics, Polars is compelled to disassemble the data block by block at runtime, and the speed will be **hundreds of times slower** than native Arrow columnar alignment.

---

## 3. Dimension Criteria

### Common Encoding Techniques for Hand-Written Binary Frames (wire-format style)

Beyond the selection analysis, many boundary protocols (internal RPC frames, file headers, KV wire formats) do not pull in a library at all—a dozen or so lines of hand-written encode/decode suffice. The techniques of these formats are highly convergent; mature formats (protobuf, CBOR, Redis RESP, SBE) repeatedly use the same handful of tricks. Relationship-wise: **CBOR is the complete form of the "fixed-bucket length prefix" idea**—it generalizes the techniques of this section into a universal self-describing format (each item's first byte = top 3 bits type + low 5 bits length, with length buckets of 23/one/two/four/eight bytes, isomorphic to the fixed-bucket encoding below). The cost of self-description (a type on every field) cannot be paid in an internal protocol, so an internal frame only borrows the techniques, not the format.

- **A variable-length length prefix, not a byte-by-byte varint**: for values of the "length" kind (tens to tens of thousands of bytes), a better form is fixed-bucket encoding—the first byte's top 2 bits select the width bucket, the low 6 bits hold the high bits of the value, and subsequent bytes are concatenated in order:

  ```text
  00xxxxxx                 inline ≤63 (one byte covers the vast majority)
  01xxxxxx + 1 byte        14 bits
  10xxxxxx + 2 bytes       22 bits
  11xxxxxx + 4 bytes       32 bits
  ```

  Compared with protobuf/LEB128's byte-by-byte varint (7 bits/byte + a terminator bit): small values are one byte for both, but varint needs 2~3 bytes for medium values (128~16K) with a branch test on every byte, whereas fixed-bucket encoding assembles in one shift and has only a 2-bit selector for a branch. Fixed-bucket's trade-off is rounding waste within the ceiling (the 22-bit bucket is a 2-byte tail until 4M)—in the length-prefix scenario this waste does not exist, because the buckets are chosen by order of magnitude.

- **tag and length share the first byte**: when there are only 2~4 enum tag values (≤4 uses 2 bits, ≤16 uses 4 bits), merging the tag nibble and the length selector into the same first byte saves 1 byte per record. Shifts/masks are single-cycle pipeline instructions (sub-nanosecond), so performance is no concern; the real cost is **one extra selector branch per field** and a wider bad-frame test surface—this is a maintainability trade-off, not a speed trade-off.

- **Being non-self-describing is the premise of an internal protocol, not a defect**: a field name (CBOR/JSON style) or field ID (protobuf style) appearing in a frame is valuable only when "the decoder may not recognize the field" (evolution, cross-organization). When both ends are defined by the same code (compile-time type locking), field order is the schema, length is the field boundary, and all metadata is waste. The criterion: **does the decoder need to skip unrecognized fields?** Only then carry an ID/tag.

- **Leave the total-length header to the transport layer**: message-style transports (socketpair/mpsc `Vec<u8>`, WebSocket message) are naturally delimited; writing the total length inside the frame again stores the same information twice. Only byte-stream transports (raw TCP) need in-frame delimitation, and that is the transport layer's framer's job too. Likewise, **the frame layer does not compress**—compression is either done at the field-wrapper layer (knowing value semantics: timestamps can be delta'd, floats can be quantized) or not at all; general-purpose compression is a net negative for small ops (zstd's minimum frame overhead of 6+ bytes > most ops themselves), and it invalidates the boundary contract of "not parsing the content."

- **The parser's entire responsibility is bounds checking**: the only extra obligation a hand-written parser has over serde—after each len is read, first check whether it exceeds the remaining byte count, rejecting over-limit rather than panicking (malicious/corrupt frames are routine input). About 30 lines; this is the only code a hand-written scheme writes more of than a library brings in, in exchange for zeroing out dependencies.

- **Hex-layout lock-in tests**: the byte layout is a persistence contract; lock it down byte by byte with `assert_eq!(bytes, &[0x00, 0x07, ...])` (OKM's established discipline, see the key/index layout tests). A format change must explicitly change the test, to prevent a "casual optimization" from silently breaking production bytes.

### The Per-Dimension Comparison Table of Each Option

| Dimension | A: Code-First PB | B: Bincode | B2: Semi-dynamic (Cap'n/SBE/FlatBuffers) | B3: rkyv | C: Avro |
|---------|------------------|-----------|-------------------------------------|----------|---------|
| **Genealogy position** | Semi-dynamic (Tagged/TLV) | Static (ordered) | Semi-dynamic (Tagged/TLV) | Static + zero-copy (the type is the definition) | Static (ordered, but with out-of-band schema) |
| **Cross-language** | ✅ multi-language | ❌ pure Rust | ✅ Cap'n/SBE multi-language | ❌ only Rust compilation artifacts | ✅ standard JSON Schema |
| **IDL** | ❌ replaced by macro markers | ❌ not needed | ✅ needs IDL (.capnp/XML) | ❌ not needed (the type is the definition) | ✅ JSON file |
| **Encoding friction** | medium (prost macros) | **extremely low** (zero config) | high (IDL-driven) | **extremely low** (3 derive macros) | low (JSON Schema) |
| **Compression ratio** | high (Varint + Tag) | **extremely high** (0 metadata) | medium (pointer/alignment inflation) or compact (SBE) | medium (alignment padding) | extremely high (0 Tag pure data) |
| **Backward compatibility** | ✅ stable field IDs | ❌ adding a field breaks it | ✅ skip unknown IDs (Cap'n strong) | ⚠️ weak (layout depends on version, self-managed migration needed) | ✅ **Schema Evolution** |
| **Zero-copy field fetch** | ❌ | ❌ | ✅ Cap'n/FlatBuffers | ✅ **O(1) jump + mmap** | ❌ |
| **Suited scenario** | needs multi-language access | pure Rust, fixed schema | zero-copy + evolution deserialization | pure Rust mmap read-only hot queries | smooth rolling upgrades |

### Dimension-Priority Criteria

The purpose of this table is **not to pick one**, but to find the unique answer for each dimension:

- **Want minimum size + pure Rust + fixed schema** → **postcard / bincode** (0 metadata tax)
- **Want self-describing + cross-language (WASM gateway boundary)** → **CBOR** (carries field names, partially parseable)
- **Want zero-copy reads + low latency + evolution** → **Cap'n Proto** (pointer-style; fetching a single field does not deserialize the whole message)
- **Want extreme low latency + fixed schema + high frequency** → **SBE** (nanosecond-level, weak evolution)
- **Want pure-Rust zero-copy + mmap read-only hot queries** → **rkyv** (the type is the definition, O(1) jump, no IDL)
- **Want general evolution + cross-language (not chasing extreme size)** → **Avro** (0 Tag + JSON schema)

### Decision Tree (by dominant workload)

```
What is the workload of your scenario?
├── Pure Rust internal RPC, fixed schema, chasing minimum size
│        → Use Postcard (if evolution is needed) or Bincode (if purely fixed and you can tolerate adding a field breaking it)
│          Note: neither supports #[serde(tag=...)], use Externally Tagged
│
├── Pure Rust, read-only hot queries, can mmap massive data without deserialization
│        → Use rkyv (the type is the definition, O(1) zero-copy jump, no IDL)
│
├── Cross-language needed, frequent queries, requiring zero-copy field fetch + evolution
│        → Use Cap'n Proto (pointer-style zero-copy)
│
├── Cross-language needed, high-frequency fixed schema, nanosecond latency
│        → Use SBE (weak evolution, suited to constant structures)
│
├── Cross-language needed + general smooth rolling upgrades, but not chasing extreme size
│        → Use Avro (0 Tag + JSON Schema Evolution)
│
└── Cross-language needed + edge/gateway self-describing partial parsing (WASM)
         → Use CBOR (carries field names, can partially parse routing metadata)
```

---

## 4. Avro Schema Example

```json
{
  "type": "record",
  "name": "AuraRaftCommand",
  "namespace": "aura.raft",
  "fields": [
    {
      "name": "term",
      "type": "long"
    },
    {
      "name": "payload",
      "type": [
        "null",
        {
          "type": "record",
          "name": "UpdateBoothStateCommand",
          "fields": [
            {
              "name": "agent_id",
              "type": "string"
            },
            {
              "name": "arrow_payload",
              "type": "bytes"
            }
          ]
        },
        {
          "type": "record",
          "name": "TerminateBoothCommand",
          "fields": [
            {
              "name": "agent_id",
              "type": "string"
            }
          ]
        }
      ],
      "default": null
    }
  ]
}
```

---

## 5. Rust Code Example

### Avro Serialization and Deserialization

```rust
use apache_avro::{Schema, Reader, Writer};
use serde::{Serialize, Deserialize};

// Load the Schema from a JSON file
const AVRO_SCHEMA: &str = include_str!("../schemas/aura_raft_command.avsc");

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpdateBoothStateCommand {
    pub agent_id: String,
    pub arrow_payload: Vec<u8>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TerminateBoothCommand {
    pub agent_id: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RaftCommandPayload {
    Update(UpdateBoothStateCommand),
    Terminate(TerminateBoothCommand),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuraRaftCommand {
    pub term: u64,
    pub payload: Option<RaftCommandPayload>,
}

// Serialize
fn serialize_command(cmd: &AuraRaftCommand) -> anyhow::Result<Vec<u8>> {
    let schema = Schema::parse_str(AVRO_SCHEMA)?;
    let mut writer = Writer::new(&schema, Vec::new());
    writer.append_ser(cmd)?;
    writer.flush()?;
    Ok(writer.into_inner())
}

// Deserialize (automatic Schema Resolution)
fn deserialize_command(bytes: &[u8]) -> anyhow::Result<AuraRaftCommand> {
    let schema = Schema::parse_str(AVRO_SCHEMA)?;
    let mut reader = Reader::with_schema(&schema, bytes)?;
    
    // Avro automatically handles Schema Evolution
    // If the data is missing a field, it is filled in with the default value
    // If the data has extra fields, they are automatically skipped
    let cmd: AuraRaftCommand = reader.next().unwrap()?;
    Ok(cmd)
}
```

---

## 6. Aura's Final Choice (updated 2026-06-20)

In the Aura architecture, we adopt a **layered serialization strategy**:

| Layer | Serialization format | Rationale |
|------|-----------|------|
| **Raft control flow and RPC** | Postcard | pure Rust declarative, Varint compression, the Postcard-Schema macro supports Schema evolution, completely killing Bincode's fatal flaw of breaking when adding a field |
| **Local transactional working memory** | Arrow IPC | columnar-aligned, Fjall disk and Polars zero-copy interop |
| **Cloud long-term memory Lakehouse** | Lance | embedded vector and inverted indexes, natively supports streaming point-kill retrieval from remote S3, the AI-era columnar standard that replaces Parquet |
| **Massive cold-history hibernation** | Parquet | high-compression cold storage; when data goes into the cold store, one-click conversion from Lance to Parquet dictionary compression |
| **UI ↔ Gateway** | CBOR | cross-language (WASM), self-describing |
| **KV value read-only point query (candidate)** | rkyv | pure Rust mmap deserialization-free, O(1) jump to read a field (see the re-evaluation below) |

### rkyv's Adoption Assessment / Re-Evaluation

After adding rkyv, a re-examination of Aura's layered strategy—**what it fills is the "read-only hot queries on local storage" gap, but its position must be placed correctly**:

**Why it is worth adopting**: for data with frequent read-only point queries such as local working memory/caches (especially the current scenario of storing structs directly), serializing with rkyv + reading via mmap can save the overhead of "deserializing on every access," yielding a benefit different from Arrow IPC—Arrow is columnar-aligned batch, rkyv is row-store one-shot point query.

**Layer-by-layer assessment**:

| Aura layer | Should rkyv be used | Rationale |
|:--|:--|:--|
| Raft control flow and RPC | ❌ does not replace Postcard | Postcard is already sufficient; needs evolution + Varint compression, and rkyv's weak evolution and large size are not advantages |
| Local transactional working memory | ⚠️ depends on the workload | if **batch column access** dominates → Arrow IPC (Polars interop); if **single-record point queries** dominate → rkyv is more suitable (row-store O(1)) |
| Cloud Lakehouse | ❌ | Lance has embedded vectors/inverted indexes, S3 streaming; rkyv has no such capability |
| Cold history | ❌ | Parquet has a high compression ratio; rkyv has neither its compression nor its columnar layout |
| UI ↔ Gateway | ❌ | non-Rust (JS/WASM host) cannot interpret rkyv; CBOR self-describing is the right call |
| **KV value read-only point query** | ✅ **new placement** | if Fjall's value needs repeated mmap deserialization-free field fetch, rkyv is a high-value addition |

**Conclusion**: rkyv should not replace any existing layer, but **fill in a form not previously covered**—"read-only hot queries on pure-Rust local storage." It and Postcard (RPC/evolution), Arrow (batch column analytics), and CBOR (cross-language) each play their part and can coexist. The one thing to be clear about is its division of labor with Arrow: **batch column analytics → Arrow; single-record/hot-path point queries → rkyv; the two are not substitutes for each other**.

### The Core Reasons Postcard Replaces Bincode

In a pure Rust environment, although Bincode has zero metadata overhead, it has two fatal flaws:

1. **Adding a field to a struct breaks it**: during a distributed cluster's rolling upgrade, the new and old nodes' Bincode encode/decode formats are incompatible, and deserialization directly Panics
2. **serde advanced features incompatible**: Fluxora's real-world pitfall—Bincode cannot deserialize `#[serde(tag = "...")]` enums (it calls `deserialize_any`, which bincode does not support), causing all internally tagged enum types to fail deserialization [→ ADR-001](projects/fluxora-decisions-en.md)

Postcard's solution and boundaries:
- `#[derive(Serialize, Deserialize)]` mounts the `postcard-schema` macro, automatically generating field-layout metadata at compile time
- On deserialization, based on the Rust struct's actual changes, it automatically performs "safe skipping (Skip)" and "default-value completion"
- New and old nodes need to pass no network metadata at all, realizing imperceptible rolling upgrades across days
- Underlying Zigzag/Varint compression, with a transport size even lighter than Avro

#### ⚠️ Postcard also does not support `#[serde(tag = "...")]`

Postcard and Bincode belong to the same school (the ultimate lightweight Row format with zero self-description). Its byte stream likewise contains no field-name strings or self-describing Key markers, and therefore also **does not support `deserialize_any`**. Force-feeding a `#[serde(tag = "type")]` enum to Postcard throws a `WontImplement` error.

**The correct answer: switch to the default Externally Tagged enum mode**

```rust
// ✅ Correct: default external tag, variants use numeric indexes (0, 1, 2), not triggering deserialize_any
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub enum Content {
    Text(String),        // tag = 0
    Vector(Vec<f32>),    // tag = 1
    ArrowBlob(Vec<u8>),  // tag = 2
}

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct Brick {
    pub brick_id: String,
    pub timestamp: u64,
    pub payload: Content, // deterministic nesting, zero ambiguity
}
```

**Gateway lazy-unpacking shell**: the gateway layer only needs to parse the routing metadata; the Payload is forwarded zero-copy to the downstream booth with `&'a [u8]`:

```rust
#[derive(Serialize, Deserialize)]
struct GatewayRouterEnvelope<'a> {
    pub route_target_id: String,   // the gateway reads it directly, performs routing
    pub auth_token: &'a str,       // zero-copy borrow
    pub brick_payload: &'a [u8],   // not unpacked at all, forwarded directly
}
```

### The Core Reasons Lance Replaces Parquet

Parquet was born in the Hadoop era and has three innate flaws for modern AI workloads:
1. **Missing vector index**: it does not support HNSW vector indexes; multi-dimensional Embedding retrieval needs external maintenance
2. **Fear of random writes**: the columnar layout is extremely unfriendly to streaming append writes
3. **No inverted text**: full-text search needs an extra indexing layer

Lance's solution:
- embedded HNSW vector index + inverted text index, one format covering multimodal retrieval
- supports microsecond-level streaming point-kill from remote S3 (the underlying magic of LanceDB)
- disk compression ratio on par with Parquet, but crushing it in AI workload performance

## 7. Transport-Layer Channels: gRPC vs WS/WebTransport

Choosing a serialization format only solves "how the bytes are laid out"; half the problem remains in "how the bytes travel"—the channel layer. gRPC is the most typical product of these two layers being sold as a bundle: Protobuf + HTTP/2 + a per-language runtime, sold to you together.

### gRPC's Two Fatal Weaknesses

**Web ecosystem isolation**. gRPC depends deeply on low-level HTTP/2 features (Trailers, etc.), and native browser JS cannot directly issue gRPC requests. The Web side must choose one of two: add a grpc-web proxy layer (inserting an Envoy in the middle for protocol translation), or write two sets of routing in the Core. For an architecture of "Core exposes interfaces, multiple ends connect" (see [The Myth of Visualization §Core and Interface Layering](visual-myth-en.md)), gRPC is equivalent to manufacturing a split at the very place that most needs unification.

**Runtime infrastructure cost**. The IDL compilation chain (.proto → code generation per language) and dedicated debugging tools (generic HTTP tools are blind to Protobuf binary streams and private headers—the browser Network panel, curl, and standard load balancers cannot be used directly). These costs are not traded for performance; they are an ecosystem lock-in tax.

### Channel-Layer Selection: WS / WebTransport

Once the channel is decoupled from serialization, the selection criteria return to first principles: **native browser support + message delimitation + low latency**.

- **WebSocket**: the mature standard for message-style transport, naturally delimited (aligning with the "leave the total-length header to the transport layer" criterion above), full-duplex, full ecosystem coverage. Based on TCP, it has head-of-line blocking, but the impact is limited for local/single-service scenarios.
- **WebTransport**: based on HTTP/3 (QUIC), multiplexed with no TCP head-of-line blocking, natively supported by browsers—the Web side, Tauri shells, and back-end services can all connect directly, with no proxy, and the Network panel can debug it directly. It is WS's evolution direction, with ecosystem maturity still catching up.

### Combination with the Serialization Layer

| Scenario | Channel | Serialization | Rationale |
|:--|:--|:--|:--|
| Core ↔ CLI (local) | stdio / Unix socket | Postcard | defined by the same Rust code, zero metadata; message boundaries delimited by the pipe/socket |
| Core ↔ GUI (browser/Tauri) | WS / WebTransport | CBOR | cross-language (JS parses natively), self-describing, native browser channel |
| Between distributed nodes | WS / QUIC | Postcard | pure Rust end-to-end, compact Varint; evolution left to postcard-schema |

Compared with gRPC: the same architecture under gRPC requires maintaining a .proto compilation chain, adding a proxy on the Web side, and switching to dedicated tools for debugging—three layers of cost bought for Protobuf's performance, while Postcard/CBOR are no worse in performance in their respective scenarios. **Once the channel is decoupled from serialization, not one item in gRPC's bundle is irreplaceable**: its only remaining reason is an existing multi-language microservice organization's established collaboration conventions—that is an organizational constraint, not a technical advantage.

### The Downgrade Path for Local Communication

Communication between the Core and a same-machine GUI does not need a loopback network: localhost loopback still goes through the full IP stack (protocol headers, port management, the firewall face), whereas a Unix socket / named pipe is a shorter path on the same machine—the same byte-stream semantics, one fewer network-stack layer. The criterion follows the same control principle: control local resources yourself and take the shortest path; only cross-machine requires the generality of a network protocol. The same goes for GUI serialization: the TS ecosystem's CBOR library is mature and shares the same understanding with the Core side (CBOR is a standard format, no dialect), so there is no need to introduce a Rust→WASM glue layer for a unified serialization logic—a glue layer is the temptation of "define once, compile on both ends," paid for with an extra build chain and an FFI boundary, and unless measured performance is inadequate, it is not introduced.

**Core principle**: for both channel and format, choose "the native capability of the browser/OS + the lightest runtime," and reject any bundled scheme that demands a "full package"—so it is with gRPC's Protobuf+HTTP2+dedicated toolchain, and so with any "framework ships its own serialization." Only by seeing the boundaries of the physical hardware clearly and tearing off the pointless IDL nesting dolls can the engine unleash its cleanest energy between multi-core CPUs and NVMe.

---

## Cross-References

This document is a comparative analysis of serialization protocols, forming a complete analytical loop with the following documents:

- **[Aura Architecture](aura-architecture-en.md)**: a modern compute-storage-integrated distributed booth engine, adopting a layered serialization strategy.
- **[Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md)**: a full-chain compute-storage-integrated Fjall + Arrow + Polars, including a collision of 7 data formats' underlying byte layouts.
- **[Block Editor Architecture](block-editor-architecture-en.md)**: a Notion-like Block Editor, Yjs CRDT collaboration + Fjall storage.
- **[The Myth of Visualization](visual-myth-en.md)**: Core and interface layering—the architectural context of channel-layer selection (GUI/CLI/HTTP interfaces take different forms by audience).

**The unified first principle**: no technology worship, no swallowing the big pie that open source draws, only looking at the real physical limits of hardware and team productivity.
