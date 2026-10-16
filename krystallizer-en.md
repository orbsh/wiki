# Krystallizer — The Agent Memory Core

> **Languages:** [English](krystallizer-en.md) (primary) · [中文](krystallizer.md)

*An embedded memory system combining session control + atomic fact graph + vector-landing retrieval: mechanisms and implementation design. For the conceptual positioning of the property graph, see [Graph Memory](graph-memory-en.md); for the general memory selection analysis, see [Memory Architecture](agent-memory-en.md); for the Surface architecture semantics, see [Stateless Agent Architecture](stateless-agent-architecture-en.md).*

Krystallizer is a memory system extracted from skillforge `src/mem`, built on the [KV Storage Engine](kv-storage-engine-en.md) architecture: session-control primitives (branch / tail / summarize), an atomic fact graph (append-only KDL facts + weights), and vector landing from a built-in local model + graph traversal (ADR-0003 Update 2026-10-14 withdrew arroy / BM25 / RRF). An embedded Rust core (mem-core) + Python bindings (PyO3), with zero LLM dependency and zero transport dependency.

## Dual Interface Surfaces: Session Control and Memory

The memory system exposes, and only exposes, two interface surfaces (mechanism/policy separation — the core only answers "how", "when" belongs to the caller):

### Interface Surface 1: Session Control (the passive side · short-term)

Three composable primitives, explicitly driven by the agent:

```
branch(fork_point)    Fork a session from any prefix point. Checkpoints are immutable (prefix model);
                      forking is zero-copy: branches share the checkpoint prefix, incremental messages go their own way.

tail(prompt)          Single-turn branch. Fork → inject the prompt at the tail → run that turn → discard the branch.
                      The main session history is unaffected (history purity guarantee, holds structurally).

summarize(...)        Composite operation: inject a compression instruction (via tail) → collect the summary →
                      truncate the compressed messages → write a new checkpoint. The session contracts.
```

Tail-prompt injection and summarization are both only branch operations: tail is a single-turn branch, summarize is injection + truncation. One mechanism, no special cases.

**Trigger policy: the decision lives in Krystallizer, the initiation in Gravity, the execution runs on a parallel bypass.** The core provides primitives; the compression decision defaults inside Krystallizer (assembling the view at fetch time to judge the watermark — it holds all context and thus has the fullest information); on reaching the watermark it marks a bypass; Gravity issues the bypass's LLM request in parallel with the main branch when U_n arrives (same-prefix fork, shared cache), the summary lands in the turn gap before U_{n+1}, and the prefix switch is a background transaction that does not occupy the conversation's critical path. Time-interval triggers with no turn to attach to are initiated client-side (CLI idle tick, Prism timed delivery, Aura `@cron` booth). The old automatic trigger degenerates into one default policy, not hard-wiring — callers can still disable it, tune it per session, and trigger at other times; this is the lesson from the old `ConversationMemory` hard-wiring `checkpoint_threshold` into `get_context()`, which left callers with no control at all. The kernel of the lesson: **policy must be caller-controllable configuration, not private behavior hidden in the storage layer** — bypass compression must likewise make the threshold configurable and disableable, with the configuration surface at store construction time.

**LLM calls belong to the caller (Gravity); Krystallizer has zero LLM dependency.** The compression bypass's LLM call is implemented by Gravity: the context cache is bound to the model (and account), and only the side that initiates the inference knows which model to use and which account to go through — Krystallizer does not need and should not know these. The compression decision is in Krystallizer (judging the watermark and marking the bypass at fetch time), while the bypass request is issued by Gravity in parallel with the main branch, hitting the cache naturally. The only exception is the internal vector embedding: Krystallizer calls the embedding model itself, because embedding spaces of different models are mutually incompatible — writes and retrieval must use the same model to compute vectors, and this binding is a data-correctness constraint, not a policy choice. The criterion for both boundaries is the same: **calls bound to model identity belong to whoever holds the session truth** — inference caching binds Gravity's model account, the embedding space binds Krystallizer's storage internals.

