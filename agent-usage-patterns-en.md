# AI Agent Usage Patterns

> **Languages:** [English](agent-usage-patterns-en.md) (primary) · [中文](agent-usage-patterns.md)

*Human-side interaction techniques and review habits — what determines the quality ceiling of an Agent is not model capability, but how it is used.*

## Core Logic: The Quality Ceiling Is Determined by Feedback Precision

AI has no proactive "quality awareness." It will not spontaneously feel that "this reply is poorly written" or that "this operation could be better." It executes instructions and handles edge cases in the least-effort way. No feedback = no signal saying "this is wrong."

Failing to correct over the long term does not cause crashes or fatal errors, but it stops at the "works, but just a bit off" level. And that "bit" is exactly the gap that usage habits open up.

The essence of correction is not "telling the AI where it went wrong," but establishing a shared standard of judgment. The AI's memory mechanism solidifies preferences into persistent rules, so the same kind of scenario no longer needs repeated correction. This is the concrete operational path of compound interest — every correction is a calibration, and Hermes gradually shifts from a "generic default" into a "collaborator that understands you."

> For the full discussion of the compound-interest mechanism, see [Agent Compound Interest](agent-compound-interest-en.md).

## Interaction Patterns

### Drive Actively, Do Not Accept Passively

The essential difference between the two usage modes:

- **Passive use**: Accept whatever the AI does. Accumulate dissatisfaction without saying anything, and in the end conclude "this tool isn't good" — when in fact a feedback loop was never established.
- **Active driving**: Every interaction is a calibration. Check outputs frequently, immediately correct anything inconsistent and explain the reasoning, and ask for the motivation behind anything that exceeds expectations.

The second mode turns the AI from a "generic assistant that starts from zero every time" into a "collaborator that understands you more and more."

### Ask for the Reasoning Behind Actions

When the AI performs an operation beyond what you expected, simply ask "why did you do it this way."

This is a low-cost self-check mechanism: the AI is forced to scrutinize its own decision chain. If the motivation does not hold up, it can discover that itself — at far lower cost than finding the problem after the fact. You may also learn a technical reason you had not thought of ("X has problems in this scenario"); and even if the AI was ultimately just being lazy ("no particular reason"), you have still gained grounds for review.

### Ask "Why Not Use the Other Approach"

When an approach feels odd to you, ask "why not use X."

This is more effective than outright rejection — direct rejection makes the AI defend itself defensively, whereas a question makes it scrutinize proactively. Three possible outcomes: you learn the real reason (you win), you discover it was being lazy (you intercepted it), or you confirm the two approaches are equivalent (you saved yourself the time of agonizing).

## Instruction Techniques

### Negative Instructions Beat Positive Ones

"Do not auto-commit" works far better than "notify me before committing."

A positive instruction is a path, covering known scenarios. A negative instruction is a fence, covering all unforeseen paths. A single "do not" can intercept an entire category of problems, without needing to enumerate one by one "if you hit A, don't; if you hit B, don't; if you hit C, don't."

In engineering scenarios, "why not" constraints (forbidden technologies, patterns, approaches) determine output quality more than functional requirements do. Even a "global best practice" may be unacceptable in a specific environment — negative constraints force the AI to respect the local context and past failures.

> For the constraint framework of a production-grade SPEC, see the "five constraint perspectives" in [Agent Principles](agent-principles-en.md).

### Observe Before Acting, Correct Deviations Early

Before changing code, have the AI read the relevant files and explain the existing structure. Making changes with understanding rather than with assumptions lowers the error rate by an order of magnitude. Many people simply say "help me change X," and the AI skips the understanding phase and goes straight to cutting. Reading the files costs very little, but the erroneous modifications caused by skipping this step are very costly.

The other side of this: the AI's observation itself may be off. Once you find that the direction of understanding is wrong, correct it immediately — do not wait until it has finished executing. Correcting a deviation at the understanding stage costs the least; by the implementation stage the cost doubles, and once multiple files are interlinked the cost grows exponentially.

