# Cache Tree and Tail Prompt Optimization

## 1. Cache Tree (Cache Tree)

Multiple turns share a common KV cache prefix, forming a tree structure:

```
                    [system prompt + conversation history]    ← trunk (shared cache)
                   /                                        \
    [branch A: question A]    [branch B: question B]    [branch C: question C]   ← branches (new computation)
```

Core properties:
- **Shared trunk**: All branches reuse the parent's KV cache, no redundant computation
- **Independent branches**: Each branch's new tokens are computed independently
- **Cache reuse**: When switching branches, the common prefix cache remains available (within TTL)

Typical scenario: IM platform thread mechanisms. Each thread is a branch sharing the main thread's prefix cache. Usually, base information is pre-filled into cache first (e.g., loading project code/docs in coding scenarios), then different branches execute different tasks — each branch inherits the cached project context and only needs to compute its own task tokens.

```
Main thread (trunk)
  ├── thread_A: fix bug      → inherits trunk cache + bug description
  ├── thread_B: add feature  → inherits trunk cache + requirement description
  └── thread_C: write tests  → inherits trunk cache + test scope
```

Each thread's cost ≈ task description tokens (project context uses cache).

---

## 2. Tail Prompt Optimization

*A temporary instruction injected at the end of a prompt, leveraging the KV cache bypass branch mechanism to perform specific tasks.*

Tail prompts are a variant of cache trees — **Cache Vine**. One main trunk grows continuously, with leaves sprouting periodically; after leaves fall, the trunk continues growing.

```
[system prompt + conversation history] ─── trunk (accumulates continuously)
       │
       ├── [tail prompt A + question]  ← leaf (one turn, falls off)
       │
    trunk continues growing...
       │
       ├── [tail prompt B + question]  ← leaf (one turn, falls off)
       │
    trunk continues growing...
       │
       └── [tail prompt C + question]  ← leaf (one turn, falls off)
```

Differences from cache trees:

| Dimension | Cache Tree | Tail Prompt (Vine) |
|:--|:--|:--|
| Trunk | Shared trunk | Same, but trunk accumulates continuously |
| Branch | Persistent thread | Temporary leaf (one turn) |
| Lifecycle | Cross-turn, user-driven | Single turn, system-injected |
| Controller | User selects branch | System decides when to sprout leaves |
| Typical use | Multi-topic parallelism | Background tasks (compression, extraction, audit) |

### KV Cache Utilization

```
KV cache (unchanged): [conversation history]                 ← trunk
bypass branch (new computation): [tail prompt + user question] ← leaf
  → LLM completes both in a single turn: answer user + execute tail prompt task
  → After turn ends, leaf falls off (tail prompt discarded), only results remain
```

Analogy to TCO (Tail Call Optimization): tail calls reuse the current stack frame; tail prompts reuse the current KV cache. Both are "tail" operations that reuse existing state.

### Core Properties

- **Cache utilization**: History portion uses KV cache; only the tail prompt is newly computed
- **Bypass branch**: Does not modify the main trunk (conversation history), only appends temporary instructions at the end
- **One-shot**: Removed from prompt after turn ends, not written to session
- **Control**: Injector decides when and what to inject; Agent only executes

### Compression Scenario: In-Place Replacement When Leaf Falls

Tail prompts use the vine pattern — one main trunk sprouts leaves, then continues growing. The compression scenario's special case — when the leaf falls, the preceding trunk is also replaced (checkpoint replaces old history).

```
KV cache (unchanged):
  [ckpt_0] + [msg_101..msg_150]

Newly appended messages (only uncached portion):
  tail prompt: "First answer the user's question, then analyze above conversation,
                call memory_store to extract preferences/facts, mark checkpoint."
  user message: "Why is Fluxora's component set closed?"

Agent completes three things in one turn (note order):
  1. "Fluxora's component set is closed because..."       ← answer user question first
  2. tool_call: memory_store(...)                          ← then memory operations
  3. tool_call: memory_checkpoint(...)                     ← finally mark checkpoint
```

After turn ends, session stores clean history:

```
Stored in session:
  [checkpoint_1] + ["Why is Fluxora's component set closed?"] + ["Fluxora's component set is closed because..."]

NOT:
  [checkpoint_1] + [tail prompt + "Why is Fluxora's component set closed?"] + ["Fluxora's component set is closed because..."]
```

Tail prompt discarded, no residue.

### Comparison with Traditional Approaches

| Dimension | Traditional Compression | Tail Prompt |
|:--|:--|:--|
| LLM Call | Independent API call | Reuses Agent's normal turn |
| KV cache | Computes from scratch (0% hit rate) | History portion uses cache (~99% hit rate) |
| Attention | Fully concentrated on compression task | Split (answer user + execute task) |
| Control | Compression module controls | Injector controls (via prompt + tools) |

## References

- [agent-memory.md](agent-memory.md) — Tail prompt application in Prefix Checkpoint
- [graph-memory.md](graph-memory.md) — Extraction prompts in graph-structured memory
- [LLM Caching Destruction Patterns](llm-caching-destruction-patterns.md) — Cache branch patterns (thread) vs destruction patterns