### Interface Surface 2: Memory (the active side · long-term)

Memory is not a per-mode accessory — it is independent of the consumption stance, every agent has it, and it is offered AS TOOLS (any other agent can use it the same way). The axis is who initiates:

- **Passive (short-term)** — the current loop's context: the channel log and its projection. The loop fetches and appends; the model never touches it.
- **Active (long-term)** — facts outside the context: `store` / `search` / `forget` over flat memory and graph facts, queried by the LLM AS A TOOL. Retrieved context is only **injected**: alongside the session, never replacing it.
- **Initialization (persona / style)** — graph-derived: k10r assembles it through a traversal from the user node and injects it at the HEAD of the context; the assembly is a k10r internal detail, and the caller only receives it inside the projection. Not ADR-0009's `kind=profile` — that one is who a person in a channel is, not the agent's own style.

The old pair "full control mode / assist mode" is retired: it reads as "full mode makes assist redundant", which is false.

## Session as Data (Stateless Agent)

Architectural corollary of the passive (session) side: the agent loop degenerates into a pure function.

```
f(session, user_input) -> session'
```

The agent fetches the complete session by `session_id` → executes → writes the new session back. The loop's entire responsibility toward the session is to **append** (including tool-result writes): it never rewrites history, never assembles context from scratch, and holds no session state between calls. All function calls — memory calls included — are ordinary records to the loop; there is only one call mode.

This is isomorphic to Aura's "stateful API, stateless operations" stance: session persistence and recovery sink into the memory system, booths/executors are stateless between calls, and scale-to-zero comes for free — `on_sleep`/`on_wake` degenerate into two actions, store and fetch.

**View-layer trimming is a separate mechanism, not loop logic.** Trimming the function-call format out of the serialized view happens on the memory system's serialization path. The storage layer always holds the complete, untrimmed session; the view is trimmed, the storage is not, and trimming is strictly read-side only — any write-path trimming would fork "the stored session" and "the session the agent sees" into two truths.

**Dissolution of memory calls (rejected)**. We once evaluated replacing memory-recall call records in the view with the recalled context. Rejected: the gain is just one saved round trip, while the cost is that the loop must know which calls are memory calls and rewrite context mid-turn — inserting records is the data layer's job, but rewriting mid-turn context becomes the loop's private chore, giving one class of function call privileged treatment inside the loop and breaking the append-only invariant. Whether or not dissolution is adopted, putting the behavior in the view layer holds; dissolution itself is not worth it.

**The cache cost of dissolution is bounded (for reference)**. Prefix caching matches the longest common prefix per request, not the whole session at once. Dissolving the last two records (tool_call + tool_result) into one merged record only changes the tail; the hundred records before it are byte-identical and the cache hits as usual. The per-turn loss is limited to the replaced tail (a few KB of recomputation), the same mechanism as branch pruning: append + local tail change, with the prefix held stable. "Rewriting history invalidates the whole cache" is a misunderstanding — the rewrite point is just the tail.

**The agent quartet**: entry (Prism), loop (Gravity), memory surface (Krystallizer), executor (Effector). The loop contracts to: fetch session → run turn → store session. For the complete architecture, see [Stateless Agent Architecture](stateless-agent-architecture-en.md).

**Costs and transfers**:

- Larger transfer volume: every call transfers the complete session (a few dozen entries on average, with the function-call format already trimmed in the view). An LLM request is full context anyway; this is just serialization cost, not a scalability problem.
- State has not disappeared — it has moved. The memory system henceforth carries persistence, concurrent writes, and read-time trimming, becoming the sole holder of session truth; multi-writer semantics and read-time compression are its problems, but they do not block this design.

## Write-Read Loop

