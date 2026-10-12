# LLM Cache Destruction Patterns

> **Languages:** [English](llm-caching-destruction-patterns-en.md) (primary) · [中文](llm-caching-destruction-patterns.md)
> The conditions for Prompt Caching to take effect are extremely strict: the input prefix must be stable. Any behavior that breaks prefix stability — switching models, switching tools, switching users, switching projects — instantly invalidates the cache.

---

## 1. Core Principles

Prompt Caching is essentially **KV Cache reuse**: the server stores previously computed Key-Value activation states in VRAM, and skips the computation the next time it encounters the same prefix.

Conditions for it to take effect:
- **The prefix must be 100% identical** — even a single differing character makes the hash mismatch
- **The cache has a TTL** — usually 5-30 minutes, cleared on expiry
- **The cache has a capacity limit** — LRU eviction, new requests push out old caches

These three constraints determine: **any operation that causes prefix changes, VRAM contention, or long time gaps is a cache killer.**

---

## 2. IDE Scenario: Constantly Changing Prefix

IDE plugins (Copilot, Cursor, Cline) are the "sworn enemy" of caching — not because the engineering is poor, but because they are mutually exclusive at the architectural level.

**The IDE's work loop**: the user types → the context changes → a request is triggered. Every keystroke changes the prefix.

In the background, the IDE pieces together:
- Several hundred lines of code before and after the cursor
- All open tabs
- Terminal logs
- Relevant snippets retrieved by RAG

With every character the user types the prefix changes, and the cache cannot hold for even a second.

More fundamentally: **the IDE's core KPI is accuracy, not cost savings.** "Better to send too much than to miss something" — RAG and AST analysis stuff a different code snippet into the Prompt each time, running entirely counter to the "consistent prefix" cache mechanism.

→ The IDE's pragmatic compromise: high-frequency Tab completion is handed to a local 3B-7B small model; only complex tasks go to the cloud.

---

## 3. Model Switching: Physical Isolation

Switching between different models (e.g. Hermes ↔ Codex) makes the cache **100% invalid**. Three levels of reasons:

### 3.1 Physical Model Isolation

KV Cache is activation state stored in GPU VRAM, tightly bound to a specific model instance. Codex absolutely cannot read Hermes's cache — even if they are based on the same base model (e.g. Llama).

### 3.2 LRU Eviction

During the switch, the server clears your previous cache to free VRAM for handling other requests. By the time you switch back, the cache is most likely already gone.

### 3.3 Prompt Template Conflict

Different models have completely different special tokens and input formats (`<|im_start|>user` vs `### Instruction:`). Even if the user text is the same, the hidden format markers differ, the binary hash mismatches, and the cache is scrapped.

---

## 4. Mixing Multiple Tools: Prefix Position Determines the Cache's Fate

Tools do not necessarily destroy the cache by themselves. The key variable is **where the tool result sits in the prompt**.

### 4.1 Results Appended at the End → Cache Preserved

```
[system] [history] [tool_result_A]
                   [tool_result_B]    ← B appended after A
```

The result of each tool call is appended to the end of the prompt, so the prefix `[system] [history] [tool_result_A]` is unchanged and subsequent requests can still hit the cache. This is the default behavior of frameworks such as Agno and LangChain — tool results are appended to the end of the message list.

### 4.2 Results Inserted in the Middle → Cache Destroyed

```
[system] [history]          ← original prefix
         [tool_result_A]          ← insertion position 1
         [tool_result_B]          ← insertion position 2 (a different position from A)
```

If a framework inserts tool results at different positions in the prompt (for example, RAG snippets inserted after the system prompt and before the user message), the insertion position differs each time, the prefix breaks, and the entire cache is invalidated.

### 4.3 System Prompt Switching → Cache Destroyed

Different tools come with their own dedicated System Prompt (an image tool injects image-generation instructions, a data-analysis tool injects Python conventions). Switching tools = switching the system prompt = the very first token of the prefix differs = the cache is invalidated from the start.

### 4.4 Large Results → LRU Eviction

