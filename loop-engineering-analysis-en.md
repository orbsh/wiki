# Loop Engineering: An Analysis

> **Languages:** [English](loop-engineering-analysis-en.md) (primary) · [中文](loop-engineering-analysis.md)

**Status**: Analysis complete
**Date**: 2026-06-23
**Source**: A critical assessment of Addy Osmani's (2026-06-07) canonical essay

---

## 1. Original Definition

Loop Engineering was proposed by Addy Osmani (Google Chrome team) and promoted by Boris Cherny (the lead of Anthropic Claude Code) and Peter Steinberger. The core definition:

> Use the "system" you design to replace "you" in sending prompts to the Agent.

Five building blocks: Automations (cron/webhook), Worktrees (git worktree isolation), Skills (SKILL.md persistence), MCP (external connectors), Sub-agents (Maker/Checker separation) + Memory (state persistence).

Reference implementation: `cobusgreyling/loop-engineering` (723 stars); theoretical basis: `wquguru/harness-books` (2548 stars).

---

## 2. Essential Judgment

### Not a New Paradigm, but a Renaming of Existing Practice

All five building blocks of Loop Engineering **already have implementations**:

| Building Block | Hermes Implementation | Codex/Claude Code Implementation |
|--------|------------|----------------------|
| Automations | cron job, webhook | /loop, /goal, Automations tab |
| Worktrees | delegate_task isolated context | git worktree, --worktree |
| Skills | Hermes skills (SKILL.md) | Agent Skills (SKILL.md) |
| MCP | native MCP client | MCP servers |
| Sub-agents | delegate_task | Task subagents, agent teams |
| Memory | session DB + long-term memory | Markdown/Linear state files |

Like "Context Engineering" (a renaming of Prompt Engineering) and "Harness Engineering" (a renaming of the Agent framework), Loop Engineering is a 2026 branding event, not a technical one.

### Core Value: Define Once, Execute Repeatedly

The real value of Loop Engineering is **reducing human involvement**—in specific scenarios, define once and execute repeatedly, and the total amount of involvement does indeed drop. A typical case: a user files a ticket → AI classifies → AI directly develops the fix, and the customer service/triage steps in between disappear entirely.

But this benefit is **highly scenario-dependent**—it applies only to scenarios where the task source can be defined, execution can be verified, and results can be judged mechanically (CI fixes, dependency upgrades, simple bugs, automated ticket handling). Scenarios such as designing a new system or deciding on a technology choice are not applicable, because they lack an automatically verifiable termination condition.

---

## 3. Task Source: The Real Bottleneck of the Loop

### Tasks Do Not Appear Out of Thin Air

Whatever the input source of the loop is, a person must first define "what counts as a problem":

| Input Source | Human Involvement |
|--------|----------|
| Tell Hermes directly | Must speak up every time |
| Ticket system | A person must write tickets |
| K8s monitoring | A person must install monitoring and define alert rules |
| CI failure | A person must write tests and set thresholds |
| Git push event | A person must define what counts as a "problem" |

A loop cannot conjure tasks out of thin air, but in applicable scenarios it can indeed reduce human involvement (e.g., automated ticket handling eliminates the customer service triage step). The key distinction: the reduction in involvement occurs only in high-frequency, mechanical, verifiable scenarios.

### Event-Driven Beats Polling

Cron polling is the fallback when there is no webhook capability. Event-driven is preferred:

| Scenario | Polling (fallback) | Event-driven (preferred) |
|------|-------------|-----------------|
| K8s failure | cron checks pod status hourly | Watch API / Event push |
| New ticket | cron polls every 10 minutes | Webhook POST to Hermes |
| CI failure | cron checks build status | GitHub Actions webhook → Hermes |
| Code change | cron scans git log | Git hook / push event |

Hermes already has webhook capability (`hermes webhook subscribe`). If you can use events, don't poll.

### Specifying on the Side Cannot Solve the Task Source Either

Ticket systems (Linear, GitHub Issues) are a common side source of tasks, but they merely turn "you tell Hermes directly" into "you write a ticket, Hermes reads the ticket". Tasks still need a person to create and define them. What ticket systems solve is the **task management** problem, not the **task source** problem.

---

## 4. The Verification Ceiling: The Hard Boundary of the Loop

All a Loop can do is **verification** (pass/fail); it cannot do **judgment** (is it good, is it right, is it worth it).

| Automatically Verifiable | Not Automatically Verifiable |
|-----------|-------------|
| Tests pass / fail | Whether the architecture is reasonable |
| Lint clean / has warnings | Whether the business logic is correct |
| Type check passes | Whether the code style matches team conventions |
| Security vulnerability scan | Whether this is what the user actually wants |
| diff size / scope of change | Whether the technical approach is appropriate |