```
Write: raw input → [Stage 1: normalize + classify] → [Stage 2: extract atomic facts] → Fjall (working memory) → weight accumulation
                                                                                       ↓ (weight > threshold)
Read:  Fjall (long-term memory) → vector landing + graph traversal → return sorted by weight
       ↓ (weight < threshold)
Cold:  S3 → RAG retrieval when needed
```

This aligns exactly with the lifecycle of the dual-track memory architecture. The new element is: **weight determines where an atomic fact sits in the dual track**, rather than the simple "new = hot, old = cold".

### Three Extraction Approaches (Evolution Record)

#### Parallel Extraction Mechanism (⚠️ Obsolete, replaced by the Prefix Checkpoint approach)

> This approach requires modifying the Agent (instructing the LLM in the system prompt to output both a reply and a function call extracting atomic facts); it has three problems: (1) it requires modifying the Agent framework; (2) the function call may disturb the LLM's attention on the reply to the user; (3) it does not consider the context cache (Context Cache) — every turn introduces new function-call tokens, and the cache hit rate is low.

Memory updates happen in the background through the LLM's parallel function calls. The model outputs a set of tool calls in one response, and Hermes parses and executes them asynchronously. The user sees a fluent text reply while the system completes multi-dimensional state updates in the background — no need for a single massive function; the model dynamically orchestrates fine-grained tools.

#### Pure Append Instruction (⚠️ Obsolete, replaced by Prefix Checkpoint)

After the threshold is reached, a compression instruction is appended to the end of the prompt, and the LLM directly outputs checkpoint + memories (not through a tool call).

```
Turn 1..N: normal conversation, [checkpoint] + increments appended one by one to the prompt, context cache keeps hitting
              ↓ threshold reached (100 entries)
Turn N+1:  append "analyze the above conversation, output checkpoint and memories" to the end of the prompt
           → the LLM outputs the result directly, the instruction itself is not written into the session
Turn N+2:  new checkpoint + new increments, start over
```

Advantages: the LLM's attention is fully focused on the compression task (it need not answer the user at the same time), so extraction quality may be higher.
Problems: (1) it requires changing the Agent loop (recognizing special output and handling it); (2) compression and answering are split, taking two LLM calls; (3) the compression timing is not under the memory system's control.

#### Checkpoint Compression (Prefix Checkpoint, current approach)

Replaces the two approaches above. Core idea: compression is neither a separate cache-less LLM call nor something mixed into the answering turn — it is a **parallel bypass forked from an immutable prefix**. When the next user question arrives, the main branch answers as usual; at the same watermark, the compression instruction is issued in parallel as a bypass request, with the prefix KV shared by both branches and the bypass paying only its increment. The summary lands before the next user request (in the typing gap), and the prefix switch is a background transaction — the user perceives nothing.

**Four-step closed loop**:
1. **Intercept and trigger**: the memory system monitors the watermark; on reaching the threshold it marks a bypass on the fetch view, the preceding tokens stay absolutely unchanged, and both the main branch and the bypass hit the context cache 100%
2. **Parallel fork**: Gravity issues two requests at once — the main branch `[H] + [user question]` streams an answer, the bypass `[H] + [compression instruction]` does nothing but compress
3. **Bypass persistence**: the bypass's tool_call (memory_store + memory_checkpoint) is recognized and handled by Krystallizer at append time — the summary goes into the checkpoint, facts into the graph, and old messages are truncated; the output does not enter the conversation stream
4. **Gap switch**: before U_{n+1} arrives the session switches to `[S] + [recent segment]`, and S becomes the new stable prefix

```
Turn 1..N: normal conversation, [checkpoint] + increments appended one by one to the prompt, context cache keeps hitting
              ↓ threshold reached (100 entries)
Turn N issues both at once:
  main branch: [H] + [user question X]             → normal answer Y (critical path)
  bypass:      [H] + [compression instruction]     → tool_call(memory_checkpoint) → summary S
              ↓ the bypass lands before U_{n+1} arrives (typing gap, more than enough)
Turn N+1:  history becomes [S] + [X] + [Y] + [U_{n+1}], all hits, start over
```