### Do One Thing at a Time

The context window is a limited resource. "Refactor this module and fix three bugs" is worse than fixing the bugs first and then refactoring. Parallel multi-tasking crowds out the context for one another, and the AI starts dropping earlier constraints — you said "do not change the public interface," and it forgot by the fourth task.

Every atomic task must have a clear verification standard — how to confirm it is done. A task without a verification standard = a task without an endpoint, and the AI will stop when things are "about right."

### If the Spec Turns Out Unreasonable During Execution, Backtrack Rather Than Work Around It

When a flaw is found in the original plan during execution, the correct move is to backtrack and fix the plan, not to patch things arbitrarily mid-execution. The cost of working around it is that all subsequent steps are built on a false assumption, and the debugging cost grows exponentially.

### List the Plan Before Modifying

"First say which files you will change and what you will change; wait for my confirmation before starting."

This step costs very little, but it intercepts a large number of overreaching actions like "I optimized it for you along the way." Plan review is the lowest-cost intervention available to a human — reading one line of a plan is far faster than reading a hundred lines of diff.

### Distinguish Generation from Modification

Generating new code and modifying existing code are completely different task types. Modification tasks have a higher error rate, because they require simultaneously understanding the existing context and making a correct change. Modification tasks call for stricter demands on planning and verification.

## Review Habits

### Read the Diff, Not the Whole File

After the AI finishes, read the diff — do not read the whole file from the start. The diff is the shortest expression of what the AI actually did — each changed line has a clear intent, making it an order of magnitude more efficient than reading the full text.

Version control systems naturally support this review mode. git log + diff is the lowest-cost quality gate.

### Check the Quality of Background Maintenance

An Agent's long-term value lies in its background accumulation (memory, skills, automation). The quality of this accumulation requires active review — whether memory writes are accurate, whether skills are outdated, whether cron runs as expected.

Passively accumulated background data gradually drifts. Checking and correcting it regularly is a necessary investment in keeping the direction of compounding.

### Corrections Must Be Written to Persistent Storage

A verbal correction only affects the current session. Only a correction written into memory or a skill has a compounding effect.

Operational path: find the problem → correct it and explain the reasoning → the AI writes it into memory → it takes effect permanently for the same kind of scenario. Skipping the third step = repeating the correction every time.

What to write is a matter of some subtlety. Three principles of memory strategy:

- **Store facts**: `User prefers X`, `Tool Y fails on Z`. Declarative sentences, objective descriptions.
- **Do not store instructions**: `Always do X` (the system prompt already contains behavioral guidelines; repeating them pollutes memory).
- **Do not store transient state**: session progress, PR numbers, intermediate file paths — these are retrieved via session_search and are not worth solidifying.

## Persistent Configuration

### AGENTS.md Locks Down Project Rules

Write the core constraints (code style, architectural principles, forbidden operations) into `AGENTS.md` at the project root. The AI Agent reads it automatically at startup, which is equivalent to pre-installing a set of preferences for every session.

This is more reliable than a verbal agreement — a verbal agreement depends on the AI remembering it (it will forget), whereas a file constraint is hard-coded (enforced every time).

### Project Documentation as Knowledge Accumulation

Maintain a document in the project that organizes the background knowledge the AI needs. This is far more efficient than repeating the explanation in conversation every time — the document is stateless, the AI loads it automatically at every startup, and you do not need to inject it manually.

Not all knowledge is worth accumulating. Mainstream knowledge ("how to install Docker," "SQL JOIN syntax") has already been digested by the AI from its training data. What is worth accumulating is **counter-mainstream knowledge**: the deep-water pitfalls of a particular tech stack, the real reasons behind internal architectural decisions, the approaches that were rejected, and why.

> For the moat effect of counter-mainstream knowledge, see the "AI-native knowledge strategy" in [Agent Principles](agent-principles-en.md).

## Techniques for Designing Technical Solutions

