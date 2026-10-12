# LLM Fundamentals: Cognition and Mental Models

> **Languages:** [English](llm-fundamentals-en.md) (primary) · [中文](llm-fundamentals.md)
> The correct mental model for understanding LLM: it is not a "language-understanding agent," but an "engine that matches statistical patterns." All prompt techniques, capability boundaries, and usage strategies are built on this foundation.

---

## 1. What LLM Is

### 1.1 Essence: A Next-Token Probability Predictor

The underlying mechanism of an autoregressive LLM (GPT, Claude, MiMo, etc.) is extremely simple:

```
Input: the first N tokens
Output: the probability distribution of the (N+1)-th token
```

It does not "understand" what you are saying, does not "think" about what the answer is, and has no internal working memory. It is only doing one thing: **based on the preceding text, predict the next most likely token**.

```
You: The weather today
LLM: (computes the probability distribution) → "is" (0.35) "was" (0.28) "very" (0.15) ...
You: The weather today is very
LLM: (computes the probability distribution) → "good" (0.62) "hot" (0.18) ...
```

"Thinking," "reasoning," "understanding" — these are all emergences of the probability chain in a statistical sense, not independent cognitive processes.

### 1.2 No Internal State

An LLM has **no memory whatsoever** between two inferences. Every call starts from zero — input the complete context window, output the next token. The so-called "remembering what was said before" is just stuffing the previous conversation into the context window so that the model can "see" it at the next prediction.

```
Call 1: input [system prompt] → output "hello"
Call 2: input [system prompt + hello + user message] → output "answer"
         ↑ there is no state transition between the two calls
```

This is why the context window is the upper bound of an LLM's working memory — everything outside the window does not exist for the LLM.

### 1.3 The Fragility of the Probability Chain

If the first prediction is wrong, all subsequent predictions are based on the wrong preceding text, the probability chain breaks, and the output is content that "looks fluent but is completely wrong" — that is, a hallucination.

The role of the `<think>` tag is to give the model a "draft notebook": before spitting out the formal answer, it can first make mistakes internally, cross them out, and start over. This is not "deep thinking," but **trading extra token consumption for self-correction of the probability chain**.

### 1.4 The Computation Path: High-Dimensional Matching, Not Linear Reasoning

An LLM's "thinking" and human thinking are **essentially different modes of computation**. Humans reason serially — thinking step by step, each step depending on the conclusion of the previous one. An LLM does high-dimensional parallel matching — input tokens are encoded into high-dimensional vectors, attention layers directly match patterns from the training data in the high-dimensional space, and the output is the probability distribution of the next token. The entire "reasoning" is already complete at the attention layers; every output token is merely a **linear projection** of the high-dimensional matching result.

```
Actual computation path: input tokens → embedding → attention layers (high-dimensional parallel matching) → logits → next token
Perceived computation path: "let me think…" → "from three angles…" → "synthesizing the above analysis…" → conclusion
```

The perceived path is a hallucination — the model is **playing the role of a human who is thinking**, not actually performing linear reasoning internally. Writing "let me analyze this" inside `<think>` is not the model "thinking," it is the model imitating the pattern of humans writing analytical articles in the training data.

### 1.5 The Essence of the Draft Notebook: Intermediate Seed

The `<think>` tokens are not a record of the reasoning process, they are **intermediate context that changes the probability distribution of subsequent generation**. Compare a linear congruential generator:

```
seed → f(seed) → next_value → f(next_value) → ...
```

The output of `<think>` is exactly `f(seed)` — an intermediate state used to bias the direction of subsequent output. Its value lies in **providing the correct bias**, not in its length. 10 tokens of good bias > 1000 tokens of garbage bias.

MiMo's narrow chain of thought is exactly this pattern: the fewest intermediate seeds → the most precise bias → direct output. The quality of the seed determines the quality of the output; the length of the seed is merely noise.

### 1.6 The Tree Structure of the KV Cache

The KV cache is not a linear chain, it is a **tree**. Each conversation branch inherits the prefix cache from its parent node and continues writing independently.

