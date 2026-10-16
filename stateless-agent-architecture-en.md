# Stateless Agent Architecture

> **Languages:** [English](stateless-agent-architecture-en.md) (primary) · [中文](stateless-agent-architecture.md)

*An architecture distinct from conventional Agent frameworks: stateless, cloud-native, serverless. For memory design (checkpoint, retrieval, extraction) see [Memory Architecture](agent-memory-en.md); for the memory system implementation see [Krystallizer](krystallizer-en.md); for the Aura engine itself see [Aura Architecture](aura-architecture-en.md).*

## Core Proposition: The turn Is the Only Unit of Execution

A conventional Agent is a resident process: it holds the session history in memory, and when the process dies the session is lost; deploying multiple instances requires solving stickiness and state synchronization. This architecture reduces the loop to a pure function:

```
f(session, user_input) -> session'
```

One turn = the interface receives the request data → fetches the session from Krystallizer in full mode → executes (LLM call + tool-call handling) → appends the new messages back to Krystallizer. The loop holds no state between calls and never rewrites history — persistence and recovery are all pushed down into the memory system.

The direct benefit of this corollary is **decoupling of deployment form from execution location**: it makes no difference which instance a turn lands on; a local CLI is simply a loop executing turns; in a distributed environment (Aura / function compute) the next turn can execute anywhere. Statelessness is not an operations feature but a structural property — scale-to-zero, multi-instance operation, and cross-machine migration are all obtained for free.

Theoretical basis: Aura's "API is stateful, operations are stateless" — `ctx.state` is stateful programming, while state persistence/recovery/scale-to-zero sink down to the lower layers. Session as data is the agent landing of the same position.

## Components

| Component | Metaphor domain | Responsibility | Form |
|:--|:--|:--|:--|
| **Krystallizer** | Crystallization | Sole holder of session truth: append-only session + its own compression-point table + view trimming; skill graph and emergence | Memory plane (mem-core embedded or standalone service) |
| **Prism** | Prism | Entry: authentication, request parsing, casting turns into the domain. WS gateway + CLI (wrapping WS) | Built on Aura, pure entry, no queue, stateless |
| **Gravity** | Gravity | turn executor: fetch session → run turn → store delta. The turn is driven by session truth; single-pass execution (one turn per pass), not a resident loop | A stall type in Aura (partition key = session_id); ships with a CLI-driven loop, and local mode is just a loop executing turns |
| **Effector** | Probe | Operating execution environment: containerized execution environment + operation executor. Receives dispatched operations and parameters (skills are invisible to it — skill resolution and selection happen on the Gravity/LLM side); execution results flow back as tool results | An Aura base component, also deployable standalone as a remote service |

```
          request
             │
      ┌──────▼──────┐  turn dispatch  ┌──────────────────────────┐
      │    Prism    ├────────────────►│ Gravity (stall)          │
      └─────────────┘                 │  LLM selects operations, │
                                      │  generates parameters    │
                                      └───┬──────────┬───────────┘
                                          │          │ task frame (op + params)
                     fetch session        │          ▼
                     append delta     ┌───▼──────────┐  execute op  ┌──────────┐
                     (full mode)      │ Krystallizer │              │ Effector │
                                      │ (memory +    │              │ local/   │
                                      │  skill graph)│              │ remote   │
                                      └──────────────┘◄─────────────└──────────┘
                                                     tool result
```
(Skill resolution happens on the Gravity side: the session fetch from Krystallizer → Gravity carries the resolution result with it; Effector sees only operations and parameters.)

### Prism: Entrypoint Contraction

Prism is built on Aura and deliberately thin: authentication + parsing + dispatch. Queues, retries, timeouts, and cross-machine scheduling are the proper job of Aura's domain model — building another queue-carrying gateway beside Aura would violate Aura's MQ decomposition ruling (do not introduce a separate queue component). Prism is only responsible for translating a user-side request into a single turn dispatch within the domain.

**The interface is WS; the CLI wraps WS.** Only one protocol is exposed externally: the WS gateway (a long-lived connection carrying turn dispatch and streaming push-back), and the CLI is a thin wrapper around a WS client — local commands and remote calls travel the same protocol path, with no second RPC surface maintained. The CLI is shaped for automation (one command = one call, batch input, machine-readable output, meaningful exit codes) rather than an interactive REPL, and its priority sits below the web front end — the web is the human interface, the CLI is the automation interface. The statefulness of WS is absorbed by Prism's position: on the other side of the connection is the Aura domain (resident), while the turn's execution body (Gravity) remains stateless and single-pass — the connection is pinned to Prism, not to Gravity, so scale-to-zero semantics are not harmed.

### Gravity: The Loop Is a Function

Gravity's entire responsibility: fetch the session from Krystallizer in full mode → the loop of LLM calls and tool calls → append the newly added messages back. In essence each Gravity is **single-pass**: one turn per pass, and it stops when done — there is no resident loop entity; the "loop" is merely the successive dispatch of turns. This is why it is named Gravity rather than Orbit: an orbit is a property of a resident object, whereas gravity is a force applied on every pass — a turn is pulled by session truth the moment it lands, released when execution finishes, and the next pass lands elsewhere.

