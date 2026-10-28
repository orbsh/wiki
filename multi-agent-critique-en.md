# The Multi-Agent Pattern: A Critique

> **Languages:** [English](multi-agent-critique-en.md) (primary) · [中文](multi-agent-critique.md)

**Status**: Technical consensus
**Date**: 2026-06-22
**Core thesis**: The failure rate of multi-agent systems (MAS) in production environments is 41%–86.7%. The cause of failure is not model capability, but "organizational structure" and "communication mechanisms". If it can be solved with a single agent + multiple tools, never blindly go multi-agent.

> Multi-agent systems often do not lose on model capability, but on "organizational structure" and "communication mechanisms". —MAST failure taxonomy research

---

## 1. The Three Core Pain Points

### 1.1 Loss of Coordination Control and Topological Fragility

In free-form dialogue (Swarm / group-chat mode), multiple agents easily fall into infinite loops (Agent A and B passing the buck back and forth) or role drift (the agent responsible for Code Review suddenly goes off to write code).

**The fatal wound: error cascades**. An upstream agent outputs a tiny hallucination; the downstream agent takes it as "truth" by default and keeps reasoning onward, and the final result is wrong in a thoroughly coherent way, extremely hard to troubleshoot.

### 1.2 Performance and Cost Avalanche

- **Serial latency**: a single agent produces a result in 2 seconds; the multi-agent chain "plan → code → review → error → re-plan" sees latency rise exponentially (often 30 seconds+)
- **Token bloat**: the contexts synced between agents carry a large amount of redundant history, and most of the cost is wasted on "idle chit-chat"

### 1.3 The Observability Black Box and the Attribution Dilemma

When traditional software errors out, there is a clear Stack Trace. When a multi-agent system errors out, what you see is code with perfect formatting, a confident tone, and completely wrong. Lacking end-to-end link tracing, you cannot determine which agent, at which step, lost the context.

---

## 2. The Academic Ironclad Evidence: Single Agent Crushes Multi-Agent

### 2.1 After Compute Is Aligned, Multi-Agent Has No Advantage At All

A controlled-variable experiment from Stanford/UC Berkeley: allocate the same Token budget to a single agent and to a multi-agent system.

**Conclusion**: the so-called "multi-agent performance improvement" is purely because it consumes more compute by cheating through multiple roles. Once the Token count is aligned, the single agent (Rich Context) crushes the multi-agent system in both accuracy and efficiency on most tasks.

**The scientific principle: the data processing inequality (Data Processing Inequality)**. When information is "relayed" between multiple Agents in natural language, the information is lossy due to models' comprehension bias (the telephone game). A single model reasoning directly inside its own Context achieves the highest information density and logical coherence.

**The knowledge-accumulation perspective**: the lossy transfer of information between Agents is essentially the "accumulation lag" dilemma magnified within a multi-agent architecture — knowledge decays with every transfer, just as documentation cannot keep up with code. See [Agent Principles: Threefold Knowledge Dilemma](agent-principles-en.md).
A systematic treatment of the threefold knowledge dilemma is in [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md).

### 2.2 Role-Playing Is a False Proposition

If the underlying models are homogeneous (say, all GPT-4), multi-agent frameworks (CrewAI, AutoGen) do nothing more than automatically write the Prompts "now you are the architect...", "now you are the code reviewer..." for you.

**The OneFlow framework experiment proves**: a single model, through internal "self-correction (Self-Correction)" and KV Cache reuse, can match or even surpass the code-generation quality of a multi-agent system at 1/10 the cost and with extremely low latency.

### 2.3 The More Tools, the Faster Multi-Agent Dies

Google Research's in-depth test of multi-agent scalability: when tasks involve complex APIs and tool calls, multi-agent performance falls off a cliff (negative correlation coefficient -0.27).

**The reason**: Agents spend too much time agonizing over "who should call what tool when", and the coordination overhead (Coordination Overhead) directly drags down reasoning capability.

---

### 2.4 Information Transfer Loss Is the Most Fundamental Problem

The three core pain points are only symptoms; their common root cause is **information transfer loss**: information is distorted once with every transfer. This is not a defect unique to LLMs — the telephone game between people, and the distortion of level-by-level reporting in bureaucratic organizations, are universal laws of communication media (in Shannon's sense, every hop has noise, and it cannot be eliminated by "rephrasing"). In the LLM context it is sharply magnified: natural language is the only interface, there is no read-back confirmation, and Context truncation silently drops information, so the loss rate per hop is far higher than in well-trained human communication.

Corollary: the quality upper bound of a multi-agent system is determined by "number of hops × per-hop loss rate", independent of the number of agents. The data processing inequality (§2.1) is the formalized statement of this law; role-playing (§2.2, §6) does not reduce the number of hops — it turns the single agent's zero-hop reasoning into multi-hop relaying.

### 2.5 The Essence Is a Question of Parallelism

Strip away the packaging of "multiple AI employees holding a meeting", and multi-agent has only one real demand: **increase parallelism**. And parallelism does not require inter-agent communication — its requirements are that tasks be decomposable and results mergeable. The message-passing paradigm welds "parallel execution" and "coordination communication" together into the same mechanism (natural-language dialogue), so coordination loss eats up the parallel gains. The correct approach is to decouple: execute in parallel, coordinate through shared structures.

**Environment-centric: agents do not address each other.** This coordination form presupposes abolishing the "notification" layer. Notification-style collaboration (RAP, message queues, pub/sub, A2A-class agent interconnection) is at bottom "I call you, you subscribe to me": both sides must address and perceive each other, and each collaboration is a channel that must be maintained independently. As the count grows, the number of channels is itself the topological disaster. The matrix instead demotes the coordination medium to a shared environment — the atomic-fact graph and the task graph. Collaboration is not me telling you something but me changing the environment (writing a fact, opening a task vertex), and you seeing the change naturally when you extract context for your own goal. The writer does not know who reads; the reader does not know who wrote; there is no addressing, no subscription, no delivery, and the hop count is zero. "Who is waiting for whom" also disappears from the system — with no waiting, there is no timeout, retry or idempotency semantics to maintain.

**Context is derived per goal, not piled up.** With no notification there is no need to fill the context window with "who said what and which tools exist": before each turn the projection is derived from the graph against the current goal ([Krystallizer](krystallizer-en.md)). Tools and knowledge are looked up on the spot, not a list pasted into the window once — the longer the list, the more easily tool routing collapses.

**The boundary**: emit/on inside an Aura realm is the engine's in-process runtime event surface (stall decoupling, routing, the persistent queue), not an inter-agent collaboration surface. The event name is a runtime addressing mechanism and carries no semantic negotiation between agents; reading the two as one thing yields the contradictory impression of "rejecting subscriptions while living on on()".

The corresponding deterministic solutions already have a complete stance in the wiki; here we merely consolidate them:

- **Topology replaces relaying**: deterministic orchestration (§4.1, §4.2) — master-slave delegation, FSM transitions, strong-Schema messages.
- **A task graph as the coordination medium**: the graph replaces message passing; the N(N-1)/2 channels of pairwise communication among N agents converge into N-to-1 ([Graph Memory §Task Networking](graph-memory-en.md#task-networking-task-forest)).
- **Parallelism comes from the graph topology**: task nodes with no strong dependencies are naturally concurrent after topological pruning; the unit of parallelism is the graph's branch structure, not message orchestration ([Graph Memory §Task Networking](graph-memory-en.md#task-networking-task-forest)).
- **Result merging goes append-only**: facts are appended into the graph as deltas, the write path has no conflicts, and logical conflicts are resolved at read time ([Graph Memory §Read-Time Conflict Resolution](graph-memory-en.md#read-time-conflict-resolution)).

**The number of agents is not a design variable.** What is needed is parallel execution plus shared structure: the unit of parallelism is a branch of the task graph (topology sets the degree of parallelism; no scheduler doles out slots), and the unit of coordination is an increment of the fact graph (append is merge). Treating the agent count as a tunable parameter to scale up scales the coordination cost with it — §2.4's hops and the coupling described above get amplified together.

The deterministic orchestrations above (topology, strong-Schema messages) are the convergence plan "when multi-agent is unavoidable": they turn uncontrollable group chat into auditable channels. The matrix interior goes further — it sets up no channel at all, because homogeneous agents have no semantics to negotiate ([MCP Critique §The Same Class](mcp-critique-en.md)).

---

## 3. The Only Scenarios Where Multi-Agent Holds

### 3.1 Genuine Heterogeneous Model Collaboration

The "programmer" uses Claude 3.5 Sonnet (good at writing code), and the "test runner" uses a locally fine-tuned small Llama-3 model (cheap, fast). A division of labor made because model capabilities differ and costs differ is meaningful.

### 3.2 Parallel Tasks and Physical Isolation

Crawling 100 websites at once, running 100 concurrent unit tests — use 100 read-only subagents to do the dirty work, and the master control aggregates. In essence this is the AI version of distributed computing.

**The criterion**: whether there are data dependencies between subtasks. No dependency → parallelizable → multi-agent has value. Dependency → serial → a single agent is better.

---

## 4. If Multi-Agent Is Unavoidable: A Hard-Constraint Architecture

### 4.1 Abandon Free-Form Group Chat, Switch to a Deterministic Topology

- **Orchestrator-Worker mode (Orchestrator-Worker)**: a high-capability model acts as the Orchestrator and decomposes tasks; the underlying Specialists are isolated from one another and are not allowed to chat laterally — they may only report to the master control
- **Finite state machine (FSM) / graph architecture**: hard-code the inter-agent transition logic (after A completes it must go to B; after B fails it can only return to A, looping at most 3 times, otherwise force an exit)

### 4.2 Machine-Readable Communication Protocols

- **Strong Schema constraints**: forbid agents from sending plain-text essays to one another; enforce JSON or Function Calling
- **Speech Acts (speech act theory)**: inputs must carry explicit tags such as `[PROPOSE]`, `[CRITICIZE]`, `[REFINE]` for monitoring at the code layer

### 4.3 Control-Plane Governance

- **Full-link Span tracing**: score the fidelity of every LLM call and the correctness of tool use in real time
- **Checkpoint persistence mechanism**: when an intermediate step fails, persist the state so that after human intervention it can resume from the checkpoint rather than starting over entirely
- **Human-in-the-loop**: set up "approval gates" at key transition nodes; flow can move downstream only after a human expert signs off

---

## 5. Industry Cases

| Case | Architecture | Core solution | Outcome |
|------|------|---------|------|
| **Anthropic Research system** | LeadResearcher + Subagents (hub-and-spoke) | The core Planning is written into persistent shared memory; subagents search in parallel and the Lead aggregates uniformly | Solves context truncation and memory degradation after 200K+ tokens |
| **PwC compliance audit** | Strongly deterministic workflow | Human-in-the-loop + Living Specifications + approval gates | Accuracy in complex audit scenarios improved 7x |
| **Grid Dynamics financial platform** | Temporal durable workflow orchestration | Treats multi-agent conversations as distributed transactions, with automatic timeout retry and compensation | Business-flow analysis shortened from 4-6 weeks to a few hours, saving $9-14 million per year |
| **JCode (Rust-native coding Agent)** | File watching + structured message bus + master-slave delegation | A low-level hard-coded notification mechanism replaces "LLM chat pretending to notify" | 14ms startup, 4 Agents running 18 tasks in parallel, usable on an 8GB laptop |

**The commonality**: all four successful cases **abandoned free-form group chat** and switched to deterministic mechanisms — Anthropic uses persistent memory, PwC uses approval gates, Grid Dynamics uses Temporal workflows, and JCode uses file watching + structured messages.

### 5.1 JCode: Treating Agents as Microservices

JCode replaces "LLM chat pretending to notify" with file watching, a structured message bus, and master-slave delegation: when a file goes dirty, the low-level watcher hard-pushes it; messages travel over a strong-Schema channel; and subagent completion is called back by a state machine. These mechanisms are nothing novel — file watching is a cache dirty flag, a message bus is RPC + a message queue, and master-slave delegation is ordinary task dispatch. Its correctness comes precisely from "having nothing new": treating inter-Agent communication as distributed-systems engineering, without Prompt theater, lands right back on the hard constraints of §4.

---

## 6. Why "Role-Playing" Is a False Proposition

| Dimension | Multi-agent role-playing | Single agent, multiple rounds |
|------|--------------|-----------|
| Essence | Multiple calls to the same model, swapping the System Prompt each time | The same model reasoning in a longer Context |
| Token consumption | 10x (independent context per role) | 1x (shared context) |
| Information fidelity | Low (natural-language relaying, lossy) | High (direct reasoning within the Context) |
| Latency | High (serial multiple rounds) | Low (single inference) |
| Observability | Poor (cross-agent link tracing is difficult) | Good (a single Stack Trace) |

**Conclusion**: multi-agent under homogeneous models = simulating a single agent's multi-step reasoning at 10x the cost. Role-playing does not change model capability; it only adds coordination overhead.

---

## 7. Cross-References

- **[Agent Principles](agent-principles-en.md)**: detailed discussion of determinism vs non-determinism work boundaries, subtask delegation, and context isolation
- **[LLM Overthinking Critique](llm-overthinking-critique-en.md)**: the LLM's capability boundary (vision × speed), the correct division of labor for the Harness
- **[Qwen to MiMo Migration](qwen-to-mimo-migration-en.md)**: model-selection decisions, the iron rules for taming a thick Harness

**First principles**: single agent + deterministic toolchain + good Harness design > multi-agent + free-form group chat + role-playing. Complexity does not disappear; it only sinks downward — multi-agent shifts complexity from "writing code" to "managing agents", and the latter is harder.