Interacting with AI to produce a technical solution is not "asking the AI for an answer" but "using the AI to stress-test." The techniques below are distilled from an NL-to-SQL architecture design case and apply to one-off solution design scenarios (complementing the daily usage habits above).

### 1. Give Constraints First, Then the Problem

Wrong approach: directly asking "help me design a solution for querying data with AI" — the AI will give a vague, general solution.

Right approach: first list all the hard constraints (technical, operational, organizational limitations), then describe the problem. The constraints greatly narrow the AI's reasoning space and rule out a large number of unrealistic directions.

### 2. Record "Why It Was Excluded," Not Just "Why It Was Chosen"

In a decision document, the reasons for exclusion matter more than the reasons for selection. When someone later questions "why not use X," the answer already exists, and the debate does not need to be repeated.

### 3. Make the AI a "Critic," Not a "Designer"

You already have an idea — have the AI find the holes. Give your analysis to the AI and say explicitly, "analyze and evaluate this logic, point out the problems." The AI's suggestions are not necessarily right (it lacks your domain knowledge), but its challenges can expose edge cases you had not thought of.

### 4. Replace a List of Conclusions with a Decision Chain

Do not give only conclusions; show the reasoning process:

```
Pure Parquet is not enough
  ├─ ACID for concurrency is needed → choose Delta or Iceberg
  ├─ Iceberg needs a Catalog → operational overhead too heavy
  └─ Conclusion → Delta Lake is the only reasonable choice
```

Readers can follow the reasoning process and judge whether they agree with each step. If some premise changes, the decision chain automatically becomes invalid and must be re-evaluated.

### 5. Iterate in Stages: Big Framework → Details → Engineering Pitfalls

- **Round one: big-framework alignment** — confirm whether the core direction is correct
- **Round two: detail supplementation** — add scenarios and constraints
- **Round three: engineering implementation** — discuss the pitfalls of actual implementation (retries, partitioning, schema-change strategy)

### 6. Tell the AI Explicitly What Does Not Matter

The AI tends to give a "complete solution" and will add advanced features that are not needed. Explicitly ruling them out saves discussion time and keeps the solution from bloating.

### 7. Keep a Complete Decision Document

The final output is not "a solution" but a complete document containing the decision chain, the reasons for exclusion, and the architecture diagram. When a new member joins, there is no need to re-explain "why it was designed this way."

### Quick-Reference Checklist

When discussing a technical solution with the AI, provide information in this order:

1. **Background** — what actually happened (do not describe it abstractly)
2. **Constraints** — all hard limits (technical, operational, organizational)
3. **Your thinking** — the parts you have already thought through
4. **Your doubts** — the parts you are unsure about
5. **Exclusions** — the things you explicitly do not want
6. **Iteration** — have the AI pick holes, you supply domain knowledge, loop 2-3 times
7. **Output** — a decision-chain document (including the reasons for exclusion)

## Cross-References

- **[Agent Compound Interest](agent-compound-interest-en.md)**: Usage patterns are the operational path of compound interest. Correction → memory → skill accumulation → cross-project knowledge transfer. Without proactive usage patterns, the compounding mechanism idles.
- **[Agent Principles](agent-principles-en.md)**: Design philosophy and capability boundaries. Usage patterns are built on an understanding of capability boundaries — only by knowing what the AI does poorly can you know what needs correcting.
- **[LLM Fundamentals](llm-fundamentals-en.md)**: The AI's three layers of limitation (high vision but low intelligence, egalitarianism, semi-mechanical nature) explain why specific usage patterns are needed to sidestep its weaknesses.
- **[Fractal](Fractal.md)**: The charter-level basis for usage patterns. Fractal defines how the AI should work (persona, cognitive protocol, workflow discipline), and this document defines how humans should use it. The two are complementary — Fractal's "Why Not" methodology, task atomization, and spec-backtracking discipline are the underlying logic of the instruction techniques and review habits in this document.
