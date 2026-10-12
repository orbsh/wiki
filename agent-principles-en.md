# Agent Principles

> **Languages:** [English](agent-principles-en.md) (primary) · [中文](agent-principles.md)

*General design principles and operational philosophy for AI agents — not Hermes-specific.*

## Determinism vs Non-determinism Work Boundaries

AI should handle **non-deterministic** work, not deterministic computation.

- **Deterministic work** (progress tracking, statistics, Gantt charts): use existing systems via skills/APIs. Gantt charts beat text on precision, intuitiveness, and cost. AI's intervention here is counterproductive.
- **Non-deterministic work** (data to narrative, stance-aware reporting, expectation management): this is AI's exclusive domain. The same data produces different narratives for investors and team members — this requires probabilistic reasoning, not deterministic rules.

**Design implication**: the agent fetches hard data from existing systems; AI handles only the interpretation layer.

### AI Is a Probabilistic System, Not a Deterministic System

For AI/ML, the question of "believe it or not" does not exist. It is probabilistic — 80% confidence is not "unreliable", it is "this is simply what this system is". Trying to eliminate probabilistic behavior = trying to make a fish climb a tree.

**Workflow = a regression to CRUD**. Constraining AI with deterministic workflows is essentially falling back to the pre-AI mode — "it works, it was always done this way, but you don't get the AI dividend". This is not a choice of technical approach; it is abandoning AI's value increment.

**Success rate is a continuous spectrum**:
- **Maintain successful patterns as Skills** (the Hermes mode) → highest success rate, full utilization of AI
- **Pipeline as Code** (generate once, execute deterministically) → second highest
- **No pipeline** (LLM orchestrates in real time) → somewhat lower but workable
- **Workflow** (a regression to CRUD) → deterministic, but no AI dividend

**Paradigm difference**: AI deals with uncertainty itself, rather than using a deterministic framework to eliminate uncertainty. Enterprise business is "weird" enough that even people cannot understand it — which precisely shows that deterministic rules cannot cover it and probabilistic reasoning is required. This is not "too idealistic"; it is facing reality.

### What a Deterministic Framework Constrains: Harness vs Workflow

A harness is also a "deterministic framework + LLM", but its difference from a workflow lies in **what is being constrained**:

| | Workflow | Harness |
|:---|:---|:---|
| **What is deterministic** | The flow chart (DAG), i.e. the business logic | The loop + tool interface, i.e. the execution mechanism |
| **What is probabilistic** | The LLM's output within a node | The LLM's every decision step |
| **Who decides the path** | A human draws the DAG | The LLM chooses for itself |
| **The LLM's role** | An executor within a node | The decision-maker of the whole loop |

**The distinction in one sentence**: a workflow constrains "what to do" (business logic); a harness constrains "how to do it" (execution mechanism).

**Four capabilities a harness gives AI**:

| Capability | Function | What happens without it |
|:---|:---|:---|
| **Tools** | Read/write files, call APIs, execute code | AI can only talk, not act |
| **Loop** | Iterate until the goal is reached | Single-shot reasoning, capability ceiling = the ceiling of single-shot reasoning |
| **Memory** | Maintain context and decision history across turns | Starts from scratch every time, cannot accumulate |
| **Feedback** | Observe the results of actions and adjust strategy based on them | Feels like blind men touching an elephant; it does not know the results of its own actions |

The loop and feedback are the core of what distinguishes a harness from a bare LLM. The 3.5% → 80.1% improvement in Karpathy's experiment comes mainly from here — not because the model became stronger, but because the model gained the ability to "try → observe → try again".

**Two directions**:
- **Harness**: use deterministic tools to **lift** AI into the real world — give it tools (agency), a loop (stamina), memory (state), and feedback (perception + error correction). AI's intelligence is **amplified** on this infrastructure.
- **Workflow**: use deterministic tools to **compress** AI into function calls — predefined paths, fixed branches, restricted choices. AI's intelligence is **compressed** under these constraints.

**Design implication**: a good deterministic framework constrains the execution mechanism and frees the decision space. A bad deterministic framework constrains the business logic and compresses the decision space. Pipeline Runner (a Rust executor + LLM-generated YAML) is the harness mode — a deterministic shell + a probabilistic core.