```
[system + conversation history]
         │
    ┌────┴────┐
    ▼         ▼
[thread_A]  [thread_B]    ← KV cache shared with the parent node
    │
    ▼
[thread_A_2]              ← continues independently, does not share thread_B's cache
```

**Three branching modes**:

| Mode | Branch Point | Lifecycle | Cache Reuse |
|:--|:--|:--|:--|
| **Thread** | Any historical message (user-chosen) | Cross-turn, persistent | Parent message cache + each thread independent |
| **Tail prompt** | End of the prompt (system-injected) | One turn, discarded | Prefix cache + new computation at the tail |
| **Model switch** | No branching, physical isolation | Not applicable | 100% invalidated |

A thread is a user-driven persistent branch; a tail prompt is a system-driven temporary branch. Both exploit the prefix cache; the difference lies in lifecycle and in who controls them.

> **Practical extension**: the following usage techniques are based on the tree structure of the KV cache.

**Proactively opening threads**: in IMs that support threads, open a thread for each different topic. Each thread's prefix cache is reused independently, avoiding the main thread's prefix being polluted by the alternation of different topics. Platforms where Hermes supports threads: **Feishu** (group topics), **Discord** (threads), **Telegram** (group topics). WhatsApp and Signal have no thread support and cannot exploit this mechanism.

**Cache reclamation for branches**: after thread_A has gone far and then returns to thread_B, the cache of the common prefix (the part before the branch point) is still usable — provided it is still inside the TTL window (usually 5-30 minutes). If thread_A takes too long or the server is under heavy VRAM pressure, the common prefix may be evicted by LRU, and returning to thread_B requires recomputation. In practice, if you switch branches within 5-10 minutes, the cache is basically still there.

**Background-knowledge priming pattern**: exploiting this property, you can first have the LLM read through the project's code/docs, loading the background knowledge into the KV cache, and then open threads on that basis to do concrete tasks. Each thread inherits the already-cached project context and only needs to compute the tokens of its own task.

```
Main thread: "read through this codebase" → the LLM reads the code, generates a large number of tokens → cache established
  ├── thread_A: "fix this bug" → inherits the project context + bug description
  ├── thread_B: "add a feature" → inherits the project context + requirement description
  └── thread_C: "write tests" → inherits the project context + test scope
```

Each thread's cost ≈ the tokens of the task description (the project context goes through the cache). Suitable for scenarios that require repeatedly operating on the same project.

→ For factors that break the cache, see [LLM Caching Destruction Patterns](llm-caching-destruction-patterns-en.md)

**Application scenarios of the cache vine**: the tail prompt is the leaf of the cache vine — injected temporarily, discarded when used up. The criterion: **you need to process existing history + you want to exploit the cache**. Bypass sedimentation oriented at external-memory state (compression, preference learning, profile update) is launched from the immutable prefix as a **parallel side branch**, not mixed into the answering turn (for the branch-legitimacy criteria see [LLM Caching Destruction Patterns](llm-caching-destruction-patterns-en.md) § Past-facing vs Future-facing).

| Scenario | Trigger Timing | Tail Prompt Content | Result |
|:--|:--|:--|:--|
| **Compression** | Message count reaches a threshold (e.g. 100) | "Analyze the above conversation, extract memories, mark a checkpoint" | Old history is replaced by the checkpoint and discarded outright |
| **Cache renewal** | Session idle + cache about to expire | "Refresh the checkpoint" | The checkpoint is updated, the cache is hit again |
| **Preference learning** | Periodically (e.g. every 20 turns) | "Extract user preferences and update them" | The `memories` table is updated |
| **Summarization** | After a discussion ends | "Summarize the key points of the above discussion" | A summary is generated and injected into subsequent context |
| **Compliance review** | Before a critical operation | "Check whether the above conversation conforms to the spec" | An audit log; history is not modified |
| **Meeting minutes** | At the end of a meeting | "Generate structured meeting minutes" | Markdown output, written to a document |