Tool A generates 20KB of text, and tool B immediately fetches a 50KB web page — large inputs occupy VRAM, and to free space for processing the new data the server may evict the previous cache. This is LRU eviction, not a prefix problem.

---

## 5. Routing Aggregation: The Cache Breakdown Trap

When using third-party routing services (OpenRouter, etc.), the cache hit rate may be far lower than when calling the official API directly.

### 5.1 Cost Inversion

Cache hit = 10-20% of the standard input price. Cache miss = 100% of the standard price.

To improve utilization of the underlying accounts, routing services distribute requests randomly across different accounts — each account's context is overwritten by other users, and the cache hit rate plunges from 95% to 20%. The result: the base unit price is discounted by 20%, but the total bill soars 2-3x.

### 5.2 How Routing Services Save Themselves

- **Sticky session routing** (Sticky Session): the same conversation is pinned to the same underlying account
- **Smart semantic pooling** (Semantic Pool): requests with the same System Prompt are directed to a dedicated account pool

---

## 6. Multiple Users Sharing an Account: Contexts Pollute Each Other

When multiple users share one API account, the cache is destroyed by personalized information:

- Different users have different System Prompts → the prefix hashes disagree → the cache is invalidated
- User A's conversation context is overwritten by user B's requests → the context cache (Context Cache) is evicted
- High-frequency calls trigger RPM/TPM rate limits → requests are rejected

---

## 7. Cache Branching Patterns: The Opposite of Destruction

The previous six sections discussed what **destroys** the cache. But the cache is not merely consumed passively — a sensible branching design can **actively exploit** it.

### 7.1 Thread: A User-Driven Persistent Branch

```
[main_1] [main_2] [main_3]
                   ├── [thread_A_1] [thread_A_2]
                   └── [thread_B_1] [thread_B_2]
```

The KV state of the parent message is the shared prefix. Each thread continues independently from the fork point, sharing the parent node's cache. Switching threads reloads the cache of the corresponding branch — the cost is switching latency, the benefit is zero recomputation within the branch.

The difference from destruction patterns: a thread is an **explicit branch** (the user chooses the reply position), whereas a destruction pattern is **implicit pollution** (tools/models/users unintentionally changing the prefix).

### 7.2 Tail Prompt: A System-Driven Temporary Branch

```
Context cache (unchanged): [history]
Bypass branch (newly computed): [tail prompt + user question]
  → after the turn ends, the tail prompt is discarded and only the result is kept
```

The difference from a thread: the tail prompt is a one-off branch (single turn), whereas a thread is a persistent branch (across turns). Both exploit prefix caching; the difference lies in their lifecycle.

### 7.3 Compression Scenario: A Parallel Bypass Branch

Prefix Checkpoint is a generalization of the tail prompt — not an in-place replacement, but a **parallel bypass**. When the watermark is reached, the main branch answers the user's question as usual, while the compression instruction forks off the same immutable prefix as a parallel bypass: the prefix KV is shared and computed only once, and the bypass pays only its own increment; the summary lands before the next user request (in the typing gap), turning the prefix switch from a hop in the middle of the conversation into a background transaction.

The original scheme mixed compression into the normal turn ("answer first, then call tools") — that commandeers the conversational rhythm, and the two tasks of answering and compressing split attention. The parallel bypass splits the two tasks onto two branches: the main line stays clean, the bypass is stored externally. The serial latency of the old scheme ("compression must wait for the model to free up") is zero under parallelism: the inference service naturally supports prefix reuse for concurrent requests with the same prefix.

→ For the complete mechanism, see the "Compression is a bypass parallel to the main line" section of [Stateless Agent Architecture](stateless-agent-architecture-en.md); for the general mechanism of the tail prompt, see [Tail Prompt Optimization](tail-prompt-optimization-en.md)

### 7.4 Criterion for Branch Legitimacy: Past-Facing vs Future-Facing

Multiple kinds of branches can fork from the same prefix, and legitimacy is determined by **how much the output depends on the future**:

```
Branches that can fork from the immutable prefix [H]:

✅ Past-facing read branch (input already determined, output always valid)
   ├── Compression / summarization (serves context management)
   ├── Long-term memory extraction ("user prefers X", "project uses option B")
   ├── User profile update
   └── Fact/state archiving (decision log, list of rejected options)
   Traits: extracting determined information from a determined prefix. If the
   user changes direction next, changes topic, or even closes the session, the
   extraction result is still valid — it describes "the conversation that has
   already happened", not "the behavior that is about to happen".

❌ Future-facing speculative branch (betting on the user's next move)
   ├── Pre-generating answers
   └── Pre-expanding candidate options
   Traits: the value of the output must be independent of future input outside
   the prefix, otherwise it is speculation. In a continuous Q&A scenario the
   future input is precisely the highest-entropy part — guess wrong and the
   whole result is wasted, and the cache increment was paid for nothing. This
   is a principled exclusion, not a matter of engineering parameters.
```

Past-facing branches also share a common structural trait: **the output does not enter the conversation but lands in external state** — summaries go to the session manager, profiles to the user archive, memories to the memory store. All are bypass deposits that naturally do not pollute the prefix. This is entirely different from the positioning of pre-generation, where "the output waits to be consumed by the main line": the former is a read-only prefix + persistence into a store, the latter is betting on the future + waiting for consumption.

This set of relationships is isomorphic to materialized views / read-write splitting in databases: the prefix is the single source of truth, all read branches are read-only derived views of it, and write switches happen only in the gaps between turns; derived views can be arbitrarily many, arbitrarily parallel, and recomputed at any time (idempotent), because they face the past — the past does not change, so the views are always valid. The pre-generation branch, by contrast, corresponds to cache warming: it is worth doing only when the access pattern is predictable; the access pattern of open-ended conversation is unpredictable, so it is not done — and the boundary is thereby made clear.

---

## 8. The Spectrum of Valid vs Invalid Cache

| Scenario | Prefix Characteristics | Cache Effectiveness |
|------|---------|-----------|
| Single user / single model / pure-text conversation | Static prefix + changing suffix | **High** |
| Single user / multi-turn conversation (no tools) | Fairly stable | Medium |
| Thread branch | Explicit branching, parent node shared | **High** (within the branch) |
| Tail prompt | Temporary branch, single turn | **High** (prefix reuse) |
| IDE real-time completion | Constantly changing | **Almost zero** |
| Model switching | Physical isolation | **Zero** |
| Mixing multiple tools | Dynamic data insertion | **Very low** |
| Routing aggregation / multi-user sharing | Contexts overwrite each other | **Very low** |

**Pattern: the more participants involved (models, tools, users, projects), the less controllable the prefix and the less effective the cache. Actively designing branches (thread, tail prompt) beats passive pollution.**

---

## 9. Essential Conclusion

Cache effective = prefix stable = context predictable.

Real-world usage scenarios — IDE, multiple models, multiple tools, multiple users — are precisely the unpredictable ones. This is not a problem LLM vendors can optimize away; it is a structural constraint.

Pragmatic responses:
- **Do not count on caching to save money** — treat it as an unexpected bonus, not the basis of a cost model
- **Use a local small model for high-frequency short tasks** — bypass the cache problem and cut out network overhead outright
- **Keep long conversations on a single model with few tools** — if you really need caching, reduce the variables
- **Choose sticky sessions for routing services** — if you use an aggregation platform, confirm that it supports Session Affinity

→ For related insights, see [LLM Fundamentals](llm-fundamentals-en.md)

---

## Cross-References

| Document | Related Argument |
|------|---------|
| [LLM Fundamentals](llm-fundamentals-en.md) | Probabilistic predictor / horizon × speed / high-dimensional keyword matching / KV cache tree structure |
| [Tail Prompt Optimization](tail-prompt-optimization-en.md) | Cache branching mechanism, in-place replacement in compression scenarios |
| [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md) | The triple knowledge dilemma of AI programming |
| [Language Selection for LLM-Generated Code](llm-language-selection-en.md) | Language criteria on the generator side: verification cost, escape hatch, and governance |