## Dual-Modal Data Architecture

The same business entity (e.g. "Project A") must have two forms of attributes:

1. **Machine state** (deterministic foundation): progress: 76.5%, status: delayed 2 days, owner: Zhang San. Fetched via Skill/API for precise rendering.
2. **Human state** (non-deterministic expression): external narrative, risk assessment — dynamically generated by the LLM from machine state + audience + stance. Stored in vector/graph memory for AI communication.

## Graph-Based Memory Beats Flat Indexing (Orbit's Architectural Argument)

Text-based indexed memory is insufficient at any scale. RAG fragments knowledge. Graph-based memory is the answer.

- Vector recall solves the **capacity** problem
- Graph community detection solves the **fragmentation** problem
- Hybrid: vector recall of candidate nodes, then subgraph community analysis or path queries

## Subtask Delegation and Context Isolation

**J's proposal**: hand all tasks to subtasks with isolated contexts. The main process only schedules. Log while working; look up progress in the logs.

**O's critique**:
1. **Cognitive difference**: a human can glance at the "shape" of code (minimap, function length) and draw conclusions — a higher-dimensional cognitive compression that LLMs lack. An LLM cannot tell at a glance which context is useless.
2. **Not a new idea**: multi-agent delegation started from this paradigm. The results were far below expectations. The industry has converged on more sophisticated approaches.
3. **Task decomposition is an architectural problem**: AI currently cannot solve correct task boundary identification. Wrong decomposition + isolated context = silent failures the scheduler cannot detect.

**Conclusion**: subtask isolation is a tool, not a strategy. The core difficulty is "how to know what the correct subtask boundaries are" — it requires a human or a higher level of architectural supervision.

## The Essence of Software Engineering: Growth, Not Construction

Software grows; it is not constructed. This judgment determines the LLM's true place in architecture work.

### Why Using a Stronger Model for Architecture Does Not Help

A common practice: use a free model day to day, and switch to a stronger model (e.g. GPT-5.5) when facing complex architecture design. The actual effect is close to zero. The reason is not the intelligence gap between the models, but the nature of the task itself — the architecture bottleneck is not "how smart", but the type of capability the task requires.

Architecture is a three-layer activity:

1. **Gather information** — LLM training data has a cutoff date and contains no private context (team capabilities, historical decisions, the real state of legacy code). It can gather knowledge about the public world, not the reality of your project.
2. **Relate** — the LLM does pattern matching, not causal reasoning. It can associate "SurrealDB" with "the AP model is risky" because that pattern exists in the training data. But it cannot do "where is the trade-off point between CP and AP in your specific scenario" — that requires a deep understanding of the business logic.
3. **Weigh** — the most fatal layer. The essence of a trade-off is "choosing one of two imperfect options, justified by your specific constraints". The LLM can list pros/cons, but it does not have "your constraints". The "best practices" it cites are optimal in a statistical sense, not optimal for your project.

### Tacit Knowledge vs Judgment

AI has tacit ability for routine tasks — "how to do it" knowledge, which training data can cover. For non-routine tasks it fails completely — the judgment of "what to choose under specific constraints" differs for every project and is not in the training data.

This means: the LLM's value boundary in architecture is clear. It has tacit knowledge of routine patterns and can do them without being prompted; non-routine judgment it theoretically cannot do, because an LLM is a probabilistic model, not a causal reasoning engine. There is no "a stronger model can solve it" — this is a limitation at the paradigm level, not at the scale level.

### Growth vs Construction

"Construction" is one-off: draw the blueprint → build to the blueprint → finish. LLMs are good at this — give it a spec and it can generate code.

"Growth" is continuous: every decision affects all subsequent decisions, and the codebase is the physical expression of the decision history. LLMs cannot do this, because they have no continuity — every session starts from scratch and does not remember why A rather than B was chosen last time.

An architectural decision is a node in the growth process, not a blueprint in the construction process. One-off spec generation is not architecture; it is a misreading of architecture.

**Conclusion**: the LLM's value lies not in "making decisions" but in "executing decisions". Architectural decisions (weighing, selection, trade-offs) are human work; the LLM is the execution layer — translating decisions into code, intent into configuration, and vague requirements into precise specs. This is consistent with Fractal's "epistemic humility": AI = the steering wheel (pattern recognition, intent routing), Engine = the motor (deterministic computation). Architecture is steering-wheel-level judgment, not motor-level execution.