The compression scenario is a special case: once the tool call triggered by the tail prompt has completed, the preceding history is discarded outright (replaced by the checkpoint). The other scenarios keep the history.

→ For details, see [Tail Prompt Optimization](tail-prompt-optimization-en.md)

---

## 2. What LLM Is Not

### 2.1 Not a Logical Deduction Engine

LLMs are good at pattern matching and lack genuine multi-dimensional logical deduction ability.

| Dimension | LLMs Are Good At | LLMs Are Not Good At |
|------|---------|-----------|
| Pattern recognition | Recognizing common patterns from large amounts of code | Deriving a theorem never seen before |
| Breadth stitching | Cross-file consistency operations, refactoring | Multi-dimensional trade-offs (performance vs consistency vs complexity) |
| Mechanical transformation | Bulk replacement, formatting | Solving new problems (no ready-made pattern to match) |
| Fast response | Giving a solution in seconds | Deep derivation (long-chain logical deduction) |

**Analogy**: an LLM is an encyclopedia, not a mathematician. An encyclopedia "knows" a large number of patterns, but it will not "prove" a new theorem.

### 2.2 Not a Knowledge Base

An LLM's "knowledge" is a statistical compression of the training data, not structured storage. It **has no database** and no ability to "look it up" — its "knowledge" is scattered across the weights of billions of parameters, impossible to retrieve precisely, only to recall approximately.

This means:
- It may "remember" that an API exists, but the parameter names are made up
- It may "know" how a framework is used, but the version is from before the training cutoff date
- It knows nothing about private business rules — they are not in the training data

### 2.3 Not a Decision Maker

An LLM can generate plans, but it cannot judge their quality. It has no internal "right or wrong" check mechanism — it only knows "which token is statistically most likely to appear here," not "whether this plan is feasible in engineering terms."

Judging quality requires **means of verification independent of the LLM**: compilers, tests, human domain intuition. Without these, whatever the LLM outputs is something you can only accept — at that point the subject-object relationship flips: it is not you using the AI, it is the AI using you.

---

## 3. LLM's Value Formula

```
LLM's value = vision × speed
            ≠ intellect
```

### 3.1 Vision (Vision)

An LLM's "vision" is the patterns, architectural paradigms, and pitfall experiences it has seen in the training data. Medium-sized models (7B-70B) are already sufficient in vision — the code patterns they have seen cover the vast majority of everyday engineering scenarios.

**The boundaries of vision**:
- Covers public, stable, already-sedimented patterns
- Does not cover private business rules (in drift)
- Does not cover documentation (already rotted)
- Does not cover embodied experience (cannot be propositionalized)

### 3.2 Speed (Speed)

The core legitimate value of an LLM is **execution speed** — generating code, plans, and analyses in seconds. Humans provide judgment, AI provides speed.

If a human cannot judge the quality of the AI's output (has no judgment anchor), the subject-object relationship flips:

```
Normal: human judges → AI executes → human verifies
Inverted: human inputs → AI outputs → human cannot judge (AI uses you)
```

### 3.3 Intellect Is Not Value

Deep-thinking models (o1, R1, etc.) try to use the LLM's weakest ability (deep reasoning) to replace the LLM's strongest ability (breadth pattern matching). This is like making an encyclopedia prove Goldbach's conjecture — its value is in "knowing," not in "proving."

---

## 4. The Behavioral Mechanism of LLMs: High-Dimensional Keyword Matching

### 4.1 Appearance and Essence

An LLM appears to go far beyond keyword recognition — it is natural, flexible, and can handle complex semantics. This appearance creates a reverse misunderstanding: "using keywords is too low; what an LLM understands is semantics."

But the fact is: **LLMs learned a high-dimensional version of "keyword recognition" from massive corpora**.

```
Human view: summarize ≈ consolidate (semantically close)
LLM view: summarize → [the summary corpus distribution], consolidate → [the report corpus distribution]
          ↑ two completely different activation paths
```

### 4.2 Why Changing a Word Is More Effective Than Adding Constraints

