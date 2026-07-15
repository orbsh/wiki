# Memory Architecture Design

When chatting with AI, every time you say something, the AI has to re-read all previous conversations from scratch. The longer the conversation, the slower and more expensive it gets.

Our approach: compress old conversations into a fixed summary, then append new conversations directly after it. Since the summary doesn't move (prefix stays constant), the AI doesn't need to re-understand old content — like turning to a bookmarked page in a book without re-reading from page one. This is called **Prefix Checkpoint**.

This mechanism also gives the memory system full control over AI behavior: by appending instructions at the end of the conversation, it drives the AI to call memory tools for storage and organization, without any modifications to the AI itself. See [Surface Design](#ii-surface-design).

## I. Two-Layer Architecture

The memory system is divided into **Surface** and **Engine**. Surface couples with the Agent framework; Engine is framework-agnostic.

```
┌─────────────────────────────────┐
│         Surface (Framework-Dependent)    │
│  Trigger Timing │ Injection Method │ Framework Adaptation │
├─────────────────────────────────┤
│         Engine (Framework-Agnostic)      │
│  Extraction     │ Storage        │ Retrieval            │
└─────────────────────────────────┘
```

| Layer | Responsibility | What Changes During Evolution |
|:--|:--|:--|
| **Surface** | When to capture, how to inject prompt, how to adapt to Agent framework | Change when switching frameworks |
| **Engine** | Conversation→Memory conversion, storage format, retrieval algorithm | Change when switching storage/retrieval strategies |

The value of separation: **existing solutions (agentmemory, cognee) have coupled components where changing any layer requires modifying others**. After separation, each layer can evolve independently — migrating from KV to graph storage only changes Engine, switching frameworks only changes Surface.

### Surface

| Decision Point | Options | Representative Solutions |
|:--|:--|:--|
| **Trigger Timing** | auto hooks (per-turn system auto) / manual calls (Agent judgment) / threshold trigger | agentmemory uses hooks, skillforge uses threshold |
| **Injection Method** | Full injection / top-K retrieval injection / checkpoint + increment | Hermes full, agentmemory top-K, skillforge checkpoint |
| **Framework Adaptation** | Adapter interface (get_session_messages / add_message) | SnapshotSessionDB adapts to Agno |

### Engine

| Decision Point | Options | Representative Solutions |
|:--|:--|:--|
| **Extraction** | flat KV (content + category + tags) / graph triples (subject + predicate + object) | skillforge currently uses flat, graph evolution uses triples |
| **Storage** | SQLite / SurrealDB / Postgres / three-storage separation | agentmemory uses SQLite, skillforge uses SurrealDB |
| **Retrieval** | Vector / BM25 / Graph traversal / RRF fusion / multi-strategy routing | agentmemory vector+keyword, skillforge HNSW+BM25+RRF, cognee 9 strategies |

### Computational Timing Spectrum

The core Engine difference is **when computation happens**:

```
Write-Time Heavy ─────────────────────────────────── Read-Time Heavy

LLM Wiki        KG (Property Graph)  KV Pattern         Vector Pattern
                graph-memory         skillforge         agentmemory
                SurrealDB graph      SurrealDB KV       SQLite + vector

Write: LLM organizes/embeds/topology inference  Write: Simple segmentation/storage
Read: Direct load/traversal                      Read: Matching/ranking/vector search
```

Position on the spectrum depends on usage, not storage product. SurrealDB's multi-model architecture covers the entire range.

---

## II. Surface Design

### Prefix Checkpoint

When chatting with AI, every time you say something, the AI has to re-read all previous conversations from scratch. The longer the conversation, the slower and more expensive it gets.

The traditional approach is "remember the last N messages" — but every time a new message arrives, those N messages change (oldest dropped, newest added), so the AI effectively re-reads everything each time.

Prefix Checkpoint works differently:

1. When conversation accumulates to a certain length (e.g., 100 messages), the AI summarizes it: "User is discussing project architecture, decided to use SurrealDB..."
2. This summary **becomes fixed and never changes**. All subsequent conversations are appended after it.
3. Because the beginning doesn't move, the AI can "skip" already-understood parts and only process new messages — like turning to a bookmarked page without re-reading from page one.

The summary is the "prefix" (Prefix), fixed and immutable; "Checkpoint" is the save point. After each new checkpoint, old messages are cleared and accumulation restarts.

**Why traditional compression can't use cache**: The traditional approach sends 100 messages to a separate model (or makes a separate API call) to generate a summary. This call has no cache relationship with previous conversations — even with the same model, it sees entirely new text, KV cache computes from scratch, cache hit rate 0%.

Prefix Checkpoint is different: those 100 messages are already in cache (used in every previous turn). The compression instruction is just a small piece of new text appended at the end. When the model processes it, 99% of tokens hit cache directly, only the final instruction + user question requires new computation.

```
Traditional Compression:
  [100 messages] → New API call → Full recomputation → Output summary
  Cache hit rate: 0%

Prefix Checkpoint:
  [100 messages (in cache)] + [compression instruction + user question]
  Cache hit rate: ~99%
```

Result: fast, good, and cheap. Fast because of cache, good because it uses the same strong model (not a small one), cheap because most tokens don't need recomputation. And no Agent loop modification needed — compression completes through normal tool calls.

**"Auxiliary model" ≈ Traditional mode**: Agent systems that can configure independent auxiliary models are essentially traditional compression — requiring a separate LLM call, unable to use cache. So-called "fast models" are relative: sending 100 messages to a small model still requires computing from scratch, latency might only improve 30-50%, but quality is much worse. Prefix Checkpoint doesn't even need an auxiliary model — compression is part of the Agent's normal response.

### Memory Control

Prefix Checkpoint is not just a cache optimization mechanism, but a **memory control framework**.

The AI reads the complete conversation before every response. If we want the AI to do something (like "remember the preference just discussed"), we just need to add a sentence at the end of the conversation.

Specifically, when conversation accumulates to 100 messages, the memory system inserts an instruction before the user's next question:

```
[All previous conversation]        ← AI already understood (in cache)
[Instruction inserted by memory]   ← "Analyze the above conversation, extract user
                                      preferences and facts, call memory_store tool
                                      to store them, then answer the user's question"
[User's question]                  ← What the user actually wants to ask
```

After seeing this instruction, the AI automatically calls memory tools to complete storage, then answers the user normally. The user is completely unaware — they just asked a question, the AI answered, but behind the scenes memory organization was completed.

**Control Mechanism**: The Agent is passive — it only sees the prompt and available tools, calling tools according to instructions in the prompt. The memory system achieves full control by controlling two things:
1. **What to inject** — the instruction appended at prompt end (determines what the Agent does)
2. **What tools to provide** — the Agent's available tool list (determines what the Agent can do)

**This allows the memory system to**:
- Control compression timing (inject compression instructions when threshold is reached)
- Control extraction content (instructions specify extracting preferences/facts/decisions)
- Control storage method (tools determine which table to write to, what format)
- No Agent loop modification — all control through prompt + tools

**Analogy**: The Agent is a "executioner without opinions" — it has capabilities (LLM reasoning + tool calls), but no intent. The memory system injects intent through the prompt, and the Agent executes. Similar to an OS "system call": the kernel provides capabilities (syscall), user-space programs decide when to call.

### Trigger Timing

| Method | Mechanism | Advantages | Disadvantages |
|:--|:--|:--|:--|
| **Active Trigger** | Agent calls memory_store when it judges "this is worth remembering" | Precise, only stores valuable | Depends on Agent judgment, may miss |
| **Threshold Trigger** | LLM batch extraction when messages accumulate to N | Low-frequency calls, LLM cost controllable | Not compressed within N messages (but raw messages still in session, Agent can read directly) |

### Active Trigger

Prefix Checkpoint's memory control isn't just for compression — it's also for **active memory**. During conversation, when the Agent judges "this is worth remembering," it proactively calls the `memory_store` tool.

Trigger method is **user request or implication**:
- User explicitly says "remember this" or "always do this in the future"
- User expresses preferences, habits, decisions (Agent judges worth long-term storage)
- Agent discovers important facts or context

The Agent doesn't proactively capture knowledge generated during execution — it uses LLM judgment to filter, only storing valuable information, precise but dependent on judgment.

These are the memory system's two complementary trigger methods:

| Method | Timing | Who Decides | What to Store |
|:--|:--|:--|:--|
| **Prefix Checkpoint** | When threshold reached | Memory system | Conversation summary + extracted preferences/facts |
| **Active Trigger** | During conversation | Agent | What the Agent judges worth remembering |

### Injection Method

| Method | Mechanism | Cache Friendliness |
|:--|:--|:--|
| **Full Injection** | Inject all history every turn | Poor (content changes each turn, KV cache invalidation) |
| **top-K Retrieval** | Retrieve relevant memories each turn | Medium (retrieval results may change) |
| **Checkpoint + Increment** | Immutable checkpoint + recent N messages | High (checkpoint fixed, permanently cacheable) |

#### Checkpoint + Increment Mechanism

Similar to event-sourcing snapshot pattern. Messages are appended one by one to the session_messages table. When reaching threshold, they're compressed into a checkpoint (written to session_checkpoints table), and all subsequent uncompressed messages become the increment:

```
msg_001 ... msg_100    ← Messages appended to session_messages table
              ↓ Reaching threshold (100 messages)
ckpt_0: summary="User discussed project architecture, decided to use SurrealDB..."
              ↓ Written to session_checkpoints table, immutable
msg_101 ... msg_150    ← All increment (all current uncompressed messages)
              ↓ Reaching threshold again
ckpt_1: summary="..."
msg_151 ...            ← New increment
```

**Read Flow**:
Messages are appended one by one to the prompt (not reassembled each time). The LLM API's multi-turn mechanism naturally caches all previous tokens:

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← Previous tokens use KV cache
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: Reaching threshold → Integrate into next Agent turn for compression
Turn N+1: [ckpt_1] + [User question X] + [Agent answer Y]
```

**Write Flow**:
1. Each message appended to session_messages table
2. Check if messages since last checkpoint >= threshold
3. Reached threshold → Mark for compression → On next user question, append compression instruction as normal message to prompt end

**Prefix Checkpoint**: After threshold is reached, on the next user question, the memory system appends a special message (not system prompt) at prompt end:

```
KV cache (unchanged):
  [ckpt] + [msg_101..msg_150]

Newly appended message (only uncached part):
  "First answer the user's question, then analyze the above conversation,
   call memory_store to extract preferences/facts, mark checkpoint"

Agent completes three things in one turn (note the order):
  1. "Y"                              ← Answer user question first (user experience priority)
  2. tool_call: memory_store(...)    ← Then memory operations
  3. tool_call: checkpoint(...)       ← Finally mark checkpoint
```

**History Purity**: Compression instructions are temporary — they only exist in that turn's prompt, never written to session. Session always contains clean history:

```
Stored in session:
  [checkpoint_1] + [User's original message X] + [Agent answer Y]

Not:
  [checkpoint_1] + [Compression instruction + User's original message X] + [Agent answer Y]
```

Even if the prompt modifies the user message to trigger compression (appending instructions), the session must **restore to original content** after turn ends. This ensures:
- History is replayable — anytime session is reloaded, content is real conversation
- No pseudo-instructions — session won't contain residual system instructions like "analyze the above conversation, call memory_store"
- Logical coherence — checkpoint + original message + answer form complete causal chain

**Order matters**: Must answer user question first, then do memory operations. User asked a question — if they see a bunch of tool calls running first, experience is poor. The instruction explicitly requires "answer first, process memory later" — user sees the answer as first output, memory operations are completed "incidentally."

**Why this design**:
- **No Agent loop modification** — Compression through normal tool calls, memory system fully controls
- **No separate LLM call** — Reuses Agent's normal turn, answer + compress share KV cache
- **Cache naturally hits** — History fully cached, only compression instruction + user question are new tokens
- **User experience seamless** — Answers user normally, behind-the-scenes completes compression and memory extraction
- **History auto-cleanup** — After turn ends old messages no longer needed, history becomes `checkpoint_1 + X + Y`

**Why cache-friendly**: Checkpoint is immutable once written, all subsequent turns share the same checkpoint text. LLM API's prompt caching stores checkpoint in GPU VRAM, only the increment changes each time. As increment messages grow and next checkpoint triggers, increment is compressed into new fixed summary, cache hits again.

**Comparison with Agno sliding window**: Agno's `add_history_to_context` reads the last N complete messages from DB each time. As new messages arrive, N's composition constantly changes (old dropped, new added), KV cache invalidates each turn. In checkpoint mode, history is compressed into fixed summary, only the uncompressed increment part changes, cache continuously hits.

### Framework Adaptation

The adapter layer isolates framework differences. Core interface has only two methods:

```python
class SessionAdapter:
    def get_session_messages(session_id) → list[dict]  # checkpoint + all increments
    def add_message(session_id, message)                 # Append message
```

`get_session_messages` returns checkpoint + all subsequent uncompressed messages, length determined by threshold, no limit parameter needed.

When switching frameworks, only rewrite the adapter layer (dozens of lines), Engine completely reused.

---

## III. Engine Design

### Extraction

From conversation to memory conversion, two formats:

| Format | Structure | Use Case |
|:--|:--|:--|
| **flat KV** | content + category + importance + tags | Simple preferences/facts, current implementation |
| **graph triples** | subject + predicate + object + category | Knowledge with relational structure, graph evolution direction |

Both formats can be output simultaneously in a single LLM call (dual-layer output):

```
100 messages → [Single LLM call]
                ├→ Checkpoint summary (injected into prompt)
                └→ flat memories / graph triples (written to storage)
```

#### Extraction Timing: Three Approaches

**Approach A: Per-turn parallel extraction (⚠️ Deprecated)**

Instruct LLM in system prompt to simultaneously output user response and extract triple function calls, completing two things in one response.

Problems: (1) Requires Agent framework modification; (2) Function calls interfere with LLM attention on user response; (3) New function call tokens each turn, KV cache hit rate low. → See [Graph Memory](graph-memory.md) §Parallel Extraction Mechanism

**Approach B': Pure append instruction (⚠️ Deprecated, replaced by B)**

After threshold reached, append compression instruction at prompt end, LLM directly outputs checkpoint + memories (not through tool call).

```
Turn 1..N: Normal conversation, [ckpt] + increment appended each turn, KV cache continuously hits
              ↓ Reaching threshold
Turn N+1:  Prompt end appends "Analyze the above conversation, output checkpoint and memories"
           → LLM directly outputs results, instruction itself not written to session
Turn N+2:  New checkpoint + new increment, restart
```

Advantage: LLM attention completely focused on compression task (no need to answer user simultaneously), extraction quality may be higher.

Problems: (1) Requires Agent loop modification (identify special output and process); (2) Compression and answer separated, two LLM calls; (3) Compression timing not under memory system control.

**Approach B: Prefix Checkpoint (Current)**

Replaces Approach B'. Compression is not a separate operation, but integrated into Agent's normal conversation turn — on next user question, append compression instruction (not system prompt) at prompt end. Agent completes extraction and answer in one turn:

```
Turn 1..N: Normal conversation, [ckpt] + increment appended each turn, KV cache continuously hits
              ↓ Reaching threshold
Turn N+1:  User asks question X
           Prompt end appends: "Analyze the above conversation, call memory_store to extract
                                memory, mark checkpoint, then answer: {X}"
           → Agent one turn: tool_call(memory_store) + tool_call(checkpoint) + answer Y
Turn N+2:  History becomes [ckpt_1] + [X] + [Y], restart
```

Advantages: (1) No Agent loop modification; (2) answer + compress share KV cache; (3) User experience seamless; (4) Compression fully controlled by memory system (through tool calls).

| Dimension | A (Parallel Extraction) | B' (Pure Append) | B (Integrated Agent Turn) |
|:--|:--|:--|:--|
| Agent Modification | Required | Required | Not Required |
| LLM Calls | Per turn | Separate compression call | Reuses normal turn |
| Attention | Distributed | Focused on compression | Distributed (but tool call mechanism ensures) |
| Control | Agent | Uncertain | Memory System |

### Storage

| Backend | Characteristics | Use Case |
|:--|:--|:--|
| **SQLite** | Embedded, zero deployment | Personal tools, local Agent |
| **SurrealDB** | KV + vector + full-text + graph, multi-model | Enterprise services, multi-user |
| **Postgres** | pgvector + SQL + graph backend | Existing PG infrastructure |

### Retrieval

| Algorithm | Mechanism | Determinism |
|:--|:--|:--|
| **Vector (HNSW)** | Semantic similarity | Probabilistic |
| **BM25** | Keyword matching | Deterministic |
| **Graph traversal** | Entity relationship chains | Deterministic |
| **RRF fusion** | Multi-way ranking fusion | — |

RRF (Reciprocal Rank Fusion) is the standard approach for fusing multi-way retrieval results — each way independently ranked, merged by reciprocal rank weighting. Doesn't depend on score normalization, robust.

---

## IV. skillforge Implementation

### Current State (Phase 2.5)

Mid-right on the spectrum — higher retrieval quality and cache friendliness than agentmemory, lighter deployment and lower LLM cost than cognee.

```
Write-Time Heavy ─────────────────────────────────── Read-Time Heavy

cognee          skillforge        agentmemory
(Knowledge Graph) (Current Impl)    (Trace Compression)
SurrealDB graph  SurrealDB KV      SQLite + vector
```

#### Surface

| Decision Point | Choice |
|:--|:--|
| Trigger Timing | Threshold trigger (100 messages) + Agent manual calls |
| Injection Method | Checkpoint + increment (SnapshotSessionDB) |
| Framework Adaptation | SnapshotSessionDB (adapts to Agno session_db) |

#### Engine

| Decision Point | Choice |
|:--|:--|
| Extraction | flat KV (LLM dual-layer output: checkpoint + memories) |
| Storage | SurrealDB (session_messages + session_checkpoints + memories) |
| Retrieval | HNSW + BM25 + RRF three-way fusion |

#### Core Interfaces

Three: `store` / `search` / `forget`. Auxiliary interfaces `list_all` (export/backup) and `get_stats` (dashboard/ops) don't participate in Agent conversation flow.

| Method | Purpose |
|:--|:--|
| `store(content, category, importance, tags)` | Store memory (embedding generated internally by SurrealDB) |
| `search(query, top_k)` | Hybrid retrieval |
| `forget(memory_id)` | Delete memory |

#### Key Design Decisions

| Decision | Reason |
|:--|:--|
| LLM unaware within 100 messages | Doesn't bloat prompt, doesn't break KV cache |
| Checkpoint immutable | Fixed after writing, permanently cacheable |
| Don't depend on framework MemoryManager | Black-box extraction uncontrollable, framework-bound, extra LLM call per turn |
| Embedding generated inside SurrealDB | Zero transmission and storage on Python side |
| No expiration cleanup | Storage not bottleneck, vector retrieval naturally ranks irrelevant old memories lower |

#### Framework Migration

SnapshotSessionDB is the only framework-coupled part (implements `get_session_messages` + `add_message`). MemoryManager and MemoryTools completely reused. Migration cost: dozens of lines of adapter code.

---

## V. Graph Evolution

Engine evolution direction: extraction upgrades from flat KV to graph triples, retrieval adds graph traversal. Surface unchanged.

```
Current:  flat KV + vector/BM25/RRF
          ↓
Evolved:  graph triples + vector/graph traversal/RRF
```

### Extraction Changes

LLM dual-layer output's second layer changes from flat memories to graph triples:

```
Current output:
  memories: [{"content": "...", "category": "fact", "importance": 3, "tags": [...]}]

Graph output:
  triples: [{"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}]
```

The single LLM call dual-layer output pattern unchanged, only the second layer's output format changes from flat to structured.

### Retrieval Changes

Adds graph traversal to existing HNSW + BM25 + RRF:

```
Current: query → vector retrieval + BM25 → RRF fusion
Evolved: query → vector retrieval + BM25 + graph traversal (multi-hop) → RRF fusion
```

### SurrealDB's Support

SurrealDB's multi-model architecture makes migration natural — graph records and KV records coexist, routing by query type during retrieval. No need to switch storage engines.

### Detailed Design

Complete graph evolution design (clustering strategy, weight system, dual-layer model) see [Graph Memory](graph-memory.md).

---

## VI. External Solution Comparison

### agentmemory (24k★, TypeScript + Rust)

**Positioning**: Persistent memory layer across Agents, adapting commercial IDEs through MCP.

| Dimension | Implementation |
|:--|:--|
| Storage | SQLite + vector index (`all-MiniLM-L6-v2` local embedding) |
| Trigger | 12 auto hooks (per-turn auto capture) |
| Retrieval | Vector + keyword (binary) |
| Injection | On-demand retrieval injection (92% token savings) |

**Advantages**: MCP universality (any Agent supporting MCP/hooks can connect); auto hooks zero invasion.

**Limitations**: Bound to MCP protocol (port resident); embedding quality ceiling (small model); captured content already in session (hooks' incremental value limited); no graph structure, no weight system.

### cognee (~22k★, Python)

**Positioning**: Knowledge graph memory platform, ingest any format data to build knowledge graph.

| Dimension | Implementation |
|:--|:--|
| Storage | Three-storage separation (relational + vector + graph) |
| Trigger | Manual remember() |
| Retrieval | 9 strategy routing (graph / vector / lexical / temporal / cypher etc.) |
| Extraction | LLM per-chunk entity + relationship extraction |

**Advantages**: Graph native (multi-hop reasoning); multi-strategy retrieval routing; Postgres unified stack (v1.0); `improve()` + feedback_weight self-improvement.

**Limitations**: LLM cost heavy (one call per chunk); deployment complex (three storage); no fact-level conflict resolution (old and new facts coexist); no organization layer (clustering pre-aggregation, relies on LLM piecing together at retrieval time).

### Comparison Matrix

| Dimension | agentmemory | cognee | skillforge |
|:--|:--|:--|:--|
| **Trigger** | auto hooks (per turn) | Manual remember() | Threshold trigger + manual calls |
| **Extraction** | iii-engine compression (black box) | LLM entity extraction | LLM dual-layer output |
| **Storage** | SQLite + vector | Three-storage separation | SurrealDB (KV + vector + full-text) |
| **Retrieval** | Vector + keyword | 9 strategy routing | HNSW + BM25 + RRF |
| **Injection** | On-demand retrieval | GRAPH_COMPLETION calls LLM | Checkpoint + increment |
| **Cache Friendly** | Poor (changes each turn) | Poor (calls LLM) | High (checkpoint fixed) |
| **Framework Binding** | MCP | Python SDK + MCP | None (adapter layer isolated) |
| **LLM Cost** | Low (compression configurable) | High (ingest per chunk) | Low (only calls once per 100 messages) |

---

## VII. References

### CodeGraph: Code Structure Layer

Local semantic code knowledge graph, auto-syncs graph on code changes, AI queries via direct traversal. On the spectrum it's "write-time heavy" — inotify triggers auto graph building, no manual maintenance needed.

Complementary to memory systems: CodeGraph answers "how is code organized" (what), design documents answer "why is it organized this way" (why), memory systems answer "what did we learn from this code" (experience).

### Pure Memory Layer Technology Landscape (2026 Open Source)

Four-quadrant topology:

```
[Quadrant I: Trace Capture]          [Quadrant II: Episodic Events/Temporal Decay]
  agentmemory (iii-engine)            Mnemosyne (Rust, Ebbinghaus)
  Role: Capture-compression-retrieval  HippoRAG (PPR multi-hop association)
───────┼──────────────────────────┼──────────────────
       │                          │
[Quadrant III: Semantic Facts/Conflict Dedup] [Quadrant IV: Vector/Graph Engine Foundation]
  Mem0 (entity-relationship graph)          sqlite-vec (pure C, single file)
  cognee (knowledge graph + multi-strategy  KuzuDB (embedded graph database)
          retrieval)                         Role: Embedded deployment-free storage
  Role: Long-term fact maintenance
```

### MCP and Memory Layer

agentmemory and cognee adapt commercial IDEs through MCP, a "political compromise." Memory layer through MCP is acceptable (low-frequency operations), tool execution layer through MCP is not (high-frequency operations). skillforge uses framework adapter layer (SnapshotSessionDB) instead of MCP, in-process calls with no network overhead.

### Gap in Synthesis Loop

Ideal memory loop: Capture → Compression → Conflict dedup → Persistence → Skill consolidation.

**Key Break: Verification Layer Missing.** Every step can introduce errors — compression loses context, reflection misjudges facts, conflict detection misses. Without verification layer, error accumulation isn't compound interest, it's compound loss. agentmemory solves triggering with hooks but binds MCP; cognee partially solves evolution with improve() but LLM cost heavy; skillforge ensures quality with human review but sacrifices automation. The trilemma hasn't been truly solved by open-source solutions.

---

## VIII. Summary

### Current State

skillforge has implemented the two-layer architecture's Surface (threshold trigger + checkpoint injection + SnapshotSessionDB adaptation) and Engine's basic form (flat KV extraction + HNSW/BM25/RRF retrieval + SurrealDB storage). Three core interfaces: store / search / forget.

### Evolution Direction

| Phase | What Changes | What Doesn't Change |
|:--|:--|:--|
| **Graph Evolution** | Engine: flat KV → graph triples, retrieval adds graph traversal | Surface (trigger, injection, adaptation) unchanged |
| **Framework Migration** | Surface: rewrite SnapshotSessionDB (dozens of lines) | Engine (MemoryManager, MemoryTools) unchanged |
| **Weight System** | Engine: add read_count/write_count tracking and ranking | Surface unchanged |

### Design Principles

1. **Surface and Engine separation**: Switch frameworks only change Surface, switch storage/retrieval only change Engine
2. **LLM call minimization**: Unaware within 100 messages, compression integrated into Agent normal turn, no separate LLM calls
3. **Framework agnostic**: Memory logic doesn't depend on any specific Agent framework
4. **Progressive evolution**: Use flat KV when sufficient, upgrade to graph structure when needed