**Uncompressed content stays in the session**: messages within the threshold have not yet been compressed into a checkpoint or extracted into long-term memory, but the raw content is preserved intact in the session_messages table, and the Agent sees it directly through the incremental messages. It is not "lost" or "lagging", merely not yet compressed — the raw data is always available.

**History purity**: the compression instruction exists only in the bypass request's prompt; the main branch's user message is never modified and never written into the session. The session always holds `[checkpoint] + [user's original message] + [Agent reply]` — under the bypass model this holds structurally: compression is simply not in the main branch's prompt.

**Advantages**:
- No change to the Agent loop — the bypass flows back through the standard tool call, fully under the memory system's control
- Focused attention — the bypass only compresses, not dispersed by analyzing the user's question (the drawback of the old mixed-into-turn form)
- Imperceptible user experience — answer latency = an ordinary turn; the queueing delay of "compression must wait for the model to free up" in the old serial form is eliminated by parallelism
- Cache hits naturally — the two branches share the prefix KV (inference services naturally support prefix reuse for concurrent requests with the same prefix), each paying only its increment

**Approach evolution comparison**:

| Dimension | Parallel extraction (old) | Pure append instruction (old) | Mixed into turn (old form) | Parallel bypass (current) |
|:--|:--|:--|:--|:--|
| Agent changes | Needed | Needed | Not needed | Not needed |
| LLM calls | Every turn | One separate compression call (no cache) | Reuses the answering turn | Parallel branches, shared prefix cache |
| Attention | Dispersed | Focused on compression | Dispersed (answer + compression crammed into the same turn) | Focused (the bypass only compresses) |
| User latency | Preemption | Serial while queueing | Preempts the conversation rhythm | Zero (hidden in the turn gap) |
| Control | Agent | Uncertain | Memory system | Memory system |

### Base Strategy: Checkpoint Compression

Under the Prefix Checkpoint approach, compression runs on a bypass parallel to the main branch — when the threshold is reached, the bypass request forks from the immutable prefix and devotes itself to extraction and marking:

```
100 messages → bypass request [H] + [compression instruction]:
  tool_call: memory_store(...)      ← the LLM decides actively, extracting long-term memory (flat or graph facts)
  tool_call: memory_checkpoint(...) ← executed by the Agent framework, pure DB operations
  (the main branch answers the user as usual at the same time, neither waiting for the other)
```

#### One Bypass Call, Two Layers of Output

The core idea is unchanged: one LLM call produces two layers of output at once. Only the implementation changes from "outputting JSON directly" to "through a tool call", and the carrier changes from the answering turn to a dedicated bypass:

| Dimension | Old approach (CheckpointCompressor) | Mixed into turn (old form) | Current approach (parallel bypass) |
|:--|:--|:--|:--|
| Trigger | Separate LLM call | prompt tail prompt → answered along with the answering turn | prefix marking → parallel bypass |
| Output | LLM outputs JSON directly | answer + tool call crammed into the same turn | bypass tool call (memory_store + memory_checkpoint) |
| Cache | Not shared (a new API call) | Shared (same turn) | Shared (forked from the same prefix) |
| Attention | Focused | Dispersed | Focused |
| Long-term memory | LLM extracts during the compression call | LLM extracts in the tool call | LLM extracts in the bypass tool call |

All three try to produce two layers from one call; the current approach gets both: the cache hit of mixing into the turn + the focused attention of the pure append instruction, with user latency at zero.

#### Extraction Format Evolution

| Stage | Extraction format | Output |
|:--|:--|:--|
| Current (Phase 2.5) | flat | `{content, category, importance, tags}` |
| Future (graphified) | graph facts | `{subject, predicate, object, category}` |

The extraction mechanism is unchanged (the LLM calls the `memory_store` tool), only the output format changes from flat to structured.