The four concepts are orthogonal and do not lie on the same dimension:

| Concept | Responsibility | Level |
|------|------|------|
| **Harness** | Gives the Agent the correct environment, tools, and context so it can work. Solves "can it be done" | Technical architecture (fixed solution, universal across scenarios) |
| **Inner Loop** | Self-verification within a single task (fix → test → fix → test → pass) | The Agent's capability |
| **Loop Engineering** | System-level scheduling: triggering, dispatching, advancing, cross-session state management. Solves "was it done right" | System orchestration capability (scenario-dependent, different for each business) |
| **Event-Driven** | Decides when to trigger, replacing cron polling. Solves "when to do it" | Trigger mechanism |

What Harness addresses is the mechanical fix itself (lint errors get lint fixed, once is enough); the inner loop is the Agent repeatedly verifying within a single task until it passes; Loop Engineering is system-level orchestration (triggering, dispatching, cross-session state); and event-driven decides when to trigger. The "explicit termination condition" emphasized in Loop Engineering documentation is the business of the inner loop, not a contribution of Loop Engineering itself.

Verification is binary; judgment is multidimensional. As soon as "should this feature be done this way" is involved, you must stop and wait for a person.

---

## 5. Practical Guide: Five Steps

What follows is the practical methodology that survived the critical analysis—not the branding narrative of Loop Engineering, but genuinely useful engineering practice.

### Step 1: The Four-Condition Test (in Consensus with the Karpathy/Niklaus Experiments)

Not every piece of work is suited to being stuffed into a loop. Before starting, pass four gates first:

| Condition | Meaning | Consequence of Not Meeting It |
|:---|:---|:---|
| **Does the task recur?** | Frequency determines ROI | A task that happens once a year is not worth designing a loop for |
| **Is there an automated acceptance method?** | Without verification there is no termination condition. Note: only binary verifiability (pass/fail) counts; things requiring multidimensional judgment (whether the architecture is reasonable, whether the business logic is correct) are out of scope (see Chapter 4, The Verification Ceiling) | The loop never knows whether it did the job right |
| **Can the Token budget hold up?** | Every run has a cost | Burning money costs more than hiring a person |
| **Does the Agent have the tools of a "senior engineer"?** | Can access the codebase, CI, and tests | Even a clever cook cannot make a meal without rice |

Only when all four gates are passed is it worth building a loop. Most impulses of "I want to automate this" will not pass these four gates.

