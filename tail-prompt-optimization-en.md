# Cache Tree and Tail Prompt Optimization

*A temporary instruction injected at the end of a prompt, leveraging the KV cache bypass branch mechanism to perform specific tasks.*

## Definition

**Cache Tree**: Multiple turns share a common KV cache prefix, forming a tree structure:

```
                    [system prompt + conversation history]    ← trunk (shared cache)
                   /                                        \
    [branch A: tail prompt A + question]    [branch B: tail prompt B + question]   ← branches (new computation)
```

Tail prompts leverage the tree's branching特性: share the trunk (history cache), only compute new branches (tail prompts).

```
KV cache (unchanged): [conversation history]                 ← trunk
bypass branch (new computation): [tail prompt + user question] ← branch
  → LLM completes both in a single turn: answer user + execute tail prompt task
  → After turn ends, tail prompt is discarded, only results remain
```

Analogy to TCO (Tail Call Optimization): tail calls reuse the current stack frame; tail prompts reuse the current KV cache. Both are "tail" operations that reuse existing state.

## Core Properties

- **Cache utilization**: History portion uses KV cache; only the tail prompt is newly computed
- **Bypass branch**: Does not modify the main branch (conversation history), only appends temporary instructions at the end
- **One-shot**: Removed from prompt after turn ends, not written to session
- **Control**: Injector decides when and what to inject; Agent only executes

## Application Scenarios

Core criteria: **needs to process existing history + wants to leverage cache + can complete within a single turn**.

| Scenario | Trigger | Tail Prompt Content | Result |
|:--|:--|:--|:--|
| **Compression** | Messages reach threshold (e.g., 100) | "Analyze above conversation, extract memory, mark checkpoint" | Old history replaced by checkpoint, discarded directly |
| **Cache renewal** | Session idle + cache near expiry | "Refresh checkpoint" | Checkpoint updated, cache re-hit |
| **Topic boundary detection** | Every N turns | "Detect if conversation switched topic" | If switched, trigger checkpoint |
| **Preference learning** | Periodic (e.g., every 20 turns) | "Extract user preferences and update" | `memories` table updated |
| **Summarization** | After discussion ends | "Summarize key points of above discussion" | Generate summary, inject into subsequent context |
| **Extraction** | Anytime | "Extract user preferences from conversation" | Structured knowledge written to storage |
| **Compliance audit** | Before critical operations | "Check if above conversation complies with standards" | Audit log, no history modification |
| **Meeting minutes** | After meeting ends | "Generate structured meeting minutes" | Markdown output, written to document |
| **Code review** | After code discussion | "Review the code approach discussed above" | Review comments, written to issue |
| **Auto-tagging** | Session ends | "Generate tags for this conversation" | Tags written to session metadata |

Compression is a special case: after the tail prompt triggers tool calls, the preceding history is discarded directly (replaced by checkpoint). Other scenarios preserve history.

**Compression = in-place replacement, not bypass branch**: Normal tail prompts are bypass branches (append temporary instructions, history unchanged). Compression differs — cannot modify external Agent loop, must replace in-place within the prompt: after LLM executes tool calls, old history is replaced by checkpoint and discarded directly. Prerequisite: compression doesn't need branches (not even the bypass branch), only needs to complete replacement within a single turn.

## Compression Scenario Full Example

Assume conversation has accumulated 150 messages, user asks "Why is Fluxora's component set closed?"

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

## Comparison with Traditional Approaches

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