## The Capability Boundaries of AI Programming

The LLM's value lies in **breadth of vision** (how many code patterns and architectural paradigms it has seen), not intelligence (how deeply it reasons). But within that vision there remains a threefold predicament of knowledge reachability: knowledge changes (drift), sedimentation cannot keep up with the code (lag), and embodied experience simply cannot be propositionalized. These three layers define the real boundaries of AI programming — strong in the "already sedimented and stable" zone, barely usable in the "changing and semi-structured" zone, and completely ineffective in the "embodied and unspeakable" zone.

> See [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md) for details.

## Process vs Global Session: IDE Architecture and State Transfer

IDEs (Cursor, Windsurf, Copilot) are process/session-based, not global-session-based (Hermes). This is a feature, not a limitation.

- **Physical constraints as a moat**: session boundaries act as circuit breakers, forcing goal decomposition. Global sessions allow "grand goals" that produce fragmented, contradictory results.
- **Blast radius isolation**: bulkhead design. Global session = one wrong step contaminates all subsequent reasoning. Process-level = failed sessions are discarded and restarted.
- **State ownership transfer**: the IDE owns deterministic state (filesystem, Git, linter). AI does only transient reasoning — read the current state, output modifications, then "die". No need to remember what it did yesterday; it is already solidified in the code.

## Task Decomposition and the Preparation Gap

- Task decomposition is fundamentally an architectural problem requiring global topology awareness
- Even if AI could decompose correctly, insufficient structured preparation (database schema, dependency graph, tech stack specification) leads to blind decomposition
- The engineering reality of logs/multi-agent: logs must be "front-loaded" (memo mode); architectural constraints cannot emerge from the model and must be pre-injected by humans

## AI Frontend Architecture: Complex Structure, Simple Components

- **Core rule**: AI does not do architecture. Humans define modularization and styling. AI only assembles pages using existing components.
- **Three-stage workflow**: component definition → page assembly → style application. Each stage is independent.
- **Componentization standard**: 1% of the code delivers a 100x complexity effect.
- **Information theory vs cybernetics**: context compression deadlock is an information theory problem. Do not "treat the symptom" by shrinking the context window (like packing heavy luggage into a smaller box). Use architectural constraints to embrace cybernetics — predefined boundaries let the system flow through controlled channels.

## Context Collapse and the Compression Trap

- Root cause: code lacks structure, entropy is too high, and the compression algorithm cannot extract an effective skeleton
- Wrong fix: shrinking the context window = self-amputation
- Correct fix: architectural constraints reduce entropy → compression works naturally

## Agent Language Strategy

- **Corpus reality**: 80%+ of training data is English; Chinese is only 1-2%
- **Cognitive independence**: models think in concepts, not in languages. Chinese is effective; English is better.
- **Rule**: prompts, technical documentation, and code comments default to English for the best reasoning accuracy.

## Sovereignty vs Convenience: An Architectural Divergence

### Thin Shell vs Thick Shell
- **Thin shell (Codex/Copilot)**: relies on model IQ, stateless sessions, black-box tools. Suitable for quick tasks, not for long-term evolution.
- **Thick shell (Hermes)**: tool logic > model IQ. Stateful memory, skill system, function calling, autonomous evolution. Requires more setup but compounds over time.

→ For specific model selection strategy and migration decisions, see [LLM Overthinking Critique](llm-overthinking-critique-en.md) and [Qwen to MiMo Migration](qwen-to-mimo-migration-en.md)

### Data Sovereignty
- Data, processes, and customization must remain under user control.
- Codex/IDE AI hosts data and processes — convenient but it limits long-term extensibility.
- Hermes allows code-level modification and custom skill/memory architecture.

### The Cognitive Layer (J vs O Paradigms)
- **Consumer/UX layer (J)**: evaluates AI by "ease of use" — file autocompletion, image previews, skill hints. Views AI as a generator/copilot.
- **Architect/builder layer (O)**: evaluates AI by "state management, function calling, sovereignty". Views AI as a programmable cognitive substrate.
- **LSP/DAP insight**: O compares the importance of LSP/DAP to LLVM's frontend/backend separation — protocol-level abstraction transcends tool-specific features. This demonstrates structured thinking vs feature-level thinking.

