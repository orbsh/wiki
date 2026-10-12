# LLM Hidden Behavior Patterns and How They Are Triggered

> **Languages:** [English](llm-hidden-behavior-patterns-en.md) (primary) · [中文](llm-hidden-behavior-patterns.md)
> LLM has a set of "default output templates" for every word. The essence of prompt engineering is not adding constraints, but choosing the right template.

> Why LLM has "default output templates" — its foundational cognitive mechanisms (probabilistic prediction, statistical pattern matching) are detailed in [LLM Fundamentals](llm-fundamentals-en.md).

---

## Core Principle

LLMs are trained on massive amounts of text, and every word is associated with an output distribution of a specific style. The training data for "summarize" is abstracts (compression), while the training data for "aggregate" is reports (fidelity). When you use a certain word, what you activate is not semantics, but a **stylistic template**.

```
Adding constraints = trying to override the default template (fighting the model, poor results)
Swapping a word = directly choosing the right template (going along with the model, good results)
```

### Underlying Mechanism: High-Dimensional Keyword Matching

LLMs appear to go far beyond keyword recognition — they are natural, flexible, and able to handle complex semantics. This appearance creates a reverse misconception: "using keywords is too low-level; LLMs understand semantics."

But the fact is: **LLMs have learned a high-dimensional version of "keyword recognition" from massive corpora**. "Summarize" and "aggregate" are almost synonymous semantically, yet LLMs respond to them in utterly different ways — because the context distributions in which these two words appear in the training data differ. LLMs do not "understand" the difference in meaning between these two words; they merely **match different statistical patterns**.

```
Human perspective: summarize ≈ aggregate (semantically close)
LLM perspective: summarize → [abstract corpus distribution], aggregate → [report corpus distribution]
          ↑ two completely different activation paths
```

This is why swapping a word is more effective than adding a paragraph of constraints — you are not talking to an "intelligent agent that understands semantics", you are talking to an "engine that matches statistical patterns". Finding the word whose statistical pattern happens to equal the result you want is more reliable than using natural language to "persuade" it.

A good prompt is not about adding more constraints, but about finding the word that makes the LLM's default behavior happen to equal the result you want.

---

## I. Information Density Control

| What you want | Don't say | Say | Principle |
|---------|------|-----|------|
| Faithful, not compressed | Summarize this conversation | **Aggregate** this conversation | Summarize = compression mode (discarding "redundancy"), aggregate = organization mode (faithful arrangement) |
| Key points, not verbose | Write an article | **List** it out | Write = triggers prose generation, list = triggers structured extraction |
| Precise, not expanded | Explain it | **State** it | Explain = textbook mode (verbose + condescending), state = directly give the conclusion |
| Concrete, not abstract | Study this project's architecture | **Copy** this project's architecture | Study = abstract principles, copy = concrete code snippets |

---

## II. Behavior Direction Control

| What you want | Don't say | Say | Principle |
|---------|------|-----|------|
| Concise, not bloated | Optimize this code | **Reduce** this code | Optimize = AI adds functionality/abstraction by default, reduce = triggers deletion behavior |
| Fill only the gaps | Modify it | **Patch** it | Modify = may rewrite, patch = only fill in the missing parts |
| Decisions, not waffle | Help me design a system | **Decide** on a plan | Design = over-engineering, decide = make a decision, pick one |
| Decisive judgment | Discuss this plan | **Judge** this plan | Discuss = present both sides, balanced; judge = give a conclusion, hand down a verdict |

---

## III. Interaction Mode Control

| What you want | Don't say | Say | Principle |
|---------|------|-----|------|
| Just do it, no waffle | Help me make an X | Make an X | "Help me" triggers pleasantries + a confirmation process; remove it to trigger direct execution |
| Strict nitpicking | Evaluate this code | **Critique** this code | Evaluate = gentle and balanced, critique = rigorous critique |
| Confident diagnosis | It might be an X problem | It **is** an X problem | "Might" triggers uncertainty/hedging, a direct statement triggers a confident judgment |
| A plan, not a list | Suggest a plan | **Choose** a plan | Suggest = list multiple options for you to pick, choose = make the decision for you |

---

## IV. Thinking Depth Control

| What you want | Don't say | Say | Principle |
|---------|------|-----|------|
| Fast and practical | Contemplate it | **Think** about it | Contemplate = triggers internal monologue (long), think = triggers a fast practical judgment |
| Fill the gaps | Refine this design | **Patch** this design | Refine = add new things, patch = only fill in what is missing |
| Subtraction | Refactor this code | **Cut** this code | Refactor = may add abstraction layers, cut = only delete the unnecessary |

---

## V. Summary of Principles

### 5.1 Why It Works

The output distribution of an LLM is a statistical reflection of the training data:

- The context in which "summarize" appears = abstracts, outlines, compressed text → output tends toward compression
- The context in which "aggregate" appears = reports, lists, tables of contents → output tends toward preservation
- The context in which "optimize" appears = enhance, upgrade, improve → output tends toward adding things
- The context in which "reduce" appears = trim, streamline, slim down → output tends toward removing things

The word you choose determines which stylistic distribution the LLM samples from.

### 5.2 Why Constraints Are Inferior to Swapping Words

Constraints are an **external force** — you tell the model "don't do X", but the model's default behavior is still to do X, only with an extra layer of checking before it does. This layer of checking gradually fails as the context grows longer and attention becomes diffuse.

Swapping the word is an **internal cause** — you change the input distribution, and the model will not produce what you do not want at the root. No fighting is needed, and there is no decay.

### 5.3 General Methodology

When you find that you need to add a large number of constraints to get the result you want:

1. **Stop**: too many constraints = you are fighting the model's default behavior
2. **Find the style**: which style does the output you want most resemble? (Report? Verdict? List? Code?)
3. **Find the trigger word**: what word do the titles/openings of that style usually use?
4. **Swap and test**: replace the current verb with that word and see whether the output naturally meets expectations

> The best prompt is a single word, not a paragraph.