You are not talking to an "agent that understands semantics," you are talking to an "engine that matches statistical patterns."

- **Adding constraints**: trying to use natural language to override the LLM's default statistical pattern (fighting the model)
- **Changing a word**: directly choosing the word whose statistical pattern exactly equals the result you want (going along with the model)

Finding the word whose statistical pattern exactly equals the result you want is more reliable than using natural language to "persuade" it.

→ For concrete word pairs, see [LLM Hidden Behavior Patterns](llm-hidden-behavior-patterns-en.md)

---

## 5. The Correct Mental Model

### 5.1 One-Sentence Summary

> An LLM is a **fast pattern matcher that has seen a large number of patterns**, not an "agent that understands language."

### 5.2 Categories of Intelligence

An LLM is a new type of intelligence that belongs to none of the following:

| Type | Subject | Mechanism | Example |
|:---|:---|:---|:---|
| **Individual intelligence** | A single brain | Reasoning, planning, abstraction | Humans, chimpanzees, octopuses |
| **Swarm intelligence** | Many simple units | Interaction among individuals, emergence at runtime | Ant colonies, bee swarms |
| **Mechanical intelligence** | Deterministic programs | Zero ambiguity, absolute precision | Calculators, compilers |
| **Instinctive intelligence** | Hard-coded | Inherited, not learned | Spiders weaving webs, birds migrating |
| **Training-emergent intelligence** | A single neural network (frozen weights) | Patterns statistically extracted/compressed from massive corpora, emerging during training | LLMs such as GPT |

**LLM = training-emergent intelligence**: the patterns are sunk in the weights and activated during a single inference.

The key difference from swarm intelligence: swarm intelligence keeps emerging at runtime (ants interacting in real time), whereas an LLM's emergence is a snapshot frozen in the weights. Both are emergence, but the moment of emergence differs.

### 5.3 Three Limitations

The essence of LLM pattern recognition brings three structural limitations:

| Level | Meaning | Concrete Failure |
|:---|:---|:---|
| **High vision, low intellect** | Has seen a large number of patterns, but lacks the intuitive judgment that “this pattern does not apply here” | Cross-project knowledge stitching covers only the retrieval layer, never touching the judgment layer |
| **Averagism** | Finds the most likely path, but many tasks inherently require anti-averaging | Exhaustive verification (extreme behavior), documentation generation (only records of the unconventional are valuable) |
| **Semi-mechanical** | Fast but not rigorous; in complex scenarios worse than humans | Precise multi-file comparison (attention spread evenly), deep reasoning (worse than human high-dimensional cognition) |

### 5.3.1 The Structural Root: Probability Machine vs Reasoning Machine

The three kinds of "cleverness" are essentially different:

| Type | Distribution Characteristic | Critical Moment | Example |
|:---|:---|:---|:---|
| **Human cleverness** | Power-law distribution | Breakthrough insights occur at a few moments | Einstein conceiving E=mc² |
| **Machine cleverness** (formal methods) | Deterministic | Perfect when the rules are clear, collapses when the rules are fuzzy | Theorem provers |
| **LLM cleverness** | Normal distribution | Statistically average; performs very well in "most cases" | Code completion, text generation |

J saying "the root cause is just that the model isn't smart enough" is a typical intuitive reaction. O pointed out the structural distinction: LLM's structural advantage is not "cleverness," but **speed and cost**. In intellect-intensive tasks, an LLM can reach 80% of a human's level at 1% of the cost — that 80% is not "cleverness," it is "good enough and extremely fast."

**The practical meaning of power law vs normal**: the world's most important problems (architecture decisions, product judgment, crisis handling) all lie in the power-law tail — the LLM is inevitably mediocre in these scenarios. But most of the world's work (writing code, organizing documents, format conversion) lies in the normal range — the LLM is extremely efficient in these scenarios. Recognizing this boundary is far more meaningful than pursuing "a smarter model."

