# Fluxora Architecture Decision Records (ADR)

> **Languages:** [English](fluxora-decisions-en.md) (primary) · [中文](fluxora-decisions.md)

## ADR-001: CBOR Over Bincode

- **Decision**: Use CBOR for all Kafka queue traffic. Bincode is rejected.
- **Root cause**: Bincode fails on `serde(tag = "...")` (internally tagged enums) because it requires `deserialize_any`, which bincode does not support.
- **Impact**: Affects all deserialization of the `Brick`/`Content` types, not only the wire protocol.
- **Alternative path** (added 2026-06-20): If extreme lightness is the goal, one need not switch formats; instead, switch to the Externally Tagged default enum mode (variants indexed by number, which does not trigger `deserialize_any`). Postcard likewise does not support `#[serde(tag)]`, but can be used safely together with the Externally Tagged mode. See [Serialization Protocol Comparison](serialization-protocol-comparison-en.md#the-core-reasons-postcard-replaces-bincode).

## ADR-002: Zero-RTT Handshake via URL Parameters

- **Decision**: The codec is pinned at handshake time based on the URL `?codec=json|cbor`.
- **Alternatives considered**: First-frame detection (the client sends information, then Greet).
- **Reason for rejection**: Greet carries the base UI layout; delaying it adds 1 RTT and causes a white screen on connect. URL parameters are cleaner and zero RTT.
- **Pitfall**: Do not use `CodecHint` (redundant). Do not delay greet.

## ADR-003: Streaming Merge Strategy (`Vec<String>`)

- **Decision**: Accumulate streaming text as `Vec<String>` fragments, flattened lazily.
- **Alternatives considered**: `String` (push_str) or `&[&str]`.
- **Rationale**:
  - `String`: each token reallocates and copies the entire buffer (O(N²)).
  - `&[&str]`: lifetime hell + memory amplification (references pin the full original fragment).
  - `Vec<String>`: amortized O(1), allows immediate extraction of the payload, discarded after envelope routing.

## ADR-004: Codec Dispatch via Enum (`ActiveCodec`)

- **Decision**: Use `enum ActiveCodec { Json, Cbor }` rather than `Box<dyn Codec>`.
- **Reason**: `dyn Codec` is not object-safe, because `encode<T: Serialize>` is a generic method. Enum dispatch avoids vtable overhead and supports `Clone`.

## ADR-005: AI vs Template Usage

- **Principle**: Minijinja templates for fixed logic; AI for dynamic generation.
- **Atomicity**: Keep `Table` and `SVG` atomic (direct HTML mapping).