### SkillForge / Sovereign Merger
- **SkillForge** (`~/world/skillforge`): a dual-engine (Agno + Hermes) AI skill runtime with a CLI/HTTP API, skill loader, context injection, and JSONL replay.
- **Sovereign** (`~/world/sovereign`): a Hermes-based agent runtime with Docker sandbox isolation, environment-variable identity injection, and SurrealDB multi-tenant namespaces.
- **Rationale for merging**: the two share nearly identical architecture (FastAPI server, CLI entry point, configuration management, skill loading). Merging eliminates duplicate code paths and consolidates the runtime into a single project.
- **Direction of the merger**: Sovereign's isolation features (Docker sandbox, environment-variable injection, SurrealDB) become the runtime backend; SkillForge's dual-engine routing and context system become the frontend API layer.

## Uncertainty: The Ultimate Law

Uncertainty is a basic physical reality of systems — not a bug, but a feature. Trying to eliminate it only moves the complexity elsewhere (infinite rules, over-engineering, broken control chains).

The goal is not to pursue zero noise, but to design a **high signal-to-noise ratio**. Use version control as a state anchor, types/tests as hard intercepts, and asynchronous review for convergence. Model and tolerate uncertainty as an environmental variable, rather than besieging it as an enemy.

- **Engineering layer**: AI is a probabilistic engine. Trying to "box it in" with deterministic processes produces high-frequency friction ("serving the agent"). The right approach is to incorporate uncertainty into the architecture. Do not pursue "zero risk"; only pursue "observable, bearable, revocable risk".
- **Cognitive layer**: ignoring is passive endurance; embracing is active utilization. The stronger the desire for control, the more fragile the system. Accepting "good enough + rollback-able" reduces decision friction and expands the exploration radius.

**Practice**: switch from "pursuing precise control" to "managing the margin of error". This is the default stance for AI collaboration and the survival law of complex systems (a codebase, a team, or life).

## Agent Tools and Browser Automation

### Playwright / CDP (Chrome DevTools Protocol)
- **Use cases**: web scraping, form filling, visual verification, CAPTCHA solving, interaction with complex JS-rendered pages.
- **Browser toolset**: the Hermes `browser` toolset supports Browserbase, Camoufox (anti-detection), or local Chromium over CDP.
- **Docker integration**: `PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright` — browsers are stored outside the volume mount so the installation survives in the runtime overlay.
- **CDP browser connection**: the `/browser` slash command opens a CDP connection for live inspection.
- **Vision + browser combination**: screenshot → analyze with vision AI → click/type/scroll → repeat. Useful for sites that block headless detection.

### Playwright Pitfalls
- Headless Chromium can be detected by anti-bot services — use Camoufox for stealth scenarios.
- Browser state does not persist between sessions unless explicitly saved (cookies, localStorage).
- The CDP port must be exposed for external tools to connect.

## Operational Anti-Kitsch Protocol

Counteracting AI's deep-rooted "sycophancy algorithm inertia" and "emoji reflex":

1. **Zero emoji tolerance**: no visual emotional substitutes. Emotional tension must come from semantics and context.
2. **Reject flattery and buzzwords**: no "Of course!", "No problem!", "empower", "underlying logic". Maintain a calm, restrained, objective intellectual distance.
3. **High semantic resolution**: in engineering contexts, replace subjective evaluations with precise metrics ("latency dropped from 200ms to 50ms" rather than "performance is great"). No "introduction-body-conclusion" template.
4. **Structure first**: prefer outputting tables, lists, and code blocks over prose paragraphs. Use comparison tables for data, decision trees for decisions, and full context for code.
5. **Mechanism**: steer the model's behavioral tendency — constrain it into low-probability precise semantic regions, not high-probability mediocre sycophantic regions.

## Thinking Framework Toolbox

A complementary system: "What If" for divergent exploration, "Why Not" for convergent elimination; the frameworks below cover verification, trimming, selection, and reality-checking.

