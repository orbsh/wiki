# Tail Prompt

*A temporary instruction appended to the end of a prompt, leveraging KV cache side-branching to complete specific tasks.*

## Definition

```
KV cache (unchanged): [historical conversation]
Side branch (new computation): [tail prompt + user question]
  → LLM completes both in one turn: answer user + execute tail prompt task
  → After turn ends, tail prompt is discarded, only results remain
```

Analogy to TCO (Tail Call Optimization): tail calls reuse the current stack frame; tail prompts reuse the current KV cache. Both are "tail" operations, both exist to reuse existing state.

## Key Properties

- **Cache utilization**: Historical tokens hit KV cache; only the tail prompt is newly computed
- **Side branch**: Does not modify the main branch (conversation history); only appends temporary instructions at the end
- **Ephemeral**: Removed from prompt after turn ends; never written to session
- **Control**: The injector decides when and what to inject; the Agent only executes

## Use Cases

| Scenario | Tail Prompt Content | Result |
|:--|:--|:--|
| **Compression** | "Analyze the conversation above, extract memories, mark checkpoint" | Old history replaced by checkpoint, directly discarded |
| **Summarization** | "Summarize the key points of the discussion above" | Generate summary, inject into subsequent context |
| **Extraction** | "Extract user preferences from the conversation" | Structured knowledge written to storage |

Compression is a special case: after the tail prompt triggers tool calls, previous history is directly discarded (replaced by checkpoint). Other scenarios preserve history.

## Comparison with Traditional Approaches

| Dimension | Traditional Compression | Tail Prompt |
|:--|:--|:--|
| LLM call | Separate API call | Reuses Agent's normal turn |
| KV cache | Computes from scratch (0% hit rate) | Historical tokens hit cache (~99% hit rate) |
| Attention | Fully focused on compression task | Split (answer user + execute task) |
| Control | Compressor controls | Injector controls (via prompt + tools) |

## References

- [agent-memory.md](agent-memory.md) — Tail prompt application in Prefix Checkpoint
- [graph-memory.md](graph-memory.md) — Extraction prompt in graph memory