It is implemented in pure Rust (booth form = an induction native child process speaking the stdio pipe frame protocol; Aura's established topology: interpreted languages ride the embedded carriers, compiled languages ride induction). Gravity itself is unaware of the language of tool scripts — tool call in, tool result out; language diversity is solved on the Effector side.

**LLM calls belong to Gravity**: LLM calls triggered by tail prompts such as compression are implemented by Gravity — the context cache is bound to the model (and account), and only the side that initiates inference knows the model information. Krystallizer provides only primitives (branch injection, summary collection, checkpoint writing) and holds no model identity; the sole exception is its internal vector embedding (the embedding space is bound to data correctness and is self-held by the memory system). See [Krystallizer](krystallizer-en.md).

**Compression is a side channel parallel to the mainline**:

The compression decision lives in Krystallizer — it holds the entire context, so its information is the most complete. The decision input is multidimensional, not a single threshold:

- **Volume**: the message count/token volume reaches a threshold (the original implementation used a fixed count such as 100 messages; the trimming boundary should align to **structural units** — a closed tool-call pair or a single standalone message, rather than a cut at a fixed count; if the cut falls in the middle of a tool pair, extend it until it aligns)
- **Time interval**: the last message is a long time ago (e.g. on the order of the cache TTL) and the leading segment is most likely cold — solidify the invariant prefix into a checkpoint before the cache expires
- **Weight**: old messages have a low reference density (the `tool_invoke_count` side-channel written by the graph)

When the conditions are met, compression does not occupy the user's critical path: on the arrival of U_n, the same watermark **forks two read branches in parallel** off the immutable prefix H —

```
Main branch (user critical path): [H] + [U_n]                        → normal answer A_n
Side channel (issued in parallel): [H] + [compression instruction]   → summary S
      The two requests share prefix H: for concurrent requests with the same prefix the
      inference service natively supports prefix reuse (a fork of one prompt), so H's KV
      is computed only once and the side channel pays only its own delta.

Before U_{n+1} arrives, the side-channel result lands: the session switches to [S] + [U_n, A_n] + [U_{n+1}]
```

- **Invisible to the user**: the answer latency equals that of an ordinary round, with no pause of "the system is organizing memory"; the compression generation time is hidden in the user's typing interval (seconds — more than enough).
- **Switching cost minimized**: the switch happens before the U_{n+1} request, and the rebuild of the new prefix S + recent segment is exactly the prefix this round's request was going to pay anyway — the rebuild volume is far smaller than H; afterwards S becomes the new stable prefix and the system returns to a full cache hit. Parallelization turns the "switch" from a hop inside the conversation into one background operation inside a typing interval: the theoretical lower bound is unchanged, but the way it is reached changes from a "perceptible cost" into a "background transaction hidden in the gap between rounds".
- **The side channel does not enter the mainline**: S is externally stored state (written to the checkpoint table); it does not pollute the prefix and does not interrupt the conversational rhythm; mainline requests stay clean, with stable prefix bytes.
- **For a time-interval trigger with no turn to attach to** (the cache-TTL cooling scenario): the right to initiate still lies on the client side — the CLI idle tick (local deployment), a scheduled dispatch on the Prism connection (server-side deployment), or an Aura `@cron`/`on_debounce` stall sending it on behalf; Krystallizer passively waits for the fetch and can never wake Gravity — the right to initiate lies only on the side that has an LLM. The difference is that the compression initiated here is just a standalone side-channel request, not on any user's critical path: whether to trigger is purely a cost question and no longer touches experience.
- The trimming product is **not limited to a message count**: how large the compression interval is and how much tail to keep are computed from the decision dimensions, not fixed at one or two messages. Summaries and external writes go through Krystallizer's `summarize`/checkpoint primitives (internally composing branch injection, summary collection, truncation of old messages, and writing a new checkpoint) — the side channel is one use case of branch, and Krystallizer adds no new primitive.

**pending input buffering**: user input may consist of multiple messages (several sent in a row merge into a single turn). When the session is fetched, pending input lands in Krystallizer's buffer and is consumed by the main branch. The compression side channel reads only the finalized prefix (H does not contain pending) and does not absorb user input — on switching, `[S] + [recent segment]` preserves all finalized messages, and the buffer is cleaned up normally along with the main branch. Under the side-channel model this is both simpler and safer: the summary never touches user input that has not yet been consumed.

**Implementation form on the Krystallizer side**: the compression decision is hidden inside fetch (when assembling the view it decides whether the watermark triggers the side channel), and the side channel itself goes through the `summarize(session_id)` primitive (internally composing branch injection, summary collection, truncation of old messages, and writing a new checkpoint). The decision dimensions (volume/time/weight) are caller-controllable configuration, not private behavior hidden in the storage layer.

### Concurrency and Sharing: Two Kinds of Session Ownership

Concurrent input to the same session (typing on a phone and a computer at the same time) is not a race to be scheduled but a choice of product semantics:

- **Not shared**: two sessions, each evolving independently. The default form, with no new mechanism whatsoever.
- **Shared**: a later message **cancels the execution of the previous round** and re-executes the complete turn — both consecutive user messages must be in the context (when the API does not support multiple user messages, Krystallizer merges them; when the UI supports forking, a branch naturally grows here: the earlier message runs to completion on the original branch, and the later one opens a new branch). Cancellation holds naturally thanks to Gravity's single-pass model: the append happens only after the turn finishes, so cancellation is just discarding the un-appended execution.

A shared session therefore has only one semantic — "the latest intent wins" — with no partially completed intermediate state. When the execution body is stateless, "cancel and redo" is the entire answer for concurrent sessions — no coordination mechanisms such as version numbers or conflict arbitration are needed.

## Surface Design: Injection and Triggering


### Prefix Checkpoint

When you chat with AI, every time you say something, the AI has to re-read all the previous conversation from the start. The longer the conversation, the slower and more expensive the AI's reading becomes.

The conventional approach is to "remember the most recent N messages" — but every time a new message arrives, the previous N change (the oldest is squeezed out, the newest is added in), which is equivalent to the AI re-reading everything from scratch every time.

Prefix Checkpoint works differently:

1. Once the conversation accumulates to a certain length (say 100 messages), the AI summarizes a passage itself: "The user is discussing project architecture and has decided to use SurrealDB..."
2. This summary is **fixed and never changes**. All subsequent conversation is appended after it.
3. Because the beginning does not move, the AI can "skip" the part it has already understood and process only new messages — like turning to a bookmarked page in a book, without re-reading from page one.

The summary is the "prefix" (Prefix), fixed and unchanging; "Checkpoint" is the save point. After each new save, the old save is cleared and accumulation starts over.

**Why conventional compression cannot exploit the cache**: the conventional approach sends the 100 messages to a separate model (or issues a separate API call) to generate the summary. This call has no cache relationship with the preceding conversation — even if you use the same model, what it sees is an entirely new piece of text, the context cache is computed from scratch, and the cache hit rate is 0%.

Prefix Checkpoint is different: the 100 messages are already in the cache (every previous round of conversation used them). The tail prompt merely appends a small piece of new text at the very end. When the model processes it, the first 99% of the tokens go straight through the cache, and only the final prompt + user question are newly computed.

```
Conventional compression:
  [100 messages] → new API call → everything recomputed → output summary
  Cache hit rate: 0%

Prefix Checkpoint:
  [100 messages (in cache)] + [tail prompt + user question]
  Cache hit rate: ~99%
```

The result: fast and good, with low consumption. Fast because of the cache, good because it uses the same strong model (not a small model), and low consumption because most tokens do not need recomputation. And there is no need to modify the Agent loop — compression is done through a normal tool call.

**"Auxiliary model" ≈ the conventional pattern**: an Agent system that can be configured with a separate auxiliary model is essentially conventional compression — it requires a separate LLM call and cannot exploit the cache. The so-called "fast model" is only relative: send 100 messages and the small model also has to compute from scratch, so the latency may be only 30-50% lower, while the quality is far worse. Prefix Checkpoint does not even need an auxiliary model — compression is just part of the Agent's normal answer.

### Memory Control

Prefix Checkpoint is not merely a cache optimization mechanism; more importantly, it is a **memory control framework**.

Before every reply, the AI reads the complete conversation. If we want the AI to do something (say, "write down the preference we just discussed"), we only need to append a sentence at the end of the conversation.

When the conversation accumulates past the watermark, the memory system forks a compression side channel off the invariant prefix (see the "Compression is a side channel parallel to the mainline" section):

```
Side-channel request: [all previous conversation] + [compression instruction]  ← prefix goes through the cache, pay only the delta
Main branch:          [all previous conversation] + [user's question]         ← issued in parallel, neither waits for the other
```

The side-channel output summary S is written to the checkpoint table and does not occupy the user's critical path — what he is waiting for is the main branch's answer, and compression is completed in the branch beside it. The control logic is unchanged: whatever instructions the memory system injects and whatever tools it provides determine what the branch does; the only difference is that compression no longer mixes into the answering turn but is an independent fork.

**Control mechanism**: the Agent is passive — it sees only the prompt and the available tools, and calls tools according to the instructions in the prompt. The memory system achieves complete control by controlling two things:
1. **What to inject** — the tail prompt (determines what the Agent does)
2. **What tools to provide** — the list of tools the Agent can call (determines what the Agent can do)

**This allows the memory system to**:
- Control compression timing (inject the compression instruction when the threshold is reached)
- Control what is extracted (the prompt specifies the preferences/facts/decisions to extract)
- Control the storage method (the tool decides which table to write to and in what format)
- Modify nothing in the Agent loop — all control is achieved through prompt + tools

**Analogy**: the Agent is an "executor with no opinions" — it has capability (LLM reasoning + tool calling) but no intent. The memory system injects intent through the prompt, and the Agent is responsible for executing it. This is similar to an operating system's "system call": the kernel provides capability (syscall) and user-space programs decide when to call it.

### Trigger Timing

Three kinds of memory, three trigger mechanisms:

| Memory type | Tool | Who triggers | Mechanism | Pros | Cons |
|:--|:--|:--|:--|:--|:--|
| **Long-term memory** | `memory_store` | **LLM** (active judgment) | The LLM calls it during the conversation when it judges that "this is worth remembering" | Precise, stores only what is valuable | Depends on the LLM's judgment; may miss things |
| **Short-term memory** | `memory_checkpoint` | **Memory system** (threshold trigger) | When messages accumulate to N, fork a compression side channel off the prefix (not on the critical path), and the Agent framework performs the DB operations | Low-frequency calls, controllable overhead | What is within N stays uncompressed (but the raw messages remain in the session) |
| **Memory retrieval** | `memory_search` | **LLM** (active judgment) | The LLM calls it during the conversation when it judges that "I need to look this up", searching memories + session_checkpoints at the same time | Cross-session knowledge retrieval, supports historical conversation summaries | Depends on the LLM's judgment; may miss things |
| Short-term memory variant | `memory_checkpoint` | **Scheduled task** | Triggered in the background when the user is inactive (e.g. early morning) | Does not affect user experience, uses idle computing power | Requires scheduled-task infrastructure |
| Short-term memory variant | `memory_checkpoint` | **Streaming computation** | Triggered when the session is idle and the cache is about to expire, renewing the checkpoint | Prevents the cache from expiring | Requires monitoring session activity |
| Short-term memory variant | `memory_checkpoint` | **Graph clustering** | Triggered when a topic switch in the conversation is detected, compressing along semantic boundaries | Clearer summaries | Requires topic-detection capability (graph memory) |

### Active Triggering (Long-Term Memory)

Prefix Checkpoint's memory control is used not only for compression but also for **active memory** — the LLM calls the `memory_store` tool on its own initiative during the conversation when it judges that "this is worth remembering", storing it into the `memories` table.

The trigger is **user request or implication**:
- The user explicitly says "remember this" or "do it this way from now on"
- The user expresses a preference, habit, or decision (which the LLM judges worth keeping long-term)
- The LLM discovers an important fact or context

The LLM does not proactively capture knowledge produced during execution — it filters with judgment and stores only valuable information: precise, but dependent on judgment.

### Other Trigger Mechanisms

**Nightly scheduled trigger**: when the user is inactive (e.g. early morning), a background scheduled task triggers compression. It uses idle computing power to complete compression without affecting the daytime user experience. Suitable for scenarios where compression is expensive (long conversations, large models).

**Long-idle trigger (streaming computation)**: compression is triggered when the session has had no activity for a long time, and it can be combined with the cache expiry time — proactively renewing just before the cache expires. In essence this is **streaming computation**: continuously maintaining a hot cache to prevent cache invalidation from forcing a full recomputation on the next conversation. It works together with Prefix Checkpoint: after compression the new checkpoint serves as the prefix, and the cache hits again.

**Topic-switch trigger**: compression is triggered when a switch of conversation topic is detected. Compressing along semantic boundaries yields clearer summaries ("the first half discussed architecture, the second half discussed deployment" is more useful than "a mixed summary of 100 messages"). It requires topic-detection capability and works together with graph memory — the cluster boundaries in the graph are natural topic boundaries.

These are the memory system's six trigger modes, grouped by memory type:

| Memory type | Mode | Timing | Who triggers | What is stored |
|:--|:--|:--|:--|:--|
| **Long-term memory** | Active trigger | During the conversation | **LLM** (active judgment) | What the LLM judges worth remembering → `memories` table |
| **Short-term memory** | Prefix Checkpoint | When the threshold is reached | **Memory system** | Conversation summary → `session_checkpoints` table (parallel side channel, not on the critical path) |
| **Memory retrieval** | Active trigger | During the conversation | **LLM** (active judgment) | Searches `memories` + `session_checkpoints` |
| Short-term memory variant | Nightly schedule | When the user is inactive | **Scheduled task** | Same as Prefix Checkpoint |
| Short-term memory variant | Long idle | Session idle + cache about to expire | **Streaming computation** | Renews the checkpoint |
| Short-term memory variant | Topic switch | Semantic boundary | **Graph clustering** | A summary segmented by topic |

### Injection Methods

| Method | Mechanism | Cache friendliness |
|:--|:--|:--|
| **Full injection** | Inject all history into the prompt every round | Poor (the content changes every round, invalidating the context cache) |
| **top-K retrieval** | Retrieve relevant memories and inject them every round | Medium (retrieval results may change) |
| **checkpoint + delta** | Immutable checkpoint + the most recent N messages | High (the checkpoint is fixed and can be cached permanently) |

#### checkpoint + delta mechanism

Similar to the snapshot pattern of event-sourcing. Messages are written one by one into the session_messages table, and when the threshold is reached they are compressed into a checkpoint (written into the session_checkpoints table); all subsequent uncompressed messages are then the delta:

```
msg_001 ... msg_100    ← messages appended to the session_messages table
              ↓ threshold reached (100 messages)
ckpt_0: summary="The user discussed project architecture and decided to use SurrealDB..."
              ↓ written to the session_checkpoints table, immutable
msg_101 ... msg_150    ← all delta (all currently uncompressed messages)
              ↓ threshold reached again
ckpt_1: summary="..."
msg_151 ...            ← new delta
```

**Read flow**:

Messages are appended to the prompt one by one (rather than being re-concatenated every time). The LLM API's multi-turn conversation mechanism natively caches all preceding tokens:

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← the preceding tokens go through the context cache
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: threshold reached → fork a compression side channel off the invariant prefix (not on this round's critical path)
Turn N+1: [ckpt_1] + [user question X] + [Agent answer Y]     ← the new prefix is in place
```

**Write flow**:
1. Each message is appended to the session_messages table
2. Check whether the number of messages since the last checkpoint is >= the threshold
3. Threshold reached → mark that compression is needed → when the next user question arrives, the compression instruction is issued as the tail prompt of a **parallel side channel** (not in the main branch's prompt)

**Prefix Checkpoint**: after the threshold is reached, the memory system forks two read branches in parallel (see the "Compression is a side channel parallel to the mainline" section):

```
In the context cache (shared by both branches, invariant):
  [ckpt] + [msg_101..msg_150]

Main branch (user critical path):                + [user's question]       → answer Y
Compression side channel (parallel, background): + [compression instruction] → summary S

Side-channel output: tool_call: memory_checkpoint(session_id, summary)
  ——once S lands, the session before U_{n+1} switches to [S] + [recent delta]
```

The two branches each pay their own delta, and the prefix KV is computed only once; the main branch does not wait for the side channel, and the side channel does not enter the mainline.

**History purity**: the compression instruction exists only in the side-channel request's prompt; the main branch's user messages are never modified and never written into the session. What the session always contains is pure history:

```
What is stored in the session:
  [checkpoint_1] + [user original message X] + [Agent answer Y]

Rather than:
  [checkpoint_1] + [tail prompt + user original message X] + [Agent answer Y]
```

Even when the side-channel request carries the compression instruction, nothing remains in the session — the compression instruction never enters the main branch's prompt: user messages are buffered in memory by `push_user_message`, added only when `get_context()` builds the context, and cleared after `write_assistant_message()` writes to the DB. What the session always contains is pure history:
- History is replayable — reload the session at any time and the content is the real conversation
- No pseudo-instructions — no system instructions remain in the session
- Logically coherent — checkpoint + original messages + answers form a complete causal chain

**The mainline and the side channel each do their own job**: the main branch's first output is the answer — the user never first sees a pile of tool calls running. The side channel's tool_call flows back only into Krystallizer's append path, not into the conversation stream. The ordering constraint of the old form, where compression was mixed into the turn ("answer first, then handle memory"), does not exist in the side-channel form: the two tasks live in different requests and need no ordering.

**Why it is designed this way**:
- **No modification to the Agent loop** — compression goes through the side channel's normal tool call, under the memory system's complete control
- **The cache hits naturally** — the two branches share the prefix KV and each pays only its delta
- **Invisible to the user** — the answer latency equals that of an ordinary round; compression and memory extraction are completed in the branch beside it
- **History is cleaned up automatically** — once the side channel lands, the old messages are no longer needed and the history becomes `checkpoint_1 + X + Y`

**Why it is cache-friendly**: once a checkpoint is written it is immutable, and all subsequent turns share the same checkpoint text. The LLM API's prompt caching caches the checkpoint portion in GPU memory, and only the delta portion changes each time. As delta messages accumulate and the next checkpoint is triggered, the delta is compressed into a new fixed summary and the cache hits again.

**Comparison with Agno's sliding window**: Agno's `add_history_to_context` reads the most recent N complete messages from the DB every time. As new messages arrive, the composition of those N keeps changing (old ones are squeezed out, new ones are added in), and the context cache is invalidated every round. Under the checkpoint model, history is compressed into a fixed summary, only the uncompressed delta portion changes, and the cache keeps hitting.

### Framework Adaptation

**Iron rule: the mem module is thick, the agent adaptation layer is thin.**

The adaptation layer makes only three calls and implements no memory logic:

```python
# Adapter layer (~20 lines)
memory.push_user_message(session_id, user_message)   # buffer the user message
context = memory.get_context(session_id)              # checkpoint + DB + pending + tail prompt
agent.additional_input = context                       # inject into the agent
agent.run()
memory.write_assistant_message(session_id, assistant)  # write to DB + clear pending
```

Compression detection, tail-prompt injection, and pending management are all inside WorkingMemory. When switching frameworks, only the adaptation layer (~20 lines) is rewritten, and the Engine is fully reused.

Above framework adaptation lies the stateless session-as-data pattern — the three calls contract to "fetch session → execute → store session", the compression decision sinks into Krystallizer, and execution goes through the parallel side channel; see the "Compression is a side channel parallel to the mainline" section of this document and [Krystallizer](krystallizer-en.md).

**Ships with a CLI-driven loop**: Gravity provides a local CLI form — an in-process loop executing turns (fetch session → run → store, next turn), which is the local/standalone mode; when the same Gravity function is registered as a stall type in Aura, each turn event triggers one pass of execution, which is the distributed mode. The same execution function, three drivers: the CLI for loop (local), the Aura stall (distributed), function compute (serverless) — the CLI is the function's for loop, and the stall is the function's single invocation. Booth form = an **induction native child process** (2026-10-13 carrier ruling, superseding both the early python carrier and the wasm artifact forms): Gravity is compiled to a Rust guest binary speaking the stdio pipe frame protocol (`~/world/induction`), streaming frame by frame to stdout with the `done` envelope closing the boundary (single-call and stream share the one envelope shape); the stream state lives in the guest process — the process IS the stream state. Not compiled into the aura binary — Aura is a platform, not an application framework; an Agent application gains features by registering new booth types, not by repackaging the engine; storage is lifted up through ctx store frames and carried by the host (Krystallizer ADR-0007 storage-carrier split).

In Aura, Gravity is a stall type: the same session is serial (stall single-threaded semantics, partition key = session_id), and different sessions run in parallel; between turns the default is scale-to-zero, and `on_sleep`/`on_wake` degenerate into the two actions of store and fetch — retention-period residency is a refinement of this default: within the retention window, turns of the same session reuse the execution body, and only on timeout or explicit release do they fall back to the two actions of store and fetch (see the Unified Invocation Model section). Streaming output is turned into SSE pushes via high-frequency emit events — the transport surface is carried by Prism's WS gateway (between Gravity and Prism there are still domain events, with no direct connection).

### Effector: The Executor

Effector is the execution environment for operations — **execution provides only a runtime, and does not execute inside Krystallizer**. Skills are invisible to Effector: a skill is a subgraph emerging in Krystallizer's graph, and Gravity drives the LLM — the LLM selects the operation and generates the parameters, and this selection is the execution of the skill; what Effector receives are only the operation and the parameters (plus any code the operation carries). Isolation model: **Effector itself is packaged as a container** (base image + dependencies installed on demand), and isolation is cut per node, not per skill — operations within the same container share its filesystem, and the "restricted world" is enforced by the capability surface (application-layer checks), not by container boundaries. This holds under the realm isolation mechanism (Phase 3.6, construction-time prefix isolation, where the binding dimension is an application decision — cutting by user is gravity's choice, not a framework preset; not cutting by skill; renamed from namespace by ADR-0028); per-skill isolation is to be revisited only when multi-tenant shared nodes become a real requirement. **Effector binds as a stall type's execution affinity** (the type declaration points to a node alias); the control plane's calls to it go through the standard invoke route, no different from an in-domain stall.

**Effector binding = type affinity, trust = node identity (Aura PLAN 4.10 / ADR-0015 ruling)**. Effector binds to the stall TYPE (affinity is metadata: which Effector node the type declaration points to, and the registry records the stall→effector binding), never to a user — "effector registration credential = user credential → derive namespace (now named realm)" has been superseded: that welds the tenant assumption (that a user exists) into the base layer, whereas user-less applications (an intranet compute grid: one effector per node, with the stall side sharding tasks) are first-class citizens. The trust problem is separated from the user problem: "this machine may execute" is a deployment-layer ruling, resolved through a node identity handshake (ed25519, ADR-0015 — the node generates a key pair locally, the gateway challenges and it responds, and an alias conflict is rejected rather than taken over; 2026-09-25 attribution revision: the handshake/registry/approval endpoints live in the **prism gateway** (prism PLAN Phase 1.8), and what has landed on the aura side is takeover visibility + a startup disclosure of the unauthenticated posture — registration still discards credentials, so the protection is the network boundary); "whose request is this" is an application-layer ruling, with identity carried along in payload metadata (gravity distinguishes users itself — it may organize types/instances by user, while the framework presets no user dimension). The realm isolation mechanism is retained (construction-time prefix isolation, inexpressible across realms), and which dimension to bind is the application's decision. **tool target resolution = type affinity + node alias**; "send a file from the home computer to the office computer" is an orchestration of two invokes (home reads → office writes), with the orchestration logic in Gravity/LLM and the execution location in the registry, the two being orthogonal — Gravity does not distinguish remote from local; the distinction happens at the target-resolution layer.

**The data path is left to operations and the user environment.** The control plane carries only instructions and result summaries: a Result is a message and stays small; the large artifacts produced by tool execution (files, binaries) do not enter the control plane — operations dispose of them on the Effector side themselves (local filesystem, user-configured transfer tools, declared transfer-type operations). The reachability requirement for cross-machine transfer (direct connection/VPN) is a declaration in the Gravity-side skill metadata, and the control plane is unaware of the data path, so Aura stays ignorant of storage details. AI-generated function-call parameters are instruction semantics (paths, options, a small number of snippets) and are naturally limited in magnitude; the control plane needs only a loose message size limit to guard against anomalies, and this constitutes no data-plane design.

Two deployment forms, equally supported:

- **Embedded in Aura**: as part of Aura's execution base (the heavy-isolation end of the Wasmtime sandbox lineage — where Wasm cannot manage a real filesystem/real network/system packages, containers step up), reachable by in-domain calls.
- **Remote executor**: deployed on the user's own computer or a target server, it is that machine's operation executor: wherever it is deployed, that is what it can operate. A machine under an intranet/NAT has no inbound reachability, and the only feasible topology is an **outbound long connection**: on startup, Effector proactively initiates a connection to the control plane and registers (I am online, here is what I can do), then keeps the connection alive, and tasks are pushed down along the connection by the control plane (WS frames). The connection direction is outbound and the data direction is downstream push, with no inbound port opened — the inbound topology of the network where Effector resides is irrelevant. Long polling (repeatedly asking over HTTP) is a weakened implementation of this mode.

**The connection surface is the Effector stall's transport adapter, not a side channel.** The WS connection wraps the outbound long connection into the Realm's event-dispatch semantics: a downstream frame = writing into the event queue of that Effector instance (the effector stall is the single subscriber to its own command queue), and an upstream frame = that instance's return (reply_to backfilled, travelling the same delivery channel through `resolve_call` as the HTTP response and the stall return). The last hop of `ctx.invoke("effector:<node_id>:<tool>")` lands on the connection surface; what Gravity writes is just a standard stall invocation. Serial processing of the same node's tasks is obtained for free from the serial consumption of the per-subscription cursor; timeouts/errors reuse the `pending_calls` deadline scan.

**Skill distribution: resolved in real time on every tool call, zero caching.** The premise of emergence is a zero-staleness window — once an instance hits a pitfall and solves it, the solution is stored in the graph, and the next execution anywhere immediately gets the new version. The skill lifecycle aligns to tool-call granularity, isomorphic to "there is only one mode of invocation": skill resolution is an ordinary read, not a caching problem that needs an invalidation strategy. The dividing line: resolution happens on the Gravity side (Krystallizer → Gravity, where the weight write-back of the emergence loop also occurs); Effector is unaware of skills, initiates no pulls, and holds nothing between two calls — whatever is in the task frame it receives is what it executes.

**Two forms of operation code: inline and linked.** Ordinary py/steel scripts are small (KB-scale) and are sent inline directly with the task frame, with zero extra round trips. Wasm artifacts may reach MB scale, and inlining would bloat the task frame — the control plane may provide an **optional HTTP port** for delivering large artifacts: the URL carries a content version number (the content hash of the code), which the CDN/caching layer can use to cache — this is CDN-friendly caching, not Effector-side code caching (the zero-holding ruling is unchanged: when the content changes the version number changes, and the URL is the invalidation strategy). The two forms are declared by the task context: `inline` (bytes in the frame) or `link` (URL + version number + expected hash).

**Extreme-environment fallback: use the channel when only WS is allowed.** An intranet policy may forbid arbitrary outbound HTTP and permit only established WS connections — in that case the `link` form degrades to chunked delivery through the WS channel (continuation frames of the same frame protocol), and Effector need not perceive the difference: it consumes whatever the task context declares, and the degradation is a decision made by the control plane when assembling the task frame (probing/configuration knows whether that node can egress over HTTP), not a runtime judgment by Effector.

**Resolution path: Krystallizer → Gravity, not through Effector, and certainly not a direct connection.** Three reasons: access control — Krystallizer needs to trust only Gravity, one layer, and the container (which executes untrusted code) holds no data-plane credentials; network topology — a remote Effector is only outbound-reachable from the control plane and may not be able to connect directly to the storage network; injection point — this is where Gravity does view processing and the `tool_invoke_count` weight write-back, the data gate of the emergence loop, which would be broken by bypassing it. There is no pull action on the Effector side: the task frame it receives is a finished product assembled by Gravity. The HTTP large-artifact port is the exceptional form of this path: the **decision** is entirely in Gravity (whether to use it, and which version), and only the **byte stream** reaches Effector directly via the CDN side path — the version number + hash check ensure that the side-path bytes agree with the decision.

#### ToolRunner: the execution surface is a node mount (gravity ADR-0001)

**Execution lands in exactly one place: a mount on a node.** One effector node per machine (the control plane's machine included), one mount per (node alias × language), whose source is a **fixed deployment artifact** — a runner (interpreter or dispatcher) that receives the operation's code and arguments as call arguments and executes them. The mount's code never varies per skill or per version; everything dynamic is data. An emergent skill therefore requires no registration act at all: it stays in the k10r graph, Gravity fetches its script and hands it down with the call (`invoke(mount, key, entry, args{code, args})`) — no type, no ns, no definition row, no hot-swap on re-registration. The target shape (node alias + capability) thereby lands on a concrete executable form.

**A skill script is not "code delivery".** The two code forms (inline / linked) describe the mount's own definition — the runner is a definition (content-addressed + reference, ADR-0027 standing as written); a skill script travels as an argument and never enters that machinery, so no inline variant is needed for it. The "resolution path Krystallizer → Gravity" rule becomes harder here: a node never talks to k10r; the byte path is Gravity → control path → node.

**One node per machine** is the literal form of "deployed where, can operate there": a machine's abilities are its node manifest (carriers, sandbox policy, capacity). The cost is that execution ability becomes a property of the domain — Gravity holds no independent execution surface, so a self-contained install is achieved by packaging (domain + node + Gravity shipped together).

**The language surface is small, and nothing degrades to bash.** The only required installs are nushell and python (steel is compiled into the node binary); systems-level operations always route to nu mounts — nu's built-ins dispose of quoting, invocation, and portability trouble up front — while python carries business logic. There is no shell carrier, and "a language with no mount cannot be called" is the enforcement: an operation's language is a property of the skill (the script is data, and data has a language), never something the LLM picks at call time.

**Three-way split of application forms (under discussion).** The four-way component split presupposes session-type applications; non-chat applications (mudra, scratch and other memory-primary applications) have no turn sequence, so the session projection is absent: Surface 2 (the fact graph) is the entire memory plane, and Surface 1 is reused keyed by task id only when a forkable turn sequence exists. The application's own structured state (task instance state, state fields) does not enter the memory pipeline — exact enumeration belongs to the application's own tables, and only approximate retrieval belongs to the graph. No new interface surface is needed: the core has no policy boundary to validate.

### Unified Invocation Model: CallSlot

A local call (an in-domain function) and a remote call (tool execution on an Effector) are unified at the framework layer into the same model: initiate → associate by call_id → backfill. This model is not a new mechanism — **it is Aura's `ctx.invoke()`** (oneshot + the `pending_calls` table + the `reply_to` mechanism, see [Aura Architecture](aura-architecture-en.md) §5.14): on initiation a pending call is registered and the call_id enters the task context; when execution completes, the entry is found by call_id, the value is placed into the oneshot, and the initiator completes the call. The caller (the Gravity execution body) is unaware of the execution location — the target is dispatched by the `invoke.toml` registry: an HTTP service, an in-domain stall, and a remote Effector are the three kinds of entries in the registry, all behind the same API. This is the implementation of "there is only one mode of invocation" at the invocation layer: once the invocation mode is unified, transport needs to be ruled on only once. Error handling follows the established ruling — a failure is placed into the oneshot as a value (Result), with no second channel opened.

**Two levels of waiting**: the waiting end is split by the nature of the call, not a global either/or. At the call site there is always only one line, `slot.wait().await`, and the runtime splits by declaration before the block occurs — the split point is at the entry, not in the middle of waiting.

- **Hot path (in-memory suspension)**: the tool-call loop within a turn is a high-frequency operation — LLM returns a call → execute → backfill → next inference, and a single turn may do this a dozen-plus times. The runtime registers a waker on the pending call, the task parks, and when the result arrives the waker fires and the value is backfilled. This is the actual content of "blocking": **the task parks rather than the thread blocking** — an async task that stops occupies only memory, not an execution thread, and a single control plane can park thousands of turns with no cost problem. Zero persistence. The implementation is in two layers: when the transport has an explicit peer it is a oneshot — `recv().await` internally stores the current task's waker when not ready, and `send()` wakes it, which is exactly the Async responder of `ctx.invoke()`, with no need to reinvent the wheel; only when arrival is event dispatch (the domain event bus, with no explicit channel peer) does it fall back to a bare waker/Notify. To the caller both are the same one line of `await`.
- **Cold path (event-source suspension)**: calls that reach a human (permission confirmation) or an external system wait for minutes or more. The runtime **does not let wait enter park**: it writes the transcript to disk, returns pending to the turn driver, the task ends, and the execution body is released (scale-to-zero), with the suspended state written into the session event stream ("turn T waiting for call C"); when the result arrives, the framework locates the suspended turn by call_id, places the value into the context, re-enters execution, and continues from the transcript breakpoint — calls that already succeeded are not re-run, and the transcript is breakpoint state, not an execution log. After a process crash and restart, the suspended turn is still in the event stream and is naturally recoverable.

**No suspend/continue instruction — explicitly rejected.** "Send a suspend instruction to the channel and the executor suspends on receipt" means the runtime confiscating a blocking continuation mid-flight, which implicitly requires the continuation to be serializable (the Erlang hibernate / VM snapshot kind) — snapshotting the local variables of a general future is infeasible in Rust, and retrofitting the execution model for it would be extremely expensive; and in a structure with declared splitting it has no reason to exist: a cold call is intercepted at the entry and never blocks. An unexpected long wait in a hot call (a network hang, an external service dragging things out) is also not escalated into suspension — **a pending call timeout = a failure value** (the Result is backfilled through the oneshot, and the LLM decides whether to retry or give up); escalating an undeclared unexpected long wait into suspension would amount to hollowing out the declaration mechanism.

**Escalation is driven by static declaration, not runtime guesswork**. Every tool declares its execution nature at registration time (idempotent quick return vs. reaching a human/external system) — what is declared is the tool's nature, which does not vary with the call. For a tool with no cold calls, the entire loop is the hot path; for a tool with a declared cold call, escalation is triggered only when that tool is called. The declaration travels along with Gravity's tool registration information, the same as `tool_invoke_count`, without building a new system.

**Execution-body lifecycle: retention-period residency, replacing single-pass release**. Purely single-pass turns the hot loop into a storage storm (one store/restore per tool call), while purely resident returns to a stateful service. The compromise: the turn execution body resides in memory during the retention period, and consecutive tool calls in the same session go through in-memory oneshots; at turn end or on retention timeout the session is stored and the body released; a new turn in the same session within the retention period reuses the resident body (saving the session fetch). The stateless semantics are not violated — they mean **session state is externalized** (the execution body holds no session state; the state is all in Krystallizer), and the resident body is a hot cache that can be discarded at any time, rebuilt from the event stream after a crash, with only the cache lost. The only persisted delta on the Aura side is the single field call_id (which enters the task context/event stream and is used by the cold path to route back to the suspension point); the `pending_calls` table is a purely in-memory structure that lives and dies with the resident body.

**History and transcript are separated**: the tool-call loop within a turn maintains a working transcript (the transcript: what the LLM needs for the next round of inference — calls, errors, retries), living in the resident body's memory and dying with the retention period; the session history receives only the net effect — the final result is written at turn end, and intermediate failed attempts are process, not memory. The hot path does zero writes throughout; the cold-path escalation is the only moment of writing, and trimming is completed before writing (history gets the final state, the transcript gets the minimal set needed for replay). The root cause of retaining everything is "not knowing where the suspension point is" — once the suspension point is given by the tool's static declaration, trimming changes from hesitation when writing history into a one-off ruling at escalation time.

### skill Emergence Loop

A skill is not a static file but a subgraph of high tool index in Krystallizer's graph (8-slot-tuple atomic facts + weight system). The loop:

```
Execution experience (pitfalls/solutions/operation sequences) → stored in the graph (atomic facts)
        ↓ cluster emergence
High-weight subgraph = skill boundary → discovered at Gravity runtime (vector search + weight ranking)
        ↓ tool call
Gravity assembles the task frame and dispatches it → Effector executes → tool_invoke_count writes back the weight → affects the next ranking
```

The raw-material loop of emergence reuses Krystallizer's existing mechanisms (weight delta counting, graph clustering) and builds no new system.

## Transport Ruling: WS at the Entry, No Connections Internally

WS is used in two stateful places. The first is between Prism and the client (user to entry; a long connection naturally suits an interactive session). The second is scope-limited: **WS does not enter the in-domain execution path** — between in-domain Gravity and Prism/Effector there are Aura domain events, with no direct connection; the turn's execution body holds no connection, and connections are pinned to resident components. The remote Effector is the exception: an intranet machine has no inbound reachability, so its outbound long connection is the only channel by which the control plane can reach it, and tasks are pushed down along that connection — this WS is pinned to the control plane's connection surface, the same ruling as the entry WS; what the turn's execution body receives is a call coming down the connection, and it itself remains single-pass (retention-period residency, see the Unified Invocation Model section).

This ruling replaces the previous "HTTP + SSE by default" plan. The reason for the revision: the problem of connection state is not WS itself but **where the connection is pinned**. Prism is built on Aura (resident, with multiple instances scheduled by the domain), so keeping the connection in Prism means WS load balancing and reconnection are solved once and for all by Aura's connection surface and do not seep into the execution layer; the HTTP+SSE plan, by contrast, would have each component expose its own HTTP endpoint, fragmenting the entry protocol. With the CLI wrapping WS, all clients (people, CLI, scripts) go through the same protocol, and **there is only one mode of invocation** holds at the transport layer as well.

Mid-turn interaction is still dissolved by the turn model — a person interjecting while a turn is in progress = new input → a new turn (appended to the session, with convergent handling of concurrency ruled by Krystallizer's session control primitives); permission confirmation is a pending turn in the session, not a privileged frame on the WS.

## Comparison with Conventional Agent Architectures

| Dimension | Conventional (resident process) | This architecture (stateless) |
|:--|:--|:--|
| Session holding | Process memory | Krystallizer (append-only + checkpoint) |
| Loop form | Resident loop, accumulating in memory | Pure function `f(session, input) -> session'` |
| Multi-instance | Sticky sessions / state synchronization | Free — a turn lands on any instance |
| Suspension recovery | Home-grown persistence | scale-to-zero = the two actions of store and fetch |
| Deployment | Resident service | Local loop / Aura stall / function compute, isomorphic |
| Tool execution | In-process or RPC | Effector (container), skills emerging in real time |
| Transport | The common WS long connection running through the execution layer | Entry WS (Prism), no connection on the execution path |

## Cross-References

- **[Krystallizer](krystallizer-en.md)**: the memory system implementation — session control primitives, full mode, view-layer trimming, skill graph.
- **[Memory Architecture](agent-memory-en.md)**: general analysis of memory architecture — the two layers of Surface/Engine, injection methods, comparison of external open-source solutions.
- **[Aura Architecture](aura-architecture-en.md)**: the engine base — domain model, stall semantics, MQ decomposition, multi-language embedding topology.
- **[Graph Memory](graph-memory-en.md)**: the conceptual design of skill emergence — the computation-timing spectrum, clustering strategies, the weight system.