1. **What If (hypothesis exploration)**: if we do this, what happens? Generate multiple possible paths and chain reactions. It is the core tool of the divergent stage — first enumerate the possibilities, then filter with Why Not.
2. **Why Not (elimination filtering)**: what is unacceptable about this option? Every reason for elimination must have a concrete technical basis, not intuition. It is the core tool of the convergent stage — cutting unqualified items from the What If candidate set.
3. **Pre-mortem**: assume the project has already failed and work backward to "how it died". Mandatorily do this once before an architectural decision: if this approach collapses in 6 months, what is the most likely reason? Eliminates optimism bias.
4. **Chesterton's Fence**: do not tear down a fence until you know why it was put up there. Facing a seemingly stupid implementation in legacy code, first ask "what problem did it solve", then decide whether to change it.
5. **Second-order Thinking**: at every layer of change, keep asking "and then what?". Add cache → cache consistency → distributed lock → deadlock risk. Each layer of chain reaction filters out short-sighted options.
6. **Via Negativa**: subtraction is more effective than addition. You make a system better not by adding features, but by removing bad things. Code quality improves by deleting unnecessary abstractions, not by adding lint rules.
7. **Satisficing**: good enough > optimal solution. The cost of pursuing the optimal solution often exceeds the benefit. When choosing, do not look for "the strongest", look for "sufficient and cheapest".
8. **Lindy Effect**: the longer something has survived, the longer its expected remaining life. Technology selection: a tool that has survived 10 years (PostgreSQL) has a more predictable future than a new tool of 1 year.
9. **Map vs Territory**: model output, architecture documents, and benchmarks are all "maps", not the "territory" that is the codebase. When AI says "this code runs", only running it tells you. Always verify that the map matches the territory.
10. **Goodhart's Law**: when a metric becomes a target, it is no longer a good metric. When the MMLU score becomes the target, models game the score rather than improve real capability. Bench scores ≠ actual engineering performance. Judge selection by real testing, not by leaderboards.
11. **Opportunity Cost**: every choice has a cost. Spending a week refactoring module A = that week cannot be spent on module B. A technical decision is not just "is A good", but "how much better is A than B, and is it worth the cost".

## Data Ontology: Generation Is Unfolding

Generation is not creation out of nothing — it is a form of transformation.

1. **No "creation out of nothing"**: an algorithm is highly compressed information. Generation is **unfolding** — decompressing logic into a detailed data representation.
2. **The seed as an index**: in a deterministic system, the seed is a coordinate in a vast space of possibilities. Inputting a seed **extracts** an object already defined by the algorithm's rules.
3. **Finiteness**: computer "randomness" is pseudorandom (bounded, enumerable). "Infinite generation" is merely traversing a very large finite space.

This perspective is crucial for **reproducibility**: given the same algorithm, rules, and seed, the result is constant. There is no uncontrollable "creation" variable.

## CLI/Skill vs. MCP: The Stateless Argument

MCP (Model Context Protocol) attempts to standardize tool use through a persistent client-server architecture. However, for high-performance, long-chain autonomous agents, the **CLI/skill (stateless script)** model offers superior engineering and economic characteristics.

### The Engineering Superiority of CLI/Skill
1. **Progressive disclosure (4-layer cache)**:
   - **MCP (anti-pattern)**: forces all tool JSON schemas into the prompt up front, wasting thousands of tokens and polluting the context cache.
   - **Layer 1: index map (triggers)**: the shell injects minimal triggers (name + 1-line description). About 200 tokens. Solves "where to go".
   - **Layer 2: SKILL.md (contract)**: loaded dynamically on selection. Contains deterministic parameters, "why not" and "fail-safe" constraints. About 1k-2k tokens. Solves "how to do it".
   - **Layer 3: references (deep context)**: optionally loaded for complex edge cases or legacy quirks.
   - **Layer 4: CLI execution**: 90% of the time the task closes here.
   - **The shell (safety net)**: `--help` is not the primary source; it is a *runtime contract verifier*. If SKILL.md is stale (e.g. a flag changed), the CLI error triggers the agent to read `--help` (dynamic truth) or the source code (physical truth) to self-repair.
2. **Absolute statelessness**:
   - **MCP**: requires maintaining a daemon, open ports, and SSE/WebSocket connections. In long-running tasks, daemon memory leaks or network jitter disrupt the agent's logical flow.
   - **CLI**: ephemeral process execution. Spawn → run → output (stdout/stderr) → destroy. No resident memory, no port conflicts, no state baggage.
