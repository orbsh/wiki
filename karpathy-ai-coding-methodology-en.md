# Karpathy's AI Coding Methodology

> **Languages:** [English](karpathy-ai-coding-methodology-en.md) (primary) · [中文](karpathy-ai-coding-methodology.md)
> Created 2026-06-29

**Core view**: AI agents have no long-term memory, and code quality depends on the degree of structure humans provide. The only way to counter AI code bloat is minimalist discipline and a strict working loop.

## 1. Core Document-Driven

Karpathy stresses that AI needs extremely structured context. The project root must contain two Markdown files:

**SYSTEM.md (employee handbook)**: hard constraints for the AI. It covers tech stack choices (e.g. "use only native CSS, not Tailwind"), code style, directory structure, and red lines such as "do not refactor files that were not mentioned without being asked". It is essentially drawing the AI's working boundary — the AI works efficiently inside the boundary, and crossing it causes catastrophic bloat.

**TODO.md (dynamic roadmap)**: the AI's task board. Do not dump requirements all at once; instead break them down into 5-10 micro-steps. Each time a step is completed, the AI checks it off and updates the details of the next step. The real value of this document is not task management but **preventing the context window from being flooded by one-shot requirements** — the accuracy with which an LLM handles 5 clear small tasks is far higher than that for 1 vague large requirement.

**The nature of these documents**: these two files are not "documents"; they are the AI's runtime parameters. SYSTEM.md = constraint function, TODO.md = scheduling queue. Without them, the AI is an intern with no rules and regulations, starting from zero to understand the project in every conversation.

## 2. The Think → Execute → Verify Loop

Karpathy's standard interaction flow has three steps, none of which can be omitted:

**Step one: think first**. Say to the AI: "I want to implement feature X. For now, do not write code; list your modification plan, the files involved, and the potential risks in Chinese." The purpose of this step is not to obtain a perfect plan but to **expose the AI's misunderstandings** — if the plan is wrong, the execution will necessarily be wrong.

**Step two: precise execution**. After reviewing the plan and correcting it, issue the instruction: "Modify according to the plan, change only the relevant lines, and keep the rest of the code and comments absolutely untouched." The key constraint is "change only the relevant lines" — the AI's default behavior is to "optimize along the way" the surrounding code, which is the main source of code bloat.

**Step three: automated verification**. The AI must be made to write automated tests (Unit Test), or to run them through a script. Only when the tests pass is this micro-step considered finished. This step turns the AI's "I say I'm done" into "the system verifies it's done".

**The nature of the loop**: humans are responsible for judgment (Plan), the AI for execution (Code), and the system for verification (Test). The three roles are separated, and any one of them existing alone will fail. Human judgment only = deviations cannot be corrected; AI execution only = bloat cannot be controlled; system verification only = no one knows what should be verified.

## 3. Extreme Minimalism

In writing LLM.c (GPT training hand-written in pure C), Karpathy showed his aversion to modern engineering encapsulation. He recommends:

**Eliminate heavy framework dependencies**: if a feature can be solved with a few lines of native code, never introduce a complex third-party library or framework. The value of a framework lies in abstracting and simplifying complex scenarios, but if the scenario itself is not complex, the framework is pure noise — it increases the number of concepts the AI needs to understand and lowers generation accuracy.

**Keep code flat**: reject esoteric abstraction, matryoshka-style class inheritance, and complex architecture design. The more straightforward and plainly written the code, the higher the accuracy when the AI reads and modifies it. This is not "low-level" — it is a pragmatic adaptation to the AI's cognitive ability. The accuracy with which the AI handles 100 lines of flat code is far higher than that for 30 lines of deeply abstract code.

**Zero tolerance for dead code**: the AI easily generates obsolete code snippets. Once an uncalled function or variable is found, it must be deleted immediately and completely, keeping the context absolutely "clean". Dead code does not merely waste space — it is **noise**, and it will mislead the AI into referring to already-obsolete logic in subsequent modifications.

**The true meaning of minimalism**: it is not "writing less code" but "reducing the number of concepts the AI needs to understand". Every framework, every layer of abstraction, every unused function occupies the AI's limited attention budget. If the budget is spent on noise, it cannot be spent in the right place.

## 4. The Engineer's Own Line of Defense

Karpathy warns: if engineers do not write code for a long time, their "discernment" declines — you will lose the ability to judge whether AI code is good or bad. He recommends:

**Regular off-keyboard blind coding (Blind Coding)**: set aside a short period each week, completely turn off AI plugins, and hand-write basic data structures, algorithms, or low-level network connections from scratch. The purpose is not output but **maintaining sensitivity to code syntax and low-level logic**. It is like how pilots must fly manually on a regular basis — when autopilot is fine you don't need the skill, but when problems arise you must have it.

**Become a strict code auditor**: do not blindly trust the code the AI gives you. Treat every Code Review as an interview with a junior programmer, pressing the AI: "Why is this function used here?" "What are the complexity and edge conditions of this line of code?" The purpose of these questions is not only to find bugs but to **maintain the depth of your understanding of the code** — if you cannot ask them, it means your discernment is already degenerating.

**The discernment paradox**: the easier the AI is to use, the more easily engineers become lazy; the lazier engineers become, the less able they are to judge the quality of the AI's output; as quality judgment declines, the AI's errors become harder to discover. This is a negative feedback loop. Regularly "going off-keyboard" is the only way to break this loop — not because the AI is not good enough, but because human judgment needs active maintenance.

## Relation to Other Methodologies

Karpathy's methodology is not isolated. It complements the following concepts:

- **Agent compound interest** (agent-compound-interest): Karpathy's document-driven approach is the starting point of compounding — SYSTEM.md and TODO.md are the concrete form of persistence
- **Minimalist architecture**: Karpathy's code aesthetics are consistent with the minimalism the user pursues in projects such as Neovim/Fluxora — fewer concepts = fewer errors
- **Full-chain troubleshooting**: the Think → Execute → Verify loop is the micro-scale version of full-chain troubleshooting within a single interaction

---

## Cross-Reference Table

| Related Document | Point of Relation |
|:---|:---|
| agent-compound-interest | document-driven = the starting point of compounding |
| ai-programming-optimism-critique | Karpathy's pragmatism vs. the critique of optimism |
| neovim-0.12-ai-rules | the editor as an "audit platform" = Karpathy's code auditor role |