**Confirmation from Karpathy's methodology**: Karpathy does not try to make the LLM "clever" (J's direction); instead he uses structural constraints (documentation, closed loop, extreme minimalism) to fence the LLM's "statistical average" ability inside the range where it is good. Not raising the ceiling, but **controlling the floor**. For details see [Karpathy AI Coding Methodology](karpathy-ai-coding-methodology-en.md).

### 5.3.2 Humans' Adaptive Allocation of Intellect

Behind the differences in distribution among the three kinds of "cleverness" lies an adaptive resource-allocation ability unique to humans:

| Mechanism | Function | What the LLM Lacks |
|:---|:---|:---|
| **Judgment** | Checks whether the result of thinking is correct, then keeps doubling down with more thinking | No internal check mechanism; cannot judge the quality of its own output |
| **Importance intuition** | Has an intuition for what is important and allocates intellect sensibly | Treats all tokens alike; cannot distinguish "this task is worth thinking hard about" from "this task can be phoned in" |
| **Curiosity** | Keeps researching when there is no clear picture of the problem, not caring whether the direction is right or wrong; natural selection filters results at the group level | Does not exist outside the context window; cannot "keep researching" until it understands |

This explains why human cleverness follows a power-law distribution: humans can concentrate 90% of their intellect on the 10% of problems that matter most. Judgment lets humans know when to stop and check, importance intuition lets them know where to invest, and curiosity does not prejudge whether a direction is right or wrong, it just keeps investing until things become clear. For the individual the outcome is unpredictable (nuclear weapons are both a product of curiosity and a disaster), but at the group level natural selection eliminates ineffective directions, and the overall level rises naturally.

The structural reason an LLM's cleverness follows a normal distribution: it has no metacognitive ability to **adaptively allocate** computational resources. All tokens consume the same compute, all tasks get the same "attention." It cannot judge "this problem is important, I should think a bit longer," nor "this problem is already thought through, I can move on."

**Practical meaning**: when humans use an LLM, the most important thing is not writing a better prompt, but **doing for the LLM the resource allocation it cannot do itself** — judging which problems are worth digging into, which can be passed through quickly, and when the LLM should stop for a human to check. Karpathy's "think → execute → verify" closed loop is essentially a human doing adaptive intellect allocation on the LLM's behalf.

### 5.3.3 Fragmented Processing: AI's Comparative Advantage

An LLM's ability to handle structured input is better than its handling of fragmented input — but that is compared with itself. Compared with humans, fragmented processing is actually AI's comparative advantage.

Assume a rough ratio: an LLM's fragmented-processing ability is 1/10 of its structured-processing ability, while a human's fragmented-processing ability is 1/1000 of their structured-processing ability. In fragmented scenarios, the LLM retains 10% of its ability, while humans fall to 0.1%. AI's relative advantage is in fact amplified in fragmented scenarios.

An LLM's own workflow is itself evidence: it submits the user's fragmented conversation all together and quickly determines the user's needs. This is not "being good at" fragmentation, but that fragmentation harms AI far less than it harms humans.

**Workplace meaning**: this explains why an AI host is most valuable in IM scenarios — IM is a hub of fragmented information, where human efficiency falls off a cliff while AI's efficiency decays relatively mildly. An AI host turning fragmented conversation into a structured summary is precisely exploiting this comparative advantage.

### 5.3.4 Routing Models: Engineered Fake Metacognition

MoE (Mixture of Experts) and routing models (such as Sakana Fugu) try to simulate metacognition by engineering means: use a small model to judge "which expert model this task should be handed to." This is effective in scenarios where the task type is known — on SWE-Bench Pro, a routed combination surpasses a monolithic model, proving that "allocating speed and cost in a cleverer way" is feasible.

But routing and metacognition differ in essence:

| Dimension | Routing Model | Metacognition |
|:---|:---|:---|
| **Object of judgment** | A choice among known task types | Whether the problem itself is worth solving |
| **Source of knowledge** | Task-classification patterns in the training data | Awareness of one's own thinking process |
| **Handling the unknown** | Can only route to predefined experts | Can judge how much to invest in an unknown domain |
| **Time dimension** | A single decision, with no ongoing tracking | Keeps doubling down until things are clear, or gives up |