**Serialization and visualization**: the authoritative format for atomic facts is **KDL** — one node per `fact`, with fixed `head`/`rel`/`tail` role child nodes:

```kdl
fact {
  head  <type> <entity-value> [head_props]
  rel   <relation> [relation_arg] [relation_props]
  tail  <type> <entity-value> [tail_props]
}
```

- `head`/`tail`: arg0 = type label (lowercase single token), arg1 = entity value (quoted only if it contains spaces); node props = the entity's attributes
- `rel`: arg0 = relation (uppercase with underscores); node props = the relation's modifiers (time / condition / cause / strength)
- **Empty props are not written**; the type vocabulary is open

Example (`head_props`/`relation_props`/`tail_props` all present + a rule/logic-type relation):

```kdl
fact {
  head  project "Galaxy"  priority=high
  rel   IMPLIES reason="controlled write amplification, preserving the causal chain"  confidence=0.9
  tail  architecture "event sourcing"  status=chosen since="2026-08"
}
```

KDL converts deterministically to Mermaid and Cytoscape.js rendering (no LLM involvement needed; the bracket `head[TYPE]RELATION[props]tail[TYPE]` is only for inline display). Mermaid uses a dagre hierarchical layout (good for checking whether atomic-fact extraction is correct), and Cytoscape.js uses a force-directed layout (good for checking clustering effects). The same input, two outputs.

#### Cache Utilization

Prefix Checkpoint's caching advantage: the checkpoint is immutable and can be cached permanently. The incremental part changes every time, but the historical part (the checkpoint) goes through the context cache.

The current approach needs no separate compression prompt — compression is done through the Agent's normal turn, the system prompt + few-shot is managed by the Agent framework, and the memory system does not directly control the prompt cache.

#### Extraction Prompt

The extraction prompt defines the atomic-fact output format:

```
You are a conversation compressor. Analyze the following conversation history and output two parts:

1. checkpoint: summarize the core content, key decisions and current state of this conversation in 2-3 sentences
2. facts: extract information worth remembering long-term, in the format:
   {"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}
```

**Dependency**: facts depend on the checkpoint — the LLM first understands the conversation and produces the summary, then extracts atomic facts based on the summary. In the early approach the two were independent outputs (no dependency); now that they are merged into a single call, the dependency arises naturally: the checkpoint is the global understanding of the conversation, and facts are the structured knowledge extracted from it.

This is a prompt design inside the `memory_store` tool and does not affect the overall Prefix Checkpoint flow.

### Coding Scenario: Design Documents as a Codebase Checkpoint

M asked "is there a better way for the AI to understand our code better, so it doesn't have to search every time", and O answered "after every change, have it summarize/update the design doc, because the earlier part is cached and the marginal cost is low, so next time there's no need to read a lot, and when modifying it is very precise".

This is not a tip — it is a concrete instance of the computation-timing spectrum in the coding scenario.

**Without a design doc (read-heavy)**: each session the AI re-reads the code → understands → executes → forgets → re-reads next time. An O(n) code scan, whose understanding quality depends on code readability and the context window.

**With a design doc (write-heavy)**: write the design doc once (understanding + structuring), then subsequently read the cached design doc → execute precisely. High first-time cost, low subsequent cost.

**Design doc = a codebase Checkpoint**:
- **Immutability**: unchanged between two modifications → cacheable (the same physical property as a checkpoint summary)
- **Captures decisions**: not just the code structure, but "why it is designed this way" → precisely guides future modifications
- **Incremental updates**: after changing the code, update the affected parts of the doc → O(1) maintenance cost

**Diminishing marginal cost**: under the prompt caching mechanism, once the design doc enters the cache, the computation cost of those tokens in subsequent calls approaches zero. "The earlier part is cached and the marginal cost is low" — only newly added code changes need incremental processing.

**Precision gain**: "when modifying it is very precise" — with a design doc, the AI knows what to change, why to change it, and what the change will affect. Without a design doc, the AI must reverse-engineer the architecture from the code, and can easily miss dependencies or misread design intent.

