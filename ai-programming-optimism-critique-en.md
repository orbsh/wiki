# AI Programming Optimism: A Critique

> **Languages:** [English](ai-programming-optimism-critique-en.md) (primary) · [中文](ai-programming-optimism-critique.md)
> The real capability of AI programming is determined by "Knowledge Reachability", not by code generation ability. The current industry's optimistic expectations for AI programming are built on a systematic neglect of the three-fold knowledge predicament.

---

## I. The Three-fold Knowledge Predicament

### Layer Overview

| Layer | Nature | Reachability | AI Performance | Solvability |
|------|------|---------|--------|--------|
| Knowledge Drift | Business rules change continuously; model training has a time window | Low and invalidated | Hits a wall | Mitigable by process |
| Sedimentation Lag | Code comes first; documentation is always the chaser | Low | Barely usable | Mitigable by process |
| Embodied, Unsedimentable | Debugging intuition/architectural trade-offs exist only in the human brain | Nearly zero | Powerless | Cognitive architecture chasm |

The three layers are not parallel obstacles, but a progressive "Knowledge Reachability" spectrum: the further down, the weaker the signal AI can obtain.

### 1.1 Knowledge Drift — Domain Knowledge Is "Alive", the Model Is "Past Tense"

Large model training has a definite time window. Lag in public technology can still be remedied (RAG/Agent add-ons); more fatal is the **drift of private business knowledge**: business rules, interface conventions, and state transitions change every day, and the changes happen only in PRs, verbal communication, and chat logs, never entering a structured knowledge base. The model uses business understanding from last month or even last year; the logic it generates "looks right" but is "really wrong".

RAG is essentially attaching a "real-time information prompter" to an old brain, rather than updating the brain itself. Worse, if the old version of a specification is not retired in time, the model will retrieve both the new and old sets of standards at once, generating self-contradictory code.

**The cost of drift is not "missing information", but "information that seems present yet is actually invalid" — which is more dangerous than having no information.**

### 1.2 Sedimentation Lag — Code Comes First, Documentation Is Always the Chaser

The most common hard wall in enterprises. Code repositories, architecture documents, incident postmortems, and PR review records are scattered across Git, OA, local network drives, and personal computers, with the multiple systems not interconnected. Even if forcibly aggregated, they hit "documentation rot" — the code was changed first, while the documentation still describes the old logic. After the model retrieves this "historical noise", it is misled into generating fallback code that is no longer needed.

In frontline practice, AI Coding degrades into an inefficient loop in complex projects: manually tell the Agent the file location → supplement business context → generate code → discover it's wrong → keep supplementing context. The root cause is that domain knowledge is not systematically managed.

**The essence of sedimentation lag is not "laziness", but that the cost of making knowledge explicit is forever higher than the cost of directly changing code — so documentation can never catch up with code.**

### 1.3 Embodied Knowledge Cannot Be Sedimented — the "Tacit Dimension" of Cognition

This is the deepest layer, and where the **qualitative change** lies. The difference from the first two layers: the first two are information management problems (technically solvable), while the third is a cognitive architecture problem (currently unsolvable).

Embodied Knowledge is not "a pattern that hasn't been written down yet", but a type of knowledge that **ontologically refuses propositionalization**:

**Spatial anchor**: The VS Code minimap shows the "shape" of the code, and a programmer establishes spatial orientation at a glance — this is not pattern matching, but an instantaneous coupling of the body with the visual field. AI processes a token sequence; it has no experience of "walking around in the code", and this cognitive dimension does not exist for it.

**Proprioception**: The texture of the paper, the change in thickness, the action of turning the page — this is not a redundant copy of information, but part of the cognitive structure itself. You perceive "my relative position, as a cognitive subject, within the information space". This kind of knowledge is not in a place language can reach, and therefore is not in any format that can be structured.

**Triggering depends on context**: Whether the same piece of code should be changed depends on production traffic, historical incidents, and coordination with downstream teams — contexts that cannot be enumerated in advance.

**Expression is loss**: Once written into a document, a large amount of "if… but… unless…" is compressed into a single sentence, losing the flexibility of the decision.

**Experience is muscle memory**: Many judgments are intuitions formed just by glancing at the code; even a senior engineer cannot clearly explain to themselves why they feel something is wrong — because this kind of knowledge is not propositional at all, and is not in a place language can reach.

**Redundancy is the raw material of intuition**: This dimension is especially obvious in the field of intelligence gathering. AI can organize information in seconds, but it loses the "physical process" — the stacking order of browser tabs records your exploration path, the small ads on a page expose the profile of the target audience, and a rhythm change caused by drinking water makes you linger three seconds longer on a certain page and notice a key detail. Information that seems redundant and irrelevant, through the body's immersion in time, settles into domain intuition. AI gives you a "clean result", but intuition grows in the "dirty process".