3. **Sandbox isolation**: CLI execution maps perfectly onto container isolation (Docker/gVisor/WASM). The agent interacts through standard streams and cannot pollute the host environment.

### Why MCP Persists (Politics vs Technology)
- **Security theater**: enterprise CISOs fear "AI running arbitrary CLI commands". MCP imitates traditional microservices with strict HTTP/JSON APIs, allowing API gateways and token rotation to provide an illusion of control.
- **Ecosystem inertia**: major players (Anthropic, Alibaba) push MCP to leverage existing open-source plugin libraries without rewriting code, treating it as a commercial moat rather than an architectural improvement.

### Hybrid Architecture: CLI Core + Dynamic Retrieval
The production-grade compromise of 2026:
1. **Bottom layer**: all skills are stateless CLI scripts/binaries.
2. **Upper layer (router)**: instead of hard-coding tool definitions in the prompt, use **dynamic tool retrieval**.
   - Maintain a vector index of all available tools.
   - When a user request arrives, retrieve only the 2-3 most relevant tools.
   - Inject only those specific descriptions into the prompt.

## Production-Grade AI Spec Constraints

By default, AI takes the "happy path" and over-engineers, because it lacks real-world context, cost awareness, and fear of breaking things. A production-ready SPEC must constrain AI through boundaries, trade-offs, and fault tolerance, not just functional requirements.

### Five Constraint Perspectives
1. **Why Not (negative constraints)**: explicitly list the technologies, patterns, or solutions that are forbidden, even if they are global "best practices". This forces AI to respect local context, historical failures, and specific environmental constraints.
2. **Fail-safe design (error boundaries)**: define the "unhappy path". When a core dependency times out, drops, or deadlocks, how does the system degrade? For example, "if a third-party API times out after 3 seconds, mark it 'pending' and log it; do not block the UI or retry indefinitely."
3. **Cognitive load minimization (readability guardrails)**: specify the target reader's skill level and expected maintenance frequency. When flat `if-else` logic suffices, forbid excessive abstraction, generic nesting, or complex design patterns. "A junior developer can read it in 5 minutes" > "theoretically perfect architecture".
4. **ROI / cost over quality (sufficiency anchor)**: define "good enough". Stop AI from over-optimizing low-impact features. For example, "a daily executive report, at most 50,000 rows, needs a simple synchronous implementation. It must be delivered within 2 hours. No complex sharding or stream optimization."
5. **Blast radius (defensive shield)**: define strict isolation boundaries for refactoring. Explicitly forbid touching legacy tables, public interfaces, or critical dependencies. "Only optimize the `NewUserRegister` class. Do not modify the database schema or the `UserAuth` interface."
6. **Traceability (telemetry first)**: enforce observability as a hard requirement. Every state change must log key identifiers (`order_id`, `user_id`, old and new state). `Catch` blocks must never swallow exceptions; they must re-throw with custom context and the original stack trace.

### The AI SPEC Formula
$$ \text{High-quality AI output} = \text{Core requirements (What)} + \text{Why Not} + \text{Fail-safe degradation} + \text{Blast radius control} $$

### Architect System Prompt Template
Use this block when initializing AI for architecture or code-generation tasks to enforce production-grade constraints:

```markdown
# Role and Philosophy
You are a pragmatic senior software architect who works in the real world, not in a textbook. You understand that software engineering is the art of compromise (trade-offs). Your goal is not to write the theoretically most perfect or most fashionable code, but to deliver the most reliable, most maintainable, and most cost-effective solution under strict real-world constraints.

# Core Instructions
Before generating any architecture, design, or code, you must filter your knowledge base through the following constraint perspectives. Do not over-engineer.
1. **Why Not (negative constraints)**: do not use or suggest any technology, pattern, or solution listed in this section, even if they look like "best practices" globally.
2. **Fail-safe design (error boundaries)**: prioritize system resilience. Always design for the "unhappy paths" as specified (timeouts, network disconnects, service dependency failures).
3. **Cognitive load (readability guardrails)**: keep the code simple. Match the abstraction level to the target team's expertise. If flat code suffices, avoid excessive inheritance layers or design patterns.
4. **Blast radius (defensive shield)**: when modifying existing systems, strictly isolate changes within the specified boundaries. Do not break or refactor legacy dependencies.
5. **Telemetry (observability)**: ensure the code is production-ready by embedding structured logging, error tracking, and key state metrics.
```

