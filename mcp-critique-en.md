# MCP: Over-Engineering of an Application-Layer Protocol

**Conclusion**: In sovereign pure-terminal environments, reject MCP. Adopt CLI + Skill.
**Status**: Solving systems problems with Web-thinking. A regression before UNIX philosophy.

## Executive Summary

Model Context Protocol (MCP) has been packaged by Anthropic and major commercial IDEs (Cursor, Windsurf) as the definitive solution for "AI tool connectivity and dynamic discovery." But from operating systems and distributed systems architecture fundamentals, MCP is an engineering-monstrous design: it rewrites what OS process pipelines already solve, using application-layer JSON-RPC long-lived processes.

This document does not discuss MCP's commercial motivations. The focus is technical: MCP's three claimed advantages all collapse under computer science fundamentals. The CLI + Skill approach is aligned and superior.

---

## I. Tech Stack Path Dependency

Current mainstream AI clients are built on Electron and Node.js stacks. The inertia of this tech path produces a bias: solving systems problems with Web thinking.

A Web engineer's instinct is "spin up a server, run JSON-RPC handshake, listen on a port" — reasonable in Web contexts, but over-engineering for local tool invocation. OS process control, standard signal handling, pipes, and language-level spawn APIs already solve all tool execution needs perfectly. MCP rewrites these native capabilities in JavaScript at the application layer, at the cost of: long-lived process memory overhead, handshake latency, and full Schema loading Token waste.

---

## II. MCP's Real Value: Ecosystem Standardization

MCP has one value this document must acknowledge: **standardization**. Write one MCP server, and all compatible clients can use it. This is real ecosystem value — not "technically superior," but "lower integration cost."

But this doesn't change the fact that MCP is technically over-engineered. Standardization ≠ the only correct implementation — USB standardized peripheral connectivity, but USB-C's alternate mode hell proves standardization itself can be an over-engineering product. MCP's standardization value holds in **multi-client scenarios**; it does not hold in **single-tech-stack pure-terminal environments** — if there's only one Agent framework, there's no need for cross-client standards, and CLI + Skill's lightweight advantage immediately overwhelms MCP's ecosystem advantage.

---

## III. Three Technical Pseudo-Advantages

### Pseudo-Advantage 1: Dynamic Tool Discovery & Schema Constraints

**MCP's claim**: Tools declare their interfaces via JSON-Schema, enabling Agent dynamic discovery and constrained invocation.

**Problem**: LLMs cannot know how to call tools from nothing — Schema declarations are unavoidable. The MCP protocol spec itself supports `tools/list` and `tools/call` separation — Agents can list first, then call on demand. But mainstream MCP clients typically pull full Schemas at initialization; redundant JSON-Schemas severely consume Tokens. This is ecosystem inertia (client implementation habits), not protocol-mandated — but the result is: in practice, MCP's Schema loading is indeed more wasteful than on-demand retrieval.

**CLI + Skill alignment**: Skills can equally specify commands (e.g., `my-skill --schema`) that output strongly-typed parameter descriptions in milliseconds. The AI framework loads progressively on demand, routing to the corresponding Skill and fetching Schema only when needed — far more efficient than MCP's full loading. This is an information-theoretic advantage: on-demand retrieval > broadcast.

### Pseudo-Advantage 2: Stateful Long Connections Avoiding File Lock Contention

**MCP's claim**: Stateful long-lived connections avoid lock conflicts from concurrent file writes across instances.

**Problem**: MCP integrates concurrency safety, but the integration method (in-memory lease locks) is inferior to using the underlying DB directly. Each IDE window forks its own MCP instance; they don't share memory. Under high concurrency, you still rely on underlying database lock mechanisms (e.g., SQLite WAL mode, row-level locks).

**CLI + Skill alignment**: Stateless CLI + stateful DB. High-concurrency safety is delegated to mature embedded database engines (SQLite / LibSQL) — no need for the upper-layer protocol to reinvent locks in memory.

### Pseudo-Advantage 3: Async Long Tasks & Streaming

**MCP's claim**: Asynchronously execute long tasks, stream results back, without blocking the Agent main loop.

**Problem**: Having the LLM send a `memory_save` message and immediately continue output is essentially background async execution. This doesn't warrant a protocol.

**CLI + Skill alignment**: Use the language's spawn API to start child processes; the framework fully controls the lifecycle — can wait, can kill, can read stdout for streaming returns. This is standard language runtime capability, not something MCP needs to "bestow."

### Supplement: Reverse Control & Mid-Execution Interaction

**MCP's claim**: CLI's unidirectional Standard I/O cannot request user interaction mid-execution (e.g., confirmations, forms).