These four conditions are consistent with the experimental conclusions of Karpathy (AutoResearch) and Niklaus (Hugging Face)—their experiments also proved that Loop automation has positive ROI only in scenarios that are high-frequency, verifiable, and have access to a real environment. See [LLM Overthinking Critique §7.8](llm-overthinking-critique-en.md#78-empirical-evidence-for-harness-first-karpathy--niklaus).

### Step 2: The Minimal Viable Loop

Don't get fancy the first time; just build the four-piece set:

1. **Trigger** (Automation): running on a schedule or on an event both work
2. **Skill**: write the project context into STATE.md, so each run doesn't have to re-explain
3. **State File**: Markdown records "how far you got, what succeeded, what broke", and the next run picks up from there
4. **Gate**: tests, type checks, builds—things that can automatically block bad results

**Order is key**: first run it through manually once → write it as a Skill → wrap it in a loop → only then put it on a schedule.

### Step 3: Setting the Exam vs Grading the Exam

The single most important principle in the entire loop design: **the one who writes the code and the one who verifies the code must be separate**.

Use one model (or sub-agent) to do the writing, and another, independent model (or sub-agent) to do the acceptance. The one doing acceptance **must not see** the reasoning process of the one writing the code.

Why? A model grading code it wrote itself uses "standards that are too low"—a direct manifestation of the lack of metacognition (see [LLM Fundamentals](llm-fundamentals-en.md) 5.3.5). Any code that "looks fine" will, in front of an independent checker, very likely have a large number of problems picked out of it. Maker/Checker separation is not an architectural preference; it is an engineering adaptation to the fact that LLMs have no metacognition.

### Step 4: Pitfall Guide

| Pitfall | Consequence | Prevention |
|:---|:---|:---|
| No hard stop condition | Runs until the bill explodes or you get rate-limited | Set a Token cap, an iteration cap, and a time limit |
| State isn't persisted | What you learned today is forgotten tomorrow | Write it into STATE.md; each run reads on from there |
| Letting the loop touch work that "requires judgment" | Errors in architecture rewrites, authentication, payment logic | The loop only does work that is "clearly right or wrong and machine-verifiable" |
| Not reading the Diff | Merges get faster and faster, understanding gets shallower and shallower | Read the Diff, even just a glance—otherwise you accrue "comprehension debt" |

"Comprehension debt" is the most insidious pitfall: the real cost is not the Token bill, but that one day you will have to debug a system no one on the team has ever read.

### Step 5: The Only Metric

Never mind how many Tokens were burned, how many PRs were opened, or how many times tasks were run.

**The only useful metric: the average cost per accepted change.**

If the "acceptance rate" is below 50%, it means you are doing review work that the loop should have saved you—the loop is losing money.

---

## 6. Supplementary Approach: AI Facilitator + Loop Execution

> Loop Engineering cannot solve communication problems—it can only automate mechanical tasks; it cannot replace the multidimensional judgment in requirements discussion. What follows is a supplementary architecture beyond the Loop: at the communication layer, use an AI facilitator to lower the cost of reaching consensus; at the execution layer, use the Loop to automate mechanical tasks; at the judgment layer, keep humans making the call.

### The Reality of Requirements Discussion

IM is the mainstream communication tool in Chinese workplaces, but its structural problem is that it synchronizes asynchronous communication—it requires everyone to be present in real time, and cascading interruptions fragment deep thinking. The root cause is hierarchy and a lack of respect for deliberation, not the tool itself (see [Workplace Communication](workplace-communication-en.md)).

The volume of communication **cannot be reduced**; it is a function of organizational complexity (10 decision-makers = 45 channels). The value of an AI facilitator is not to eliminate communication, but to make the unavoidable communication run more smoothly.

### Three-Layer Architecture

Trying to use a loop to eliminate communication is the wrong direction. The right direction is three layers, each minding its own business:

| Layer | Responsibility | Tools |
|:---|:---|:---|
| **AI Facilitator** | Organizes chaotic conversations, flags conflicts, syncs consensus anchors | IM channel + continuous AI analysis |
| **Loop Execution** | Code changes, tests, PRs, CI/CD | Gitea + CI + containers |
| **Human Judgment** | Calls the shots on requirements, reviews PRs, accepts results | Channel discussion + PR Review |

The value of an AI facilitator is not "reducing communication", but making the unavoidable 45 channels run more smoothly:
- **Reduce information asymmetry**: everyone sees the results the AI organizes, without relying on "who attended which meeting" to sync up
- **Reduce repetitive communication**: the consensus anchor is continuously updated, so there is no need for the same old back-and-forth
- **Flag disagreements**: when the AI finds a conflict of opinion, it pulls it out into the open, so there is no need for private haggling

### The Complete Flow of Requirements Discussion

```
IM channel (chaotic conversation)
  → AI continuously analyzes, organizes, flags conflicts
  → people discuss and confirm in the channel based on the AI's organized results
  → consensus reached → Gitea Issue
  → Loop: code → test → PR
  → human reviews PR → merge → release
```

Key details:
- **Extraction is not automated**: AI organizes + people review and correct in the channel; it is not the AI filing the Issue directly
- **Confirmation happens in the channel**: the person raising the requirement sees the AI's organized result at the same time (including conflicts with the goal/current path), and it is resolved through on-the-spot discussion, with higher transparency
- **The AI is a facilitator, not a decision-maker**: it continuously updates progress and consensus, and people discuss based on the new consensus anchor
- **The final judgment is made by a person**: the AI organizes and flags conflicts, but the call is a person saying "yes, that's it" or "no, change this"

### Solving the "Equal Treatment" Problem

An LLM treats all statements equally, but in an actual conversation different statements carry different weight:

| Solution | Applicable Scenario |
|:---|:---|
| **Time decay** | More recent statements carry higher weight (requirements are evolving) |
| **Consensus detection** | Several people said something similar → high confidence; only one person said it → low confidence |
| **Conflict detection** | A says "must be fast", B says "must be stable" → highlight in red for the relevant people to confirm |
| **Speaker rank ordering** | Decision-makers' statements carry higher weight |
| **Confirmation voting** | For key decisions, hold a vote among the relevant people to confirm |

### Risk Points

**Accuracy of the AI's organization**: if the AI misses key information or misreads the intent of the conversation, all subsequent discussion is based on the wrong organized result. But reviewing the AI's organized result is simpler than reviewing a code diff—each person only cares about "whether my own claim was conveyed correctly" and does not need to understand the global context. A diff, by contrast, is harder, because it requires understanding the complete logic before and after the change. Solution: add confidence annotations to the AI's organized result (which parts are clearly settled, and which parts the AI is unsure about and need human confirmation).

**Converting consensus to an Issue**: the consensus reached in the channel is colloquial ("let's go with Plan B then"). Converting it into an Issue requires structuring (goal, acceptance criteria, boundary conditions). The AI can do this step, but it needs human confirmation—"is the Plan B the AI understands the Plan B you meant".

**Comprehension debt**: if you skip communication and let the AI work on requirements directly, team members become "approval robots"—looking only at results without understanding the process. The real cost is not the Token bill, but that one day you debug a system no one understands. The AI facilitator model partially alleviates this problem by letting everyone see the AI's organization + the discussion process.

---

## 7. Scenario Analysis: The Applicable Scope of the Loop

### Two Kinds of Loops

Loop Engineering applies to two kinds of scenarios, with the same core logic (observe → classify → decide → execute) but different looping characteristics:

| | Maintenance Loop | Creative Loop |
|:---|:---|:---|
| **Goal** | Maintain system state | Get from A to B (development, writing) |
| **Number of loops** | Unlimited | Finite |
| **Pattern repetition** | High (the same kind of problem keeps recurring) | Low (every time is a new problem) |
| **Automation potential** | Grows over time | Fixed (the mechanical part is limited) |
| **Human role** | Classify + make the call | Plan + judge the direction |
| **Typical scenarios** | K8s inspection, inventory management, monitoring alerts | Requirements development, architecture design, creative work |

The core of the **Maintenance Loop** is known categories + predefined strategies. Take the K8s guard as an example:

| Category | Urgent | Dangerous | Action |
|:---|:---|:---|:---|
| Known (fix) | ✓ | ✗ | Auto-fix + log |
| Known (fix) | Other | - | Wait for human approval |
| Known (suggest) | ✓ | ✗ | Auto-suggest + log |
| Known (suggest) | Other | - | Wait for human approval |
| Known (report) | - | - | Report directly |
| Unknown | - | - | Wait for human classification |

As the category library grows, the automation ratio rises for categories that are urgent and low-risk. People step in to judge only when something is non-urgent or a new category appears.

The **Creative Loop** is Karpathy's "think → execute → verify" closed loop: it starts from a vague requirement and stops once it converges. The Loop can only take over the verification step (running tests, format checks); it cannot take over the thinking itself. Document-driven development and closed-loop verification lower the vagueness of the creative Loop so that the Loop can take over more of the mechanical part.

### Applicability to Operations Scenarios

Operations scenarios such as inventory management apply as well, but the thresholds are more vague:

- Low inventory → auto-restock (urgent + low-risk)
- Inventory pile-up → suggest a promotion (non-urgent + low-risk)
- Large purchase → report for approval (high-risk)
- New products / seasonal demand fluctuation → unknown category, wait for human judgment

The difference from technical scenarios: K8s thresholds are hard (3 pod restarts = abnormal), while inventory thresholds are soft (how much counts as "low" depends on the season, supplier cycles, and financial situation). In operations scenarios people need to step in more, but the mechanical part (inventory scanning, known-pattern matching, threshold checking) can still be automated.

### Key Insight

**Scenarios suited to a Loop share one common feature**: the judgment criteria can be defined and verification can be mechanized.

- K8s inspection: clear thresholds, known categories can be automated, people handle only new situations
- Inventory management: thresholds are vague but patterns are recognizable; the automation ratio is lower than K8s but still valuable
- Requirements development: judgment criteria are vague; the Loop can only take over verification, and people lead the whole process

**Scenarios not suited to a Loop**: no automatically verifiable termination condition (architecture design, product direction, creative work). In these scenarios human judgment is the core value, and the Loop cannot replace it.

---

## 8. The Current State of Hermes

Hermes already possesses all the components of Loop Engineering:

- **Automations**: cron job + webhook ✓
- **Worktrees**: delegate_task isolated context ✓
- **Skills**: SKILL.md + /curator lifecycle management ✓
- **MCP**: native MCP client ✓
- **Sub-agents**: delegate_task + kanban worker ✓
- **Memory**: session DB + long-term memory ✓
- **Kanban**: task decomposition + dispatcher + worker toolkit ✓

Hermes has all the components of Loop Engineering, but having the components ≠ having implemented scenario-based orchestration. Loop Engineering merely gives a new name to capabilities Hermes already had—the name is new, but the scenario-based connections are still missing.

### What's Missing Is Not the Components, but the Connections

Hermes's cron, kanban, and skills each run independently. A complete event-driven production line should be:

```
External event (ticket/CI/K8s/webhook)
  → webhook triggers Hermes
  → Hermes creates a kanban card
  → dispatcher spawns worker
  → worker executes, verifies
  → on completion, webhook writes back to the external system
  → experience written into a skill
```

This "glue" is not a new paradigm; it is a well-configured workflow.

---

## 9. The Progressive Trust Model (the Only Genuinely New Idea)

The L1→L2→L3 tiering proposed by Loop Engineering is worth recording:

| Level | Mode | Human Involvement |
|------|------|----------|
| L1 | Agent only generates reports, does not modify code | Review the report |
| L2 | Agent assists with fixes, requires human confirmation | Review the diff |
| L3 | Agent is fully autonomous, only notifies of results | Spot-check |

This is not a technical issue; it is an **organizational adoption strategy**—letting a team gradually transition from "completely distrusting the Agent" to "the Agent works autonomously". For enterprise environments (such as skillforge), this is a practically usable progressive deployment path.

---

## 10. Conclusion

1. Loop Engineering is a renaming of existing practice, not a new paradigm
2. Its core value is reducing human involvement, but it holds only in high-frequency, mechanical, verifiable scenarios
3. The task source always needs to be defined by a person; a loop cannot conjure tasks out of thin air
4. Event-driven (webhook) beats polling (cron)
5. Harness, the inner loop, Loop Engineering, and event-driven—the four are orthogonal and not on the same dimension
6. The verification ceiling: it can only verify mechanical results, not judge business reasonableness
7. Hermes already has all the components; what is missing is not capability but scenario-based connections
8. The only genuinely new idea is the progressive trust model (L1→L2→L3)

### The Naming Problem

The word "Loop" is vague and all-encompassing—code loop, Agent loop, business loop, learning loop can all be fitted into it. The "Loop" in "Loop Engineering" has insufficient limiting power: any repetitive engineering practice can be fitted into this term, and it does not carve out a subset distinct from "Engineering" itself.

Precisely because it is vague, everyone can fit their own existing practice into it and say "this is Loop Engineering". If a term can be claimed by all scenarios at once, it has no discriminating power. A term with no discriminating power is a marketing tool, not an engineering concept.

### Categorization Error

Addy's article places Loop Engineering as the fourth stage of technical evolution (Prompt → Context → Harness → Loop), and this categorization is wrong. Harness is technical architecture (a fixed solution, universal across scenarios); Loop Engineering is a business modeling methodology (scenario-dependent, different for each business). The two are not on the same dimension. Analogy: DDD (Domain-Driven Design) is a business modeling methodology, not the next rung of technical architecture.

### The Fundamental Difference from Harness

**Harness** is a fixed solution designed once and universal across all scenarios. **Loop Engineering** is a custom solution that needs to be designed separately for each scenario—CI fixing is one set of loops (trigger conditions, verification criteria, termination conditions), dependency upgrades another, ticket handling yet another. Harness is engineering invested once with unlimited returns; Loop Engineering is engineering invested N times with limited returns each time.

The main variable in cost recovery is **frequency**, but it is not the only variable—the cost of errors is equally crucial. A scenario that happens 100 times a day is worth designing a loop for; happening once a year is not; but if the cost of an error is extremely high (payment logic, security vulnerabilities), it is worth investing in automated protection even at low frequency.

### Methodological Depth

Loop Engineering as a methodology is itself too simple—seeing repetitive labor and thinking of automation is an engineer's instinctive reaction. "Design trigger conditions → define verification criteria → set termination conditions" is just the mindset of writing if-else; there is nothing to expand on. The real value of Addy's article is not in proposing the concept of "Loop Engineering", but in its systematic comparison of the shared primitives of Claude Code and Codex—that part is informative. But "Loop Engineering" as a concept itself is a correct yet mediocre engineering instinct packaged as a "paradigm".

---

## Cross-References

- **[Knowledge Base](projects/knowledge-base-en.md)**: skillforge memory system design
- **[LLM Fundamentals](llm-fundamentals-en.md)**: the lack of metacognition is the root cause of Maker/Checker separation (5.3.5)
- **[Karpathy AI Coding Methodology](karpathy-ai-coding-methodology-en.md)**: the quality closed loop of a single interaction, complementary to the multi-interaction closed loop of the Loop
- **[Workplace Communication](workplace-communication-en.md)**: IM vs email paradigms, AI facilitator architecture, communication volume and organizational complexity
- **[Flux Architecture](flux-architecture-en.md)**: the Loop's "event-driven beats polling" (webhook vs cron) is homologous with the core proposition of reactive architecture—passive pull vs active push, the same principle holding at both the infrastructure layer and the UI layer.