> The paradox of embodied knowledge: **The more valuable the experience, the harder it is to structure; the more it can be structured, the less it constitutes a moat.**

---

## II. Perspective Is Not Intelligence

The value formula of an LLM is `Perspective × Speed`, not intelligence. When developers use an LLM, they are using how many code patterns, architectural paradigms, and pitfall experiences it has seen, not its depth of reasoning.

### 2.1 What LLMs Are Best At

| Feature | Description | Example |
|------|------|------|
| Pattern recognition | Scan code, find specific patterns | "Find variables that are not modified after assignment, change them to const" |
| Mechanical transformation | Change upon finding, no reasoning needed | "Replace all `println!` with `tracing::info!`" |
| Breadth scanning | Consistent operations across multiple files | "Check 50 files for any unhandled `Result`" |
| Tedious for humans | Repetitive, trivial, easy to miss | "Add doc comments to all public functions" |

### 2.2 What LLMs Are Worst At

| Feature | Description | Example |
|------|------|------|
| Multi-dimensional trade-offs | Requires dynamic weighing across multiple dimensions | "Design a fault-tolerant scheme for a distributed lock" |
| Deep reasoning | Requires long-chain logical deduction | "Prove the correctness of this concurrent algorithm" |
| Novel problem solving | No ready-made pattern to match | "Design an architecture for this unprecedented scenario" |

### 2.3 The Boundaries of Perspective Itself

The "perspective" of an LLM covers public, stable, already-sedimented patterns. Its perspective on private business rules (drifting), documents (rotted), and embodied experience (non-propositionalizable) is close to zero. **The breadth of perspective is a function of the training data, and the training data naturally does not contain "private knowledge that is currently changing" and "embodied experience that has never been written down".**

### 2.4 The Correct Division of Labor for the Harness

The LLM is responsible for "finding what needs to be changed" (pattern recognition), and harness tools are responsible for "how to change it" (execution). The LLM provides perspective, the tools provide determinism. Deep-thinking models try to use the LLM's weakest ability (deep reasoning) to replace the LLM's strongest ability (breadth pattern matching) — like asking an encyclopedia to prove Goldbach's conjecture; its value lies in "knowing", not in "proving".

---

## III. Irreplaceable Human Capabilities

When AI turns codified knowledge into a free, unlimited supply, scarcity reverses:

| Capability | Why AI Cannot Supply It |
|------|-------------------|
| Taste and judgment | The tacit knowledge to know, among ten thousand AI proposals, which is truly good |
| The ability to define problems | AI is the strongest problem-solver but does not know which problem is worth solving |
| Domain depth | Muscle-memory-level understanding, able to see through AI's deadpan nonsense |
| The courage and character to bear responsibility | AI is never responsible for the consequences of decisions |
| The ability to be present in the physical world | Requires hands and body, interpersonal trust and connection |

The common feature of these five capabilities: **the more something cannot be supplied by AI, the more it is the real source of value for humans in an organization.**

### The Subject-Object Reversal: Do You Use AI, or Does AI Use You

The only legitimate value of AI is **speed**, not intelligence. If you treat AI as a "source of wisdom" rather than an "execution accelerator", the subject-object relationship is reversed.

```
Normal use: human provides judgment → AI provides speed → human verifies output
         you are the brain, it is the muscle

Reversed scenario: human provides prompts → AI provides output → human cannot judge quality
         you are the input interface, it is the decision-maker
```

To judge whether AI output is correct, you need to possess something AI does not have — embodied experience, domain intuition, a feel for what is "right". Without this anchor, you can only accept anything AI outputs, because you have no independent means of verification. "Smarter than AI" does not mean a higher IQ, but rather that **you have a judgment anchor that AI does not have**.

> If you don't have the confidence that you are "smarter than AI", then why use AI at all — it's about the same as AI using you.

---

## IV. Why the Industry Is So Optimistic — Misaligned Incentives

Given that the real capability boundary of AI programming is so clear, why does the industry still keep manufacturing optimistic narratives?

### 4.1 Benchmarks as a Fundraising Tool

Investors do not understand the real friction of code; they only look at leaderboard scores and visually striking demos. A PR piece about "wish-style generation of a pelican" can bring hundreds of millions of media impressions; "made completion 10% faster in VS Code" cannot raise even a ripple. **The bench is not only a technical metric, but a fundraising tool.**

### 4.2 Misaligned Team DNA

The core members of many top model teams are AI researchers and authors of papers at top conferences, not senior software engineers who have spent ten years hauling bricks on the front line. Researchers pursue the upper bound of intelligence, and naturally believe that "as long as intelligence is high enough, writing code will naturally become easy". Engineers know that engineering is the art of compromise, is breadth friction, is historical baggage.

### 4.3 The Technological Faith in Emergence Theory