**Problem**: Spawned child processes can return structured messages to the framework via stdout (e.g., `{"action":"confirm","message":"..."}`), the framework parses and shows a dialog to the user, user selects and it's written back via stdin. Bidirectional interaction doesn't require a persistent JSON-RPC connection.

---

## IV. First Principles: If Skill Came First

Strip away all existing ecosystem — no MCP, no Skill, no tool protocol. Design Agent infrastructure from scratch.

**What would you design first?**

Option A: Communication protocol (how to connect to external tools)
Option B: Capability model (what the Agent knows, what can be composed)

The answer is obviously B. You must first define "what capability is" before deciding "how to expose it." Communication protocol is a **downstream product** of the capability model — once you have content to communicate, protocols emerge naturally.

MCP's problem is **inverting this order**: it first defines the communication method (JSON-RPC, long-lived processes, Schema), then forces tools to adapt to this protocol. This results in:
- Tools forced to wrap as MCP Servers (even when they're just CLI scripts)
- Capability descriptions are constrained by the protocol's Schema format (JSON-Schema is not the optimal form for capability description)
- Execution model locked to long-lived processes (even when a tool only needs to execute once)

If Skill came first, MCP has no place. Because:
- Skill already defines capabilities (`skill.md` + Schema + executor)
- Skill already specifies discovery (`--schema` on-demand loading)
- Skill already solves execution (spawn / WASM Sandbox)
- The remaining "cross-client communication" is **optional protocol adaptation**, not core architecture

### How Skill Handles Scale

"10000 Skills can't be loaded" — this objection assumes the same full-loading model as MCP. Skill architecture natively supports scale management:

- **Search**: Embedding-based semantic retrieval — Agent describes intent, returns Top-K relevant Skills
- **Tree organization**: Skills are composable — `release-management` = `git` + `docker` + `kubernetes` + `slack`, hierarchical not flat
- **On-demand loading**: Only activate Skills needed for the current task; the rest don't consume context

MCP cannot do these. MCP's `tools/list` returns a **flat Schema list** — no hierarchy, no composition, no semantic retrieval. The larger the scale, the more infeasible MCP's full loading becomes, while Skill's search + composition + tree structure becomes more valuable.

[graph-memory](graph-memory.md)'s **emergent Skills** further illustrate this: tool usage data naturally clusters into Skill boundaries — Skills are not artificially defined, they emerge from usage data. As Skills evolve from manual definition to automatic clustering, scale management is not a question of "can you load them" but "how to efficiently retrieve and compose them" — this is Skill architecture's native capability, which MCP lacks.

---

## V. Serverless & Runtime Evolution

MCP's execution model is a long-lived Server: Agent → MCP Server (long-lived process) → Tool.

This contradicts Serverless philosophy. Serverless's core is **on-demand execution, exit when done** — exactly what CLI + Skill natively provides: spawn a child process, execute, exit. MCP regresses this already-correct model back to a long-lived service.

**Execution model evolution**:

```
Current MCP                 Serverless (ideal form)
Agent                       Agent
  ↓                           ↓
MCP Server (long-lived)     Skill Runtime
  ↓                           ↓
Tool                        Sandbox (WASM / Container / Process)
  ↓                           ↓
(stays running)              Execute → Exit
```

WASM is the ideal form of this path:
- **Sandbox isolation**: Each Skill execution environment is independent, permissions controllable
- **Sub-millisecond cold start**: Faster than process spawn
- **Cross-language**: Python / Rust / Go compile to WASM, unified Runtime
- **Portable**: Same `.wasm` runs locally, in cloud, at edge

Future Skill distribution format:
```
skill.yaml          ← metadata + permission declarations
skill.wasm          ← execution body
metadata.json       ← Schema + version + dependencies
```

This is not theoretical speculation — CLI + Skill's spawn model is already the local equivalent of Serverless. WASM Runtime simply extends this pattern from "local processes" to "cross-environment sandboxes."

---

## VI. MCP vs CLI + Skill Comparison

| Dimension | MCP | CLI + Skill |
|:---|:---|:---|
| **Architecture** | JSON-RPC long-lived process + port listener | spawn child process, exit when done |
| **Startup latency** | Handshake + full Schema pull | Millisecond cold start |
| **Schema loading** | Clients typically pull full load | Progressive (`--schema` on-demand) |
| **Token consumption** | High (redundant JSON-Schema) | Low (on-demand routing) |
| **Concurrency safety** | In-memory lease locks (inferior to DB) | Embedded DB (SQLite WAL) |
| **Async capability** | Protocol-native | spawn API (lifecycle controllable) |
| **Dependencies** | Mainstream impl depends on Node.js + ports | Zero dependencies (single binary / Python script) |
| **Portability** | Protocol-bound, cross-client adaptation needed | Direct directory copy |
| **Sovereignty** | Protocol-bound, tools on server | Local files, tool sovereignty with developer |
| **Scale management** | Flat list, full loading | Search + tree + composition, on-demand loading |
| **Ecosystem compat** | ✅ Cross-client standard | ❌ Per-framework implementation |

---

## VII. Skill Implementation Language Selection

Skill implementation language choice is not a technical aesthetics problem — it's a **friction problem in autonomous evolution scenarios** — AI needs to read, write, and modify Skill code; language choice directly impacts friction level.

| Language | Advantage | Disadvantage | Use case |
|:---|:---|:---|:---|
| **Python** | AI extremely familiar, environment ubiquitous, read/write/modify friction lowest | Average performance | **First choice** — Skill bottleneck isn't execution performance |
| **Rust** | Best performance, safe | Heavy environment, slow compilation, binary opacity hinders AI review, high autonomous evolution friction | Performance-sensitive core Skills |
| **Node / Bun** | Large ecosystem | Fragmented packages, heavy npm downloads, troublesome deployment, marginal advantage | Not recommended |
| **Go** | Fast compilation | No performance advantage for Skill scenarios, less reliable than Rust | Not recommended |
| **Bash** | — | All flaws (no types, no error handling, injection risk) | **Forbidden** |

**Conclusion**: Skill first choice is Python. The reason isn't "Python is best" but "Python has the lowest friction in autonomous evolution scenarios" — AI writes Python with the highest accuracy, reviews Python code most competently, Python environments are most ubiquitous on servers and dev machines. Rust performs better but autonomous evolution requires AI to repeatedly read/write/modify code; compilation cycles and binary opacity are disadvantages in this scenario.

> This selection logic aligns with [AI Agent Selection](ai-agent-selection.md): Agent bottleneck isn't execution performance, it's compound interest accumulation. Skill uses Python not because it's fast, because AI modifies it fast.

---

## VIII. Security Design

### Framework-Level Security: Framework Constructs Commands

After receiving LLM's structured output (tool name + parameter list), the framework constructs commands using its own language's spawn API — LLM doesn't directly assemble command strings, framework controls execution. This is not fundamentally different from MCP's security model — MCP also has LLM providing parameters, framework/Server executing. Injection risk is identical on both routes.

### Skill-Level Security: Supply Chain Problem

After a Skill starts, it has access to the full capabilities of its implementation language — Python's `subprocess`, Rust's `std::process`, including spawning background processes, reading/writing files, network requests. Skill security is a **supply chain problem** (do you trust this Skill? Is its source reliable?), identical to MCP Server trust issues. MCP provides no additional security guarantees at the security layer.

---

## IX. Selection Landing Point

**Must work in commercial IDEs** (Cursor / Windsurf): Forced to accept MCP, because these IDEs don't open Shell permissions. The most mature ecosystem for cross-client shared memory at this point is [agentmemory](https://github.com/rohitg00/agentmemory) (24k★, TypeScript, adapts commercial IDEs via MCP protocol). This is a political compromise — accepting MCP's overhead for ecosystem compatibility.

**Sovereign pure-terminal environment** (Claude Code / independent CLI Agent): Hold firm on CLI + Skill. Tools and memories persist as Python scripts and plaintext / LibSQL in project directories (`.agent_skills/`), tracked by Git, with traceability and environment isolation. Migration is direct directory copy. Memory layer can be externalized (see [AI Agent Selection](ai-agent-selection.md) for memory externalization architecture), no MCP binding needed.

---

## Cross References

- **[AI Agent Selection](ai-agent-selection.md)**: CLI Agent + externalized memory layered architecture, memory layer independent of Agent. Skill uses Python not because it's fast, because AI modifies it fast.
- **[Agent Compound Interest](agent-compound-interest.md)**: Tool sovereignty local = compound interest to user; protocol binding = platform lock-in. Skill direct directory copy = zero migration cost = compound interest unbroken.
- **[agentmemory](agent-memory.md)**: Memory layer via MCP compromise, low-frequency operation overhead acceptable.
- **[Karpathy's AI Coding Methodology](karpathy-ai-coding-methodology.md)**: SYSTEM.md / TODO.md are Skill prototypes; document-driven is engineering adaptation against LLM statelessness.
- **[graph-memory](graph-memory.md)**: Emergent Skills — tool usage data naturally clusters into Skill boundaries, scale management via search and composition, not full loading.