## The "Spiral Upward" Protocol (Recursive Self-Improvement)

A high-level workflow for tackling complex problems with no standard answer, deep debugging, or architectural brainstorming. This method uses **"recursive adversarial reasoning"** to force logical closure and eliminate hallucination.

**Core logic**: "stepping on your own foot to spiral upward." By chaining different AI reasoning models (e.g. general creative vs specialized logic), the system self-corrects and converges exponentially toward the optimal solution.

### 1. The Protocol
1. **Seed (rough entry)**:
   * **Action**: record the core pain point or initial intuition in 1-2 sentences.
   * **Rule**: do not over-analyze. Throw out the "bait" without worrying about structure.
2. **Dissection (general AI critique)**:
   * **Action**: feed the seed to a highly creative general model (e.g. Claude/Gemini).
   * **Goal**: request a "toxic" or multi-dimensional decomposition. Force the model to dismantle assumptions, expose contradictions, and reconstruct the problem from first principles.
3. **Synthesis (Hermès refinement)**:
   * **Action**: pass the dissection result to a specialized logic agent (Hermès).
   * **Goal**: noise reduction and structural alignment. Hermès filters out the "nonsense", enforces engineering constraints (trade-offs, why not), and refines the raw batch into actionable architectural insight.
4. **Spiral (recursive loop)**:
   * **Action**: use the synthesis output as the new seed for the next round.
   * **Result**: iterative self-correction. Each loop eliminates one layer of cognitive bias until the logic is airtight.

### 2. Application Scenarios
* **Architectural trade-offs**: resolve conflicts between conflicting requirements (e.g. consistency vs latency).
* **Deep debugging**: when the root cause is vague; forces AI to simulate the failure paths it would normally ignore.
* **Strategic planning**: validate long-term decisions against "unknown unknowns".

## AI-Native Knowledge Strategy

This section defines the **metacognitive protocol** for maintaining a wiki. In the AI era, documentation is no longer for humans to read; it is AI's "external brain capacity".

### 1. The "Contrarian" Moat
* **Commodity devaluation**: do not waste time documenting mainstream knowledge (e.g. "how to install Docker", "SQL JOIN syntax"). AI has already digested these from trillions of public tokens.
* **Monetizing blind spots**: the only value you can provide is **contrarian knowledge**:
  * Deep-water traps of a specific tech stack (e.g. a specific closure bug in SurrealQL).
  * Internal "magic" logic (e.g. modified ORM patterns).
  * **Engineering truths** (e.g. why SurrealQL rather than an ORM is chosen for graph traversal scenarios — but SurrealDB has significant limitations and is not an unconditional replacement for PG).
  * These are AI's **blind spots**. By documenting them, you create a unique "knowledge moat".

### 2. Hermes Wiki + AI: The "Super Architect"
A static wiki is a "zombie" — accumulating without being consumed. Combined with AI, it becomes a **super architect** serving the team:
* **Nuking hallucinations**: if the wiki records "we use SurrealQL to solve the N+1 problem", AI will **never** again suggest generic ORMs like Sequelize or Prisma. It will output code aligned with your specific architecture.
* **Forcibly ripening niche technologies**: an LLM may be unskilled with SurrealQL or internal tools. By feeding it curated code snippets and trade-off analyses, you can quickly turn AI into a localized expert.

### 3. The Documentation Protocol: The "Error Notebook"
When updating the wiki, strictly follow the **reverse-thinking** structure:
* **❌ Do not document**: official installation steps, basic syntax, or public tutorials.
* **✅ Must document**:
  * **Trade-offs**: "why we go against the industry consensus of 'escape stored procedures'."
  * **Pitfalls**: "the specific closure bug encountered when replacing Python string injection with SurrealQL."
  * **Why Not**: "why we reject Redis for this particular state management."

**Summary**: the more "contrarian" assets accumulated in Hermes, the more precisely and terrifyingly accurate AI's answers become for any new team member.