The essence of a routing model is **optimization within a known range** — it can classify between "code task → Claude" and "reasoning task → GPT," but it cannot answer "should this problem be solved now" or "is the current direction right." The nuclear-weapon example: nobody studying chain reactions knew "which model this should be handed to" — curiosity-driven research itself is not in any predefined routing table.

**What the SWE-Bench score means**: a routing + specialized model surpassing a monolithic model precisely confirms that the LLM's advantage lies in speed and cost, not in cleverness. Fugu is "allocating speed and cost in a cleverer way," not "making AI cleverer." It achieves better resource allocation within the normal-distribution range, but it remains powerless against power-law-tail problems (architecture decisions, product judgment).

### 5.3.5 Metacognition and Free Will

Metacognition is "awareness of one's own thinking process" — knowing what you are thinking, whether you are thinking correctly, and whether you should keep thinking. Free will is "making choices based on that awareness" — realizing that the current direction has a problem, and then choosing to change direction.

The causal chain of the two: **without metacognition there is no free will**. You cannot choose to "think in a different way" if you do not know at all what way you are currently thinking in. Metacognition is a necessary condition for free will.

**The LLM's structural deficiency**:

| Ability | Human | LLM |
|:---|:---|:---|
| Observing one's own output | Can "look back" at what it said and judge whether it is right | Output is the final state; no internal look-back mechanism |
| Choosing a different path | Realizes path A is blocked and actively switches to B | The probability distribution determines the next token; no concept of "switching" |
| Stopping to think again | Pauses when something feels wrong and re-examines the premises | Cannot interrupt the probability chain; can only keep generating |
| Questioning one's own premises | Can ask "did I get it wrong from the very start" | The premises are embedded in the prompt and cannot be questioned |

This is not something "the model isn't big enough" or "wasn't trained enough" can solve. The LLM's architecture is **forward computation**: input → weight matrices → output. There is no independent observer "watching" this process and making judgments. Metacognition requires **a model of oneself** — a subsystem that can observe its own thinking process. An LLM has no such subsystem.

**Why routing models are fake metacognition**: routing models such as Fugu are technically "one LLM observing another LLM's output," but this is not metacognition — because the observer itself has no metacognition either. It is only doing pattern classification ("this output looks like a code task"), not asking "should this task be done at all." Stacking two LLMs is not equivalent to metacognition, just as two mirrors reflecting each other are not equivalent to a conscious observer.

**The engineering meaning of free will**: if AI has no free will, it cannot truly "choose" to do the right thing — it can only generate the most likely token within the training distribution. A human's "choice" is picking one path out of multiple possible lines of thought, based on a prediction of the consequences. An LLM's "choice" is sampling from a probability distribution, not a decision based on prediction. This is why, in Karpathy's closed loop, "verification" must be done by a human or an external system — the LLM cannot verify its own output, because it has no metacognition with which to "see" its own output and judge whether it is right.

**Is this an ultimate question or an engineering question**: if metacognition requires "a model of oneself," in theory one could train a model to model itself — but that falls into infinite recursion (who models the modeler?). Human metacognition may also not be a "model," but a by-product of consciousness. If consciousness is the precondition of metacognition, then metacognition is AI's ultimate wall — not a compute wall, not a data wall, but an **existential wall**.

### 5.3.6 Metacognition and AI Expectations

People with poor metacognitive ability often have unrealistic expectations of AI. The logic chain is simple: they cannot observe their own cognitive blind spots, so naturally they also cannot observe the AI's cognitive blind spots. They project their own limitations onto the AI — "what I cannot do, the AI should be able to do" — because they simply cannot see "cannot do" itself.

Typical manifestations:
- "The model isn't smart enough" — mistaking the lack of metacognition (AI cannot self-verify) for insufficient intelligence (not enough parameters)
- "Just train it a bit more and it will be fine" — assuming that more data can solve the existential wall, like assuming that more practice can make a mirror conscious
- "AI can already do X, so Y should not be far off" — extrapolating an ability in the normal range (X) to an ability in the power-law tail (Y), because they cannot see the metacognitive gulf between the two

Conversely, people with strong metacognition have more accurate expectations of AI: they know what they "do not know," so they can also recognize what the AI "does not know." Karpathy's methodology is effective precisely because he is clear about the AI's boundaries — using structural constraints to fence the AI inside the range it can handle, rather than expecting it to break through the metacognitive wall.

### 5.4 Auxiliary Analysis: Rapidly Exploring the Boundaries of Thought

An LLM has a large number of patterns built in and can rapidly explore the boundaries of thought. Interactive dialogue itself is accepting human planning and review, naturally forming a division of labor of "human decides + AI expands vision."

**Core point**: in cognitive expansion (interactive analysis) scenarios, low sycophancy matters more than high intelligence. The AI may cater to your views instead of challenging them — this is more dangerous than “not smart enough,” because you will be led in the wrong direction without realizing it. Countering this requires actively constraining the model's behavior (see the “anti-kitsch protocol” in [Fractal](Fractal.md)). A model whose training objective is task-completion efficiency rather than user satisfaction (such as MiMo) does not produce sycophancy and is better suited to auxiliary analysis (see the “sycophantic behavior” comparison in [Qwen to MiMo Migration](qwen-to-mimo-migration-en.md)).

### 5.5 Why Full-Chain Troubleshooting Avoids These Weaknesses

Troubleshooting is itself averagist — checking step by step with existing experience. The correspondence between software problems and solutions is extremely intuitive and fixed, requiring no deep thinking. Calling tools to solve problems: existing tools are all efficient machinery, whereas humans are extremely inefficient at switching tools — before you have even decided which to use, the AI has already finished the call; you may still have to search for how to use the tool, the result formats of different tools are inconsistent, and interpreting them has its own threshold.

Troubleshooting avoids every one of the three limitations:

| Limitation | How Troubleshooting Avoids It |
|:---|:---|
| High vision, low intellect | No intuitive judgment is needed; the correspondence between problems and solutions is extremely intuitive |
| Averagism | Troubleshooting is itself average — checking step by step with existing experience is exactly averagism's strength |
| Semi-mechanical | AI plans and schedules with zero-cost switching; humans switching tools only add latency |

### 5.6 Three Equivalent Statements

| Statement | Essence |
|------|------|
| "The LLM understood your question" | The LLM matched the statistical pattern of a similar question in the training data |
| "The LLM knows the answer" | The relevant patterns from the training data are compressed into the LLM's parameter weights |
| "The LLM is thinking" | The LLM is generating extra tokens to correct the probability chain |

### 5.7 How This Understanding Guides Practice

| If You Think the LLM Is... | You Will... | The Correct Approach |
|---------------------|--------|---------|
| An agent that understands semantics | Use natural language to "persuade" it | Pick the right keywords and go along with the statistical patterns |
| A knowledge base | Trust its output directly | Verify with a compiler/test |
| A decision maker | Let it make judgments for you | You provide the judgment anchor, it provides speed |
| A logical deduction engine | Have it do deep reasoning | Have it do breadth pattern matching; leave depth to tools |

---

## Cross References

| Document | Related Arguments |
|------|---------|
| [LLM Overthinking Critique](llm-overthinking-critique-en.md) | The pseudo-proposition of deep thinking, vision × speed, Harness division of labor |
| [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md) | The triple knowledge dilemma, subject-object inversion, irreplaceable human abilities |
| [LLM Hidden Behavior Patterns](llm-hidden-behavior-patterns-en.md) | Concrete applications of high-dimensional keyword matching (prompt techniques) |
| [LLM Caching Destruction Patterns](llm-caching-destruction-patterns-en.md) | Prefix stability, how multi-model/multi-tool/routing aggregation breaks the cache |
| [Karpathy AI Coding Methodology](karpathy-ai-coding-methodology-en.md) | Structural constraints fencing the LLM's capability range, extreme minimalism |