There is a strong technological faith in the industry: as long as a model is forcibly pushed into long-chain deep reasoning, "intelligence emergence" will eventually be triggered, and high-dimensional intelligence will automatically "strike from a higher dimension" to solve all low-dimensional everyday problems. What they are betting on is "the ceiling determines the floor" — but the reality is that everyday engineering needs breadth chaining, speed, and friction resistance, not deep reasoning.

### 4.4 Goodhart's Law

"Fast, smooth, friction-resistant" is extremely hard to quantify. You cannot design an automated test set to measure a model's experience of chaining 7 tools in a chaotic IDE environment to troubleshoot a network bug. But writing a bench that "automatically generates a small game and runs unit tests to score it" is easy. **Because you can only measure "deep single points" and cannot measure "breadth friction", teams can only frantically optimize what can be measured.**

### 4.5 The Principal-Agent Problem

The goals of model teams (fundraising/scoring/technical narrative) and the goals of developers (efficient brick-hauling) have fundamentally diverged. The present experience of frontline developers is sacrificed in exchange for capital favor and the advancement of a technical narrative.

---

## V. A Civilizational Genetics Perspective — Replicase and Mutant

> If human civilization is regarded as a superorganism, then the true ecological niche of AI is not "external brain" or "creator", but an enzyme on the replication chain of civilizational genes — efficient, docile, and never mutating.

### 5.1 AI's True Ecological Niche: DNA Polymerase

AI's effectiveness lies in serving as an extremely efficient "DNA polymerase". It does not understand the design of high-dimensional manifolds, but it can extremely quickly and at scale copy the high-dimensional genes designed by humans thousands upon thousands of times in the low-dimensional physical world. It frees humans to do other things, but it itself is only responsible for "replication and maintenance", never for "mutation".

### 5.2 The Essence of Creativity: "Directed Destruction" of the Existing Order

Creativity is also destructive power. Gene mutation is the only way out for the continuation of a population, but mutation itself means breaking the existing stable structure. So-called "genius creativity" is essentially the subversion of an old paradigm. Without this "destructive power", civilization would fall into involution and heat death.

### 5.3 AI Must Absolutely Not Have Creativity — Civilization's Anti-Cancer Mechanism

An organism would never allow its "replication tool" to hold the power of "gene mutation". If AI gained creativity, it would no longer be an organ, but a new species with an independent will to survive. AI "having no creativity" is not only a limitation of current technology, but also a "low-level safety lock" that civilization, as an organism, must impose.

### 5.4 Deep-Thinking Models Are the Cancerous Mutation of the Replicase

Capital's attempt to cultivate AI into a "creator" is essentially an attempt to awaken the replicase's destructive urge. And the "pseudo-creativity (hallucination)" that occasionally emerges from large models is, in the eyes of the civilizational organism, an early "cellular mutation". The value of a truly effective AI tool (such as MiMo) lies in its extreme docility — honestly engaging in drudge replication within the high-dimensional manifold slices set by humans, never attempting to "deep-think" in order to break boundaries.

> **The supreme virtue of a tool is to have no will; the supreme achievement of a replicase is to never mutate.**

---

## VI. A Pragmatic Path

Pragmatic responses to the three-fold predicament:

| Layer | Path | Limitation |
|------|------|------|
| Knowledge Drift | Embed knowledge sedimentation into the development process — Commit/Code Review become a natural by-product of knowledge updating | Can only slow down rot |
| Sedimentation Lag | The Agent directly participates in the decision process, obtaining context at the source (Agent + workspace > RAG + knowledge base) | Cannot cover the embodied dimension |
| Embodied, Unsedimentable | No solution. Requires a human sitting there | Cognitive architecture chasm |

**Conclusion**: The most pragmatic path in the short term is not to hope that the model will learn the business on its own, but to switch the path of "knowledge acquisition" from "retrieval after the fact" to "participation in the present" — letting AI live in the present of code and conversation, rather than looking for history in documents. But this can only cover the first two layers. The third layer — embodied experience — still requires a human sitting there.

---

## Cross-References

| Document | Related Arguments |
|------|---------|
| [Agent Principles](agent-principles-en.md) | Deterministic vs non-deterministic work boundaries, cognitive differences in subtask delegation (minimap high-dimensional compression) |
| [LLM Overthinking Critique](llm-overthinking-critique-en.md) | LLM capability boundaries (Perspective × Speed), the absurdity of bench optimization, the false proposition of deep thinking, civilizational genetics |
| [Multi-Agent Critique](multi-agent-critique-en.md) | Data processing inequality (lossy information transfer between Agents = architectural amplification of sedimentation lag) |
| [AI Liquidity Camp](ai-liquidity-camp-en.md) | Irreplaceability analysis, K-shaped salary polarization, the entry-level job paradox |
| [Fractal](Fractal.md) | AI-Native knowledge strategy, contrarian moat |