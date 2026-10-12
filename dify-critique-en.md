# Dify: The Harbor Copy of the AI Era

> **Languages:** [English](dify-critique-en.md) (primary) · [中文](dify-critique.md)

**Conclusion**: Reject.
**Status**: The AI version of Harbor — the same bloated architecture, a deeper paradigm contradiction.

## Executive Summary

Dify is the "Harbor" of the AI application field: a big name, a bloated architecture, difficult deployment, astonishing resource consumption. It tries to let non-technical people build AI applications by low-code drag-and-drop, but this makes two fundamental mistakes: first, it underestimates the essence of AI (autonomous planning vs. predefined flows); second, it overestimates the value of low-code (AI is low-code's natural enemy, not its ally).

## Affinity with Harbor

| Dimension | Harbor | Dify |
|:---|:---|:---|
| **Architecture** | PostgreSQL + Redis + Nginx + multiple containers | PostgreSQL + Redis + Nginx + multiple containers |
| **Footprint** | Storing a simple blob requires 10+ components | Building a simple chatbot requires 10+ components |
| **Deployment** | Helm chart nightmare, upgrades require database migration | Docker Compose nightmare, upgrades require database migration |
| **Resources** | Consumes several GB of memory when idle | Consumes several GB of memory when idle |
| **Fame** | CNCF graduated halo | GitHub star halo |
| **Audience** | Newcomers think it is "feature-complete" | Newcomers think it is "ready out of the box" |

**Essence**: Harbor is the over-engineering of "container image management"; Dify is the over-engineering of "AI application building". Different manifestations of the same problem — wrapping simple needs in complexity.

## Paradigm Contradiction: Low-Code vs AI, Natural Enemies

### The Core Paradox

Dify's value proposition is "build AI applications by dragging and dropping". But this proposition implies a premise: **the flow of an AI application can be designed in advance**.

If the flow can be designed in advance — dragging nodes, connecting lines, configuring parameters — then this application is essentially a **deterministic workflow**, and AI is merely one node inside it. This is not an "AI application"; this is an "automation pipeline with an AI node".

A true AI application should be: **given a goal, AI autonomously plans the execution path**. The user does not need to know what the flow looks like; AI itself decides which tools to call, in what order, and how to handle exceptions.

### Low-Code Is AI's Natural Enemy

| Dimension | Low-code thinking | AI thinking |
|:---|:---|:---|
| **Flow** | Designed in advance, fixed and unchanging | AI plans dynamically, self-adapting |
| **Exception handling** | Every branch must be wired manually | AI judges on its own, recovers automatically |
| **Extensibility** | Adding a feature = adding a node = rewiring | AI automatically discovers new tools and uses them |
| **User role** | Requires people who "understand the flow" to drag | Requires people who "understand the goal" to describe |

Low-code requires users to understand and design flows — which is exactly what AI should be doing. Making users do AI's work, and then handing users an "AI application platform" — this is putting the cart before the horse.

**AI is low-code's natural enemy**: if AI can plan autonomously, low-code's "visual orchestration" is worthless; if AI cannot plan autonomously, then it is not AI, just an ML algorithm. Dify chose the latter — it is essentially an **ML algorithm orchestration tool**, not an AI application platform.

## Experience Problems

### Code Editing: A Disaster

Dify's code editor is the inevitable product of the "low-code" philosophy — to accommodate "people who cannot write code", it provides a simplified editor. The result:

- **For people who cannot write code**: they still cannot write code, and dragging nodes does not let them understand how AI works
- **For people who can write code**: the editor is too crude, efficiency is far below a local IDE, and you would be better off just writing Python

This is the classic low-code trap: **useless to those who do not program, too inefficient for those who do**.

### Wiring and Layout: The Visualization Trap

The core selling point of a low-code platform is "visual drag-and-drop". But Dify cannot even deliver its own core selling point: the wiring algorithm only does geometric arrangement and does not understand semantics, with the result that **the visualization is equally poor**:

- **Connection edges cross chaotically after auto-layout** — for a slightly more complex flow (branches, loops, conditions), auto-layout produces crisscrossing lines that are 10 times harder to read than handwritten code
- **Manual adjustment is extremely costly** — aligning nodes and adjusting connection paths requires a great deal of mouse operation
- **Auto-layout overrides manual adjustments** — a layout carefully adjusted by hand is completely scrambled by a single auto-layout
- **Only suitable for linear flows** — real applications cannot be linear; for flows with branches, loops, and conditional judgments, visual arrangement is a disaster

**Essence**: whether visualization itself is a pseudo-need is another question, but Dify's wiring proves that it fails even at visualization — if even its core selling point is this poor, the rest goes without saying.

### Code Editing: A Cage That Cannot Be Resized

- **The edit box cannot be resized** — writing code in a tiny window; a single normal line needs three windows side by side to fit
- **Crude syntax highlighting** — not one tenth of VS Code's, and the rest of the editing experience can be imagined
- **Workflow**: copy it out cleverly → modify it → paste it back. This is not editing code, this is wrestling with the UI

### Version Control: Incompatible with Git

A low-code flow definition is stored in the database (JSON/YAML), not as code on the filesystem. The result:

- **Cannot be managed with Git** — flow changes have no diff, no history, no branch
- **Cannot be code-reviewed** — flow modifications can only be "looked at" in the Web UI, they cannot be reviewed like code
- **Cannot be rolled back** — there is no version history; if you change something wrongly, you can only change it back manually

**The irony**: Dify calls itself an "AI application development platform", yet it cannot even manage the most basic engineering practices (version control, code review, rollback) for itself.

### Deployment and Maintenance

- **Deployment**: requires PostgreSQL, Redis, Nginx, and multiple Worker containers; dozens of lines of Docker Compose configuration
- **Upgrades**: every version upgrade may require a database migration, and the vendor does not guarantee backward compatibility
- **Backup**: the state of multiple components is scattered across different places (database, filesystem, Redis), making the backup and recovery process complex
- **Resources**: consumes 2-4GB of memory when idle, which is absurd for a chatbot

**Comparison**: a Python script + SQLite + a local embedding model can implement Dify's core functionality in 50 lines of code, with resource usage < 100MB.

### The Illusion of Lowering the Barrier

Dify claims to "let non-technical people build AI applications too". But the reality is:

- **Non-technical people**: they do not understand prompt engineering, do not understand vector databases, do not understand embedding model selection. Dragging nodes does not fill these knowledge gaps.
- **Technical people**: writing code directly is faster, more flexible, and more controllable. Dify's drag-and-drop interface instead increases cognitive load — you must first understand Dify's abstractions, and then translate your requirements into Dify's nodes.

**The truth**: Dify did not lower the barrier; it merely swapped the barrier from "learn Python" to "learn Dify". And Python is a general skill, Dify is a specific tool — which is more worth learning goes without saying.

## Structural Reasons for the Fading of Interest

Dify received a great deal of attention in 2024, but that interest faded rapidly. This is not because Dify is badly made, but because the problem it solves (letting non-technical people orchestrate LLMs) is no longer needed in the Agent paradigm.

### The Paradigm Was Replaced by the Agent Model

Dify's core metaphor is the DAG — people predefine every branch, and the LLM acts as an executor inside the nodes. The evolution of LLM capabilities has a clear ordering: first it gains tool-calling ability (function calling), then it gains autonomous planning ability. Having tools but being unable to plan — knowing what it can call, but not when to call it — Dify has people make this decision on the LLM's behalf. Once an LLM gains autonomous planning ability, the predefined DAG instead becomes a negative optimization that constrains AI's generative space: a person spending 3 hours drawing a flow chart is, in essence, making the "where to go next" decision on the LLM's behalf.

### Low Barrier = Low Differentiation

Dify, Flowise, LangFlow, Coze are essentially the same thing: front-end drag-and-drop → JSON DAG → back-end chaining of LLM API calls. The visual editor is a superfluous middleman — once you understand the principle, it turns from "lowering the barrier" into "increasing cognitive load".

### The Ceiling Arrives Too Fast

Simple scenarios (single-turn RAG, simple classification) work. Slightly more complex ones (multi-step reasoning, conditional branching, state management) → the drag-and-drop editor collapses, and you find that you need to write code. And once you need to write code, Dify's visualization layer becomes a superfluous middleman.

### The Cost Structure Does Not Support It

Every node invocation = one independent API request. A three-step workflow = three LLM calls = triple the token cost + triple the latency. An Agent completes multi-step decisions within a single inference, sharing context, with better cost and latency.

### Real Enterprise Needs Are Not Here

What enterprises want is not "let business people draw flow charts", but: embedding custom logic into existing systems, permissions/auditing/observability, and deep integration with existing services. These needs are written with an SDK + code an order of magnitude faster than with a visual editor.

**Analogy**: Dify-style tools are to AI Agents what Scratch is to programming. It lowers the barrier to entry, but once you understand the principle, it becomes something that limits you. See [The Myth of Visualization](visual-myth-en.md).

### Multi-Skill Collaboration: The Orchestration Problem

The alternative to a Dify workflow is not a "better workflow", but **the layered solution space of multi-Skill collaboration**.

The core problem facing AI applications today is **multi-Skill collaboration** — a business process requires multiple Skills to cooperate in sequence/dependency relationships, involving intent routing, data passing, state synchronization, and error rollback along the way. Each of the four layers has flaws:

| Solution | Orchestration method | Advantages | Flaws |
|:---|:---|:---|:---|
| **Dify workflow** | Human-predefined DAG | Visual, strongly deterministic | Rigid, cannot handle unforeseen branches |
| **Skill (Agent autonomous orchestration)** | LLM decides which Skill to call and in what order | Flexible, self-adapting | The orchestration process is not auditable; if one Skill's data is wrong it cascades contamination, and errors are invisible |
| **AI writes code directly** | No orchestration, code is the logic | Most efficient, fully controllable | Requires the LLM to have sufficient capability to generate correct code |

**Three-layer problem**:

1. **Intent routing**: a user request comes in, and the system decides which Skill to call. This is the most surface-level problem — when routing is needed, Skill is currently the most efficient form, and workflow is worse.

2. **Data flow**: the output of the previous Skill is the input of the next one. The core pain point of multi-Skill collaboration lies here — if a Skill returns the wrong format, with missing fields or the wrong type, the downstream Skill fails silently or produces erroneous data, and **this is invisible**. This is what W described as "if a Skill is not constrained well, data errors will appear and be invisible".

3. **Observability**: the orchestration process is a black box. When an error occurs, you do not know which step caused the problem, at which step the data was contaminated, or whether the error has already cascaded and spread. This is homologous with the "auditability" flaw of visual tools — GUI operations are not auditable, and Skill orchestration decisions are equally not auditable.

**Key judgments**:

1. **When orchestration is not needed, AI writing code is a dimensional strike against workflows** — if the logic is deterministic (data transformation, format processing), generating code directly is an order of magnitude faster than drawing a flow chart.

2. **Routing cannot be eliminated** — this is the most significant value increment of AI applications. CRUD functionality already existed; and having AI look up a to-do item is much slower than before. AI's real value lies in **understanding intent, choosing a path, and combining capabilities**.

3. **A complete AI application architecture needs to solve orchestration observability**, rather than retreating to the rigid determinism of workflows. Skill flexibility + an auditable orchestration process is the balance point the current architecture needs to find.

## The Common Root with Harbor

The bloat of Harbor and Dify is not accidental; they share the same anti-pattern:

1. **The "feature-complete" illusion**: build in every feature that might ever be needed, leading to architectural bloat. Harbor built in vulnerability scanning, image replication, and a Web UI; Dify built in workflow orchestration, model management, and knowledge base management.
2. **"Enterprise-grade" marketing**: using "enterprise-grade" as an excuse for complexity. Harbor says "enterprises need RBAC and auditing"; Dify says "enterprises need visual orchestration and permission management".
3. **Dependency hell**: to achieve "feature completeness", a large number of external dependencies are introduced. Harbor needs PostgreSQL + Redis + Trivy; Dify needs PostgreSQL + Redis + Celery + Nginx.
4. **Fame masks the problems**: the GitHub star / CNCF graduated halo makes newcomers overlook architectural flaws. Harbor's star count makes people overlook its bloat; Dify's star count makes people overlook its paradigm contradiction.

## Failure at the Paradigm Level

Dify's real problem is not architectural bloat, but that **it represents a mode of thinking — constraining a probabilistic system with a deterministic framework**.

AI is a probabilistic system; 80% confidence is not "unreliable", it is "the nature of this system". Trying to eliminate probabilism = trying to make a fish climb a tree. Dify's workflow replaces the LLM's probabilistic decisions with a human-predefined deterministic DAG — this is not "augmenting AI", it is "bypassing AI".

### Harness vs Workflow: Two Directions

| | Workflow (as represented by Dify) | Harness |
|:---|:---|:---|
| **What is deterministic** | The flow chart (DAG), i.e. the business logic | The loop + tool interface, i.e. the execution mechanism |
| **What is probabilistic** | The LLM's output within a node | The LLM's every decision |
| **Who decides the path** | People draw the DAG | The LLM chooses on its own |
| **The LLM's role** | An executor inside a node | The decision-maker of the whole loop |

**The four capabilities Harness gives AI**: tools (agency to act), loops (stamina), memory (state), feedback (perception + error correction). On this infrastructure, AI's intelligence is **amplified**. Workflow cuts all of these away, leaving only an "AI node" executing predefined actions — under these constraints, AI's intelligence is **compressed**.

**Workflow = the regression of CRUD**. "It works, this is how it was always done" — but it cannot capture AI's dividends. AI's value increment lies precisely in uncertainty: understanding vague intent, handling non-standard data, choosing among multiple possibilities. Workflow cuts all of these away.

**Enterprise business is so "weird" that even people cannot understand it — which precisely shows that deterministic rules cannot cover it, and probabilistic reasoning must be used**. Using a workflow to handle "weird business" is no different in essence from using Excel to handle "weird business" — both have people making decisions on the system's behalf.

> See [Agent Principles](agent-principles-en.md)

## Conclusion

Dify is the Harbor of the AI application field: wrapping simple needs in complexity, masking architectural flaws with fame, and using "low-code" as an excuse for lack of ambition.

**The fundamental mistake it makes**: trying to make AI adapt to human flows (drag-and-drop orchestration), rather than making people adapt to AI's capabilities (autonomous planning). This is not a choice of technical route, it is a misunderstanding of AI's essence. Workflow constrains the business logic (what to do), Harness constrains the execution mechanism (how to do it) — the former compresses the decision space, the latter releases it. See [Agent Principles](agent-principles-en.md).

**Preferred alternative**: write code directly. Python + PostgreSQL (JSONB + a local embedding model), 50 lines of code, 100MB of resources, fully controllable. If graph traversal or multi-model capability is needed, SurrealDB is a supplementary option — see [Unified Data Layer](unified-data-layer-en.md).

---

## Cross-References

This document forms a complete decision loop with the following architecture analyses:

- **[Harbor Critique](harbor-critique-en.md)**: Harbor and Dify share the same bloated architecture pattern — multiple containers, multiple dependencies, multiple components, wrapping simple needs in complexity.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: Dify relies on Redis for caching/queues, homologous with Harbor's Redis dependency — the "network RAM trap".
- **[Nginx Critique](nginx-critique-en.md)**: Dify uses Nginx as a reverse proxy, homologous with Harbor's Nginx dependency — the unnecessary introduction of a legacy component.
- **[Unified Data Layer](unified-data-layer-en.md)**: SurrealDB's multi-model architecture could simplify Dify's patchwork of PostgreSQL + Redis + vector database — but PostgreSQL's JSONB already covers most schema-free scenarios, and SurrealDB's positioning is a supplement for critical data scenarios, not an unconditional replacement.

**The unified first principle**: no tech worship, no swallowing the big promises open source paints, look only at the real physical limits of hardware and at team productivity.