**Isomorphism with checkpoint compression**:

| Dimension | Conversation checkpoint | Design doc |
|:--|:--|:--|
| Write timing | Compressed every N messages | Updated after every code change |
| Content | Conversation core content + decisions | Code architecture + design decisions |
| Caching property | Immutable, cacheable permanently | Unchanged between two modifications, cacheable |
| Read method | Injected into the Agent prompt | Injected into the Agent prompt |
| Marginal cost | Approaches zero (cache hit) | Approaches zero (cache hit) |

The two are different instances of the same pattern: **invest organizing cost at write time, enjoy zero matching overhead at read time**.

## Storage Implementation

For storage selection, key encoding and retrieval implementation, see [KV Storage Engine](kv-storage-engine-en.md) (property-graph encoding patterns, delta append eliminating read-modify-write, secondary index update strategies, WriteBatch transactions, vector hibernation/loading lifecycle). Retrieval: a built-in local model (in-process, no API, no key, model locked with the data) produces the vector that lands the query, then graph traversal expands the neighborhood and weight ranks the result; a second, exact entrance is the entity name/alias point lookup (the graph's own identity face, not a third mechanism). The tokenization path (jieba + inverted index + BM25 + RRF) and arroy are withdrawn together — ADR-0003 Update 2026-10-14.

**Storage hosting is not split by runtime form: k10r holds its own Fjall directory** (ADR-0007 as revised 2026-10-10). The split by runtime form (Aura booth / wasm sandbox + host-side NestStorage executor with thrown-up frames) is withdrawn — k10r is not an aura citizen: it registers no booth, is not spawned by aura, occupies no ns; "one directory on disk" holds regardless of who calls it. Schema semantics (derive, Table/EdgeTable, WriteBatch framing) run over static OKM derive; okm-dynamic is not needed.

## skillforge Implementation Status

### Current Approach (Phase 2.5)

Slightly right of the middle of the spectrum — higher retrieval quality and more cache-friendly than agentmemory, lighter to deploy and lower LLM cost than cognee.

```
write-heavy ─────────────────────────────────── read-heavy

cognee            skillforge         agentmemory
(knowledge graph) (current impl)     (trajectory compression)
SurrealDB graph   SurrealDB KV       SQLite + vector
```

#### Surface

| Decision point | Choice |
|:--|:--|
| Trigger timing | Threshold trigger (100 entries) + manual Agent invocation |
| Injection method | checkpoint + increments (additional_input injection) |
| Framework adaptation | _AgnoAgentWrapper (three-step call: push → get_context → write) |

#### Engine

| Decision point | Choice |
|:--|:--|
| Extraction | flat KV (LLM two-layer output: checkpoint + memories) |
| Storage | SurrealDB (session_messages + session_checkpoints + memories) |
| Retrieval | HNSW + BM25 + RRF three-way fusion |
| User isolation | Working memory: `user_id` + `session_id` two-key query; long-term memory: `user_id` query |

**User isolation design**:
- Working-memory tables (`session_messages`, `session_checkpoints`) contain a `user_id` field, supporting session isolation across multiple clients for the same user
- The long-term memory table (`memories`) contains a `user_id` field, supporting knowledge accumulation across sessions
- `memory_search` searches both `memories` and `session_checkpoints`, and the result carries a `source` field distinguishing the origin

#### Core Interface

Three: `store` / `search` / `forget`. The auxiliary interfaces `list_all` (export/backup) and `get_stats` (dashboard/ops) do not participate in the Agent conversation flow.

| Method | Purpose |
|:--|:--|
| `store(content, category, importance, tags)` | Store a memory (embedding generated inside SurrealDB) |
| `search(query, top_k)` | Hybrid retrieval (memories + session_checkpoints) |
| `forget(memory_id)` | Delete a memory |

#### Key Design Decisions

| Decision | Reason |
|:--|:--|
| LLM unaware within 100 entries | Does not bloat the prompt or break the context cache |
| Checkpoint immutable | Fixed after writing, cacheable permanently |
| No dependency on the framework MemoryManager | Black-box extraction is uncontrollable, framework coupling, an extra LLM call per turn |
| Embedding generated inside SurrealDB | Zero transfer and zero storage on the Python side |
| No expiration cleanup | Storage is not the bottleneck; vector retrieval naturally ranks irrelevant old memories behind |

#### Framework Migration

`_AgnoAgentWrapper` is the only part coupled to the framework (~20 lines, three-step call). WorkingMemory, MemoryManager and MemoryTools are fully reused. Migration cost: rewrite the adapter layer (~20 lines).

---

## Graphification Evolution

```
Current:  flat KV + vector/BM25/RRF
        ↓
Evolved:  graph triples + vector/graph traversal/RRF
```

### Extraction Changes

The second layer of the LLM's two-layer output changes from flat memories to graph triples:

```
Current output:
  memories: [{"content": "...", "category": "fact", "importance": 3, "tags": [...]}]

Graphified output:
  triples: [{"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}]
```

The pattern of two-layer output from one LLM call is unchanged; only the second layer's output format changes from flat to structured.

### Retrieval Changes

Graph traversal is added on top of the existing HNSW + BM25 + RRF:

```
Current: query → vector retrieval + BM25 → RRF fusion
Evolved: query → vector retrieval + BM25 + graph traversal (multi-hop) → RRF fusion
```

### SurrealDB's Support

SurrealDB's multi-model architecture makes the migration natural — graph records and KV records coexist, and retrieval routes by query type. No need to change the storage engine.

### Detailed Design

For the complete graphification design (clustering strategy, weight system, two-layer model), see [Graph Memory](graph-memory-en.md).

---

## External Corroboration and Comparison: DeepSeek Harness (dsh)

dsh's session model and this system's design converged independently on the same form: an append-only event log as the single source of truth, the LLM message history **derived** from the log by `deriveMessages()` rather than stored, and compression as a view replacement on top of the log (surface replace), with not a single original event deleted. Two design points can be directly corroborated or absorbed:

- **Cache reuse for the compression call**: when generating a summary, dsh replays the prefix of the compressed request and appends the compression instruction at the tail, making the provider-side KV cache hit — the same cache benefit as this system's bypass compression (dsh is a standalone compression call carrying its own tail prompt; we are a bypass request forked in parallel with the main branch, reusing the same prefix KV).
- **Structural alignment of trimming**: dsh's retention unit is a complete, closed step (tool call + result as a pair); if the cut point falls in the middle of a step it extends to align, and `compactRegion` refuses to split a tool pair. The truncation boundary of compress_task should adopt the same rule, replacing the fixed entry count.

## Cross References

- **[Graph Memory](graph-memory-en.md)**: the conceptual design of the atomic fact graph — computation-timing spectrum, clustering strategy, weight system, graphification Skill.
- **[Graph Memory](graph-memory-en.md)**: read-time resolution of conflicts — append-only facts have no write conflicts, and logical conflicts are adjudicated by weight and subject attribution, not by write-time falsification.
- **[Memory Architecture](agent-memory-en.md)**: the general memory selection analysis — the two layers Surface/Engine, injection methods, comparison of external open-source approaches.
- **[Stateless Agent Architecture](stateless-agent-architecture-en.md)**: the overall component architecture — turn model, parallel compression bypass, Surface architecture semantics, Prism/Gravity/Effector.
- **[KV Storage Engine](kv-storage-engine-en.md)**: storage-layer hosting — property-graph encoding patterns, read-modify-write elimination, secondary index update strategies, WriteBatch transactions.
- **[Tail Prompt Optimization](tail-prompt-optimization-en.md)**: the cache bypass mechanism of the tail prompt.
