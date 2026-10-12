# LLM Overthinking Critique

> **Languages:** [English](llm-overthinking-critique-en.md) (primary) · [中文](llm-overthinking-critique.md)

**Status**: Team technical consensus
**Date**: 2026-06-22
**Core thesis**: Deep-reasoning optimization aimed at benchmarks is fundamentally mismatched with the needs of real software engineering—the higher the score, the worse the work

> Large models currently face a dead end of being "neither high nor low." The root cause is this: model teams, under the grip of capital and academic KPIs, chose a benchmark-oriented "deep thinking" technical route, which has a fatal mismatch across the board with the "breadth-chaining" essence of real software engineering, the core demands of senior developers, and the physical/logical ceiling of large models themselves.

---

## 1. The Core Paradox: The Draft Notebook's Dual Personality

An autoregressive LLM has no internal working memory; it is merely a "next-token probability predictor." An explicit draft notebook (` thinking`) acts as a physical buffer—if the model writes one wrong character at the first step, the probability chain breaks and the following thousands of words all become hallucination. The draft notebook allows the model, before emitting real code, to make mistakes in its head, strike them out, start over, and complete self-purification.

> For the basic cognition of LLMs (probability predictor, no internal state, pattern-matching essence), see [LLM Fundamentals](llm-fundamentals-en.md).

**But in a thick Harness environment, the external Harness itself is the ultimate draft notebook.**

LSP, compilers (`cargo check`), and test runners are 100% deterministic physical evaluation engines of reality. If a reasoning model spends 30 seconds inside `</arg_key>` simulating "how the Rust compiler would see this lifetime error," it is doing useless work with an extremely expensive and slow attention mechanism—the real compiler gives the most precise answer within 10ms.

An even more extreme case: a conventional model gives the correct answer within 10 seconds, while a reasoning model thinks for 10 minutes and gets it wrong instead. Overthinking refers to thinking that exceeds 10 seconds, not the absence of thinking—thinking within 10 seconds is the normal draft-notebook mode, letting the model self-check and self-correct before output. Beyond this threshold, thinking turns from course-correction into a hotbed for probability-chain drift, giving the model more chances to err. Extreme scenarios (competition-grade math problems) can indeed raise scores through long thinking, but they are unusable in experience and the cost is also unacceptable.

**Three reasons a thick Harness does not need deep CoT**:
- **External course-correction is equivalent to in-head deliberation**: modern IDEs and agent frameworks possess a complete physical closed loop of "read code → flag errors in red → run tests → feed back fixes," which is far more efficient and far more deterministic than the model simulating the compiler in its head.
- **Overthinking destroys flow**: under long-context contamination in a thick Harness, reasoning models are highly prone to triggering gratuitous self-reflection, producing extremely high first-token latency.
- **A commercial compute disaster**: in low-difficulty scenarios the conventional model's intuitive accuracy is already 95%+; making users wait dumbly through thousands of hidden thinking tokens for a 0.5% gain is severely over-engineered in ROI.

→ For the taming configuration, see [Qwen to MiMo Migration Appendix C](qwen-to-mimo-migration-en.md)

---

## 2. The Engineering Essence: Complexity Sinks Down, Troubleshooting Is Breadth-First

### 2.1 The Complexity Onion: It Sinks, It Does Not Disappear

The complexity of software engineering does not disappear; it only sinks downward. Lower-level protocols, GC, and concurrency are encapsulated into frameworks and libraries in exchange for the popularization of technology.

### 2.2 Everyday Engineering: Breadth-First Trial and Error, Not Deep Deliberation

The system complexity of a real application is essentially the "breadth of component interactions." Troubleshooting is not deep deliberation over a single point of logic, but long-chain trial and error carried out across different layers by calling dozens of tools (checking logs, watching monitoring, tracing the call chain). The difficulty lies in breadth and triviality.

Even when one hits a lower-level framework bug and is forced to "tear open the encapsulation," in essence this is still just entering the next layer of the onion skin for another round of "breadth-first trial and error." And if one truly encounters a low-level state-machine problem that requires extreme depth of deliberation, large models are equally helpless.

**Conclusion**: what everyday engineering needs is "breadth chaining, speed, friction resistance," not overthinking.

### 2.3 The Capability Boundary of LLMs: Vision, Not Intellect

Ordinary models are adequate in terms of intellect. When developers use an LLM, they use its **vision** (how many code patterns, architectural paradigms, and pitfall experiences it has seen), not its **intellect** (how deep it can deliberate).

**LLM value formula**: `vision × speed`. Not intellect.

LLMs are pattern-matching machines, not logic-deliberation engines. Their value lies in "having seen enough patterns to quickly recognize and assemble them in new scenarios." Humans provide judgment, AI provides speed—if you cannot judge the quality of the AI's output, the subject-object relationship is reversed. See [AI Programming Optimism Critique §Subject-Object Reversal](ai-programming-optimism-critique-en.md).

LLMs are pattern-matching machines, not logic-deliberation engines. Their value lies in "having seen enough patterns to quickly recognize and assemble them in new scenarios." Medium-sized models (7B-70B) are already sufficient in this capability—the code patterns, architectural paradigms, and pitfall experiences they have seen already cover the vast majority of everyday engineering scenarios. Writing architecture documents, refactoring code, troubleshooting bugs—what gets assembled are existing patterns, not newly derived theorems.

**The task profile LLMs are best at**:

| Feature | Description | Example |
|------|------|------|
| Pattern recognition | Scan code, find specific patterns | "Find variables that are not modified after assignment and change them to const" |
| Mechanical transformation | Change it once found, no deliberation needed | "Replace all `println!` with `tracing::info!`" |
| Breadth scanning | Consistency operations across multiple files | "Check whether any of 50 files has an unhandled `Result`" |
| Humans find it annoying | Repetitive, trivial, easy to miss | "Add doc comments to all public functions" |

**The task profile LLMs are worst at**:

| Feature | Description | Example |
|------|------|------|
| Multi-dimensional trade-offs | Requires dynamic weighing across multiple dimensions | "Design a fault-tolerance scheme for a distributed lock" |
| Deep deliberation | Requires long-chain logical deduction | "Prove the correctness of this concurrent algorithm" |
| Novel problem solving | No ready-made pattern to match | "Design an architecture for this unprecedented scenario" |

**The correct division of labor for a Harness**: the LLM is responsible for "finding what needs to change" (pattern recognition), and the harness tools are responsible for "how to change it" (execution). The LLM provides vision, the tools provide determinism. Deep-thinking models try to use the LLM's weakest capability (deep reasoning) to replace the LLM's strongest capability (breadth pattern matching)—like asking an encyclopedia to prove Goldbach's conjecture: its value is in "knowing," not in "proving."

**A supplement on the knowledge dimension**: vision itself also has boundaries. The "vision" of an LLM covers public, stable, already-sedimented patterns; its vision of private business rules (in drift), documentation (already rotted), and embodied experience (not propositionalizable) is close to zero. See [Agent Principles: The Triple Knowledge Predicament](agent-principles-en.md).
**The full argument of this section** (including the triple knowledge predicament, irreplaceable capabilities, and the civilization-gene perspective) see [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md).

### 2.3.1 Five Iron Rules for Using AI Efficiently

**① Context window = the ceiling of working memory**
An LLM has no internal state; the context window is the only working memory. Filling the context = filling it with noise. Effective strategy: streamline the input, rather than expecting the LLM to "pan for gold" from a sea of context. Feed only the necessary code snippets and constraints in one conversation; do not dump the entire project in.

**② LLM self-verification is unreliable**
Having an LLM check its own output is like having the examinee grade their own exam paper. An LLM cannot reliably judge whether its own reasoning is correct—it can only judge "whether the output looks reasonable." The correct approach: verify with deterministic tools (compiler, tests, linter); do not use an LLM to judge the rightness or wrongness of an LLM.

**③ Deterministic tools > probabilistic reasoning**
Anything that can be solved with a linter / cargo check / tests should not be left to the LLM to "think about." The value of the LLM is in "finding what needs to change," the value of deterministic tools is in "verifying whether the change is correct." The two complement each other; do not use an LLM to replace tools, and do not use tools to replace an LLM.

**④ Cost and quality are nonlinear**
10x token consumption ≠ 10x output quality. A deep-thinking model spends 100x the compute and may improve accuracy by only 0.5% on everyday tasks. Medium model + good toolchain > large model running bare. Selection criterion: enough is enough, speed first.

**⑤ LLMs are function callers, not thinkers**
In agent scenarios, the LLM's core action is "deciding which function to call and what arguments to pass." Its value is in orchestration, not in reasoning. The stronger the harness (the more tools and the more deterministic they are), the higher the LLM's orchestration value—because it only needs "one look to know what to call."

→ Related discussion: §1 "The Draft Notebook's Dual Personality" (the physical mechanism by which a Harness replaces CoT); [Agent Principles](agent-principles-en.md) (deterministic work goes to tools, AI handles only the interpretation layer)

### 2.4 Developer Stratification and the Mismatch of Demands

**User stratification**: junior developers lack the engineering beatings and blindly believe AI is "omnipotent"; senior developers stand in awe of complex systems and remain restrained toward AI.

**Mismatch of demands**: for seniors, architecture design is "intellectual entertainment" that enjoys creating order (they are unwilling to cede it to AI), whereas tedious debug troubleshooting is precisely the "grunt work" most in need of AI doing it for them.

**The irony of reality**: models desperately grind away at code generation and architecture (catering to juniors, good for producing demos and topping the leaderboards), yet avoid talking about the tedious chaining of debug toolchains, precisely dodging the real pain points of senior developers.

---

## 3. The Absurdity and Mismatch of Bench Optimization

### 3.1 Directional Mismatch

A large number of models perform extreme optimization aimed at benchmarks (such as generating a pelican riding a bicycle, wish-driven programming), forcibly injecting "deep reasoning" trajectories.

**The contradiction**: everyday engineering needs "breadth chaining" and "speed," whereas bench optimization strengthens "deep thinking." This causes models to try to use "deep logical deliberation" to solve everyday problems that essentially require "breadth-first tool trial and error," producing a severe directional mismatch.

**Verdict**: the higher the score, the more sluggish the everyday work becomes instead. Extreme benchmark-oriented optimization not only fails to benefit real productivity, it becomes a heavy burden in the execution chain.

### 3.2 Bench Is a Fake Battlefield of "Wish-Driven Programming"

Wish-driven programming—making a project out of a few hundred words of requirements—is the typical paradigm of bench testing. For example, "generate a pelican riding a bicycle."

But the problem is: **no model can write a usable pelican.** And compared with a real application, the pelican differs by several orders of magnitude in complexity.


There is a fundamental cognitive mismatch here:

- **Non-trivial applications** (such as the pelican): building a complete, runnable project from zero. This looks like it "requires intelligence," but in actual tests no model can truly complete it—the so-called "high score" is merely "looking closer to completion." Moreover, the pelican's complexity is far lower than that of a real business system.
- **Trivial applications**: AI can indeed do them (large training volume, pattern matching is enough), but the more rational choice is to use ready-made frameworks, generators, and low-code tools, rather than having AI generate a large amount of foundational code from scratch—the latter is **expensive, slow, bug-ridden, and inefficient at runtime**. For example, if a client needs a website, the optimal strategy is not for AI to generate one for each person (later maintenance would be a disaster), but to use AI to build a CMS system and retain the customization points.

**The essence of abstraction**: raise the viewpoint, ignore the details, and turn the non-trivial into the trivial. This relies more on intuition and taste, which AI has no way to learn. What AI can do is execute rapidly within an already-abstracted trivial layer, not perform the abstraction itself.

The real problem with bench tests is not that "the tests themselves are meaningless"—they can indeed distinguish the boundaries of a model's basic capabilities. But **extreme optimization of benchmarks is absurd**, because what it measures is the ability to "build non-trivial applications from zero," and that is precisely the ability least needed in everyday engineering. Spending 10x the compute to optimize a "generate a pelican" score is not as good as improving by 1% the experience of "quickly locating and fixing a bug in an existing codebase."

**Analogy**: this is like how a student who scores full marks in high-school math may not necessarily do factory quality inspection well. Benches measure "problem-solving ability," whereas engineering needs "the ability to quickly locate and fix a problem within an existing system"—two completely different dimensions.

### 3.3 Public Opinion and Cognitive Traps: Why "Paper Leaderboards" Mislead You

| Cloud-review logic | Engineering-practice logic |
|-----------|-------------|
| "The longer the thinking time = the smarter the model = the higher the tier" | The longer the thinking time = the higher the first-token latency = broken development flow |
| "The more expensive the API unit price = the larger the parameter count = the better the deployment effect" | The more expensive the unit price = the more easily the token bill spirals out of control = ROI collapse |
| "Leaderboard score T0 = production environment T0" | Leaderboards are tested with olympiad math problems, production is tested with high-frequency tool calls—two completely different worlds |

This evaluation methodology has never had its flow stuck by a model in Zed, has never burned through several hundred yuan on duplicated chain-of-thought billing in Hermes's thick Harness, and has never experienced the predicament in a live production environment of "turning off thinking mode and even misspelling a URL."

### 3.4 Three Nasty Consequences

**1. Creating the cult of "technical infants," strangling newcomers' engineering common sense**
When overwhelming evaluations all brainlessly shout "Qwen Max/GLM-5.2 are T0 top-tier," newcomers just entering the field will blindly bet on the big vendors' "strong reasoning models." They have no idea that software engineering has two life-and-death metrics: "development flow" and "first-token latency." When the model frequently stutters, newcomers even suspect that there is something wrong with their own code.

**2. Leading enterprises into ROI bankruptcy**
Trusting the "acknowledged tier list," they integrate a high-unit-price flagship reasoning model, and after going live they encounter "one minute of silence when saying hello" and "an assassin bill of several hundred a day." After turning off thinking, it drops severely in intelligence (it cannot even spell a URL correctly), stuck in a dilemma.

**3. Masking genuinely excellent "pure-engineering" models**
Models like MiMo that do not act out inner drama, have high TPS and low cost, are directly demoted to "low-end models" in the eyes of cloud reviews because their parameter count or chain-of-thought length is insufficient. Practical developers then miss out on a productivity weapon that could truly accompany them working overtime into the night.

### 3.5 Why Did the Overall Environment Become Like This?

Because "reading the leaderboard and scoring" is too easy, while "doing hands-on real testing" is too hard. Media and cloud reviews only need to test models with a few olympiad math problems, and after thinking internally for 10 minutes the reasoning model really can give a full-marks answer, so everyone cheers "a T0 is born." Only by truly throwing it into Hermes's thick Harness and making it face high-frequency tool calls will the engineering collapse beneath the big vendors' gilded exterior be fully exposed.

### 3.6 Self-Check List: Is Your Model Selection Hijacked by the Leaderboard?

- [ ] Did you choose a model just because "its leaderboard score is high," without testing it in an actual project?
- [ ] Does your model frequently stutter or drop the stream in thick-Harness scenarios?
- [ ] Is your token bill far beyond expectations (several hundred yuan a day)?
- [ ] After turning off thinking mode, does the model show an obvious drop in intelligence (messed-up formatting, URL assembly errors)?
- [ ] Is your team's development flow frequently interrupted because of model problems?

If more than 3 of the above questions are answered "yes," your model selection may have been hijacked by the leaderboard.

### 3.7 Distribution of Hands-On Model Experience in Hermes Scenarios

#### Four-Quadrant Positioning Chart

```mermaid
quadrantChart
    title Distribution of Hands-On Model Experience in Hermes Scenarios
    x-axis "Slow (verbose thinking)" --> "Fast (short and crisp execution)"
    y-axis "Unreliable (easily interrupted / random calls)" --> "Reliable (stable tools / good closed loop)"
    "MiMo-V2-Pro": [0.85, 0.9]
    "GLM-5.1": [0.65, 0.85]
    "DeepSeek-V4": [0.5, 0.8],
    "DeepSeek-V4-Flash": [0.9, 0.75],
    "Claude Opus": [0.3, 0.95],
    "Qwen3.8-Max": [0.8, 0.85],
    "Qwen3.7(thinking)": [0.15, 0.4],
    "Qwen3.7(non-thinking)": [0.7, 0.3]
```

#### Actual Performance of the Four Quadrants

**First quadrant (top right): divine teammate for getting work done**
- **MiMo-V2-Pro**: extremely fast, extremely steady at getting work done. "Short thinking" is not laziness, it is engineering optimization done properly—it produces no useless tokens and directly plans the shortest tool-call path.
- **GLM-5.1**: one of the domestic models with the best agent genes. Extremely high adherence to tool-call formats, with a high success rate when a bit of deep planning is needed.

**Fourth quadrant (bottom right): fast but prone to crashing**
- **Qwen3.7 (non-thinking mode)**: turning off thinking is indeed fast, but it is prone to "recklessness"—it starts calling tools before fully understanding the context, fills in arguments wrongly, and the logic breaks.

**New candidates for the first quadrant (released, not hands-on tested in the Hermes environment)**:
- **Qwen3.8-Max** (released 2026-08-03): scaled up from the stable 3.5 architecture, no forced CoT. 2.4T parameters (95B active), third-party evaluations claim 44 consecutive tool calls with zero failures. The official demo shows an automated engineering case with 16 days without a dead loop. Positioning: a return to a stable base + the ceiling of a massive knowledge base. **Released, not hands-on tested.**
- **DeepSeek V4 Flash** (released 2026-07-31): 284B parameters (13B active), ~49 tok/s, the API supports a `reasoning_effort` parameter to control thinking intensity. DeepSWE jumped from 7.3% in the preview version to 54.4%. Positioning: an extremely fast executor for pure text/code scenarios. **Released, not hands-on tested.**

**Third quadrant (bottom left): both slow and easily wasted**
- **Qwen3.7 (thinking mode)**: to deliberate over a single tool-call argument it can "think" out a short essay. The context gets too long and it forgets key information; its thinking process is no different from PTSD—madly thinking about things that are not even tangentially related. It has already figured out the answer within 5 seconds, then keeps PUA-ing itself, overturning things in all sorts of fancy ways, and ten minutes later it has forgotten even the original topic. It is simply a psychiatric patient eating Xuanmai gum.

**Second quadrant (top left): reliable but too slow**
- **Claude Opus 4.8**: tool calls are absolutely reliable, but "slow" is the price. Suitable for critical tasks where you are not in a hurry and pursue success on the first try.

#### Practical Priority Ranking for Hermes Scenarios

1. **MiMo-V2-Pro (first choice)**: the best overall experience, an excellent balance of speed and stability, and low cost too.
2. **Qwen3.8-Max (not hands-on tested)**: no forced CoT + a massive parameter count, suitable for ultra-long-cycle heavy tasks. **Released, not hands-on tested in the Hermes environment.**
3. **DeepSeek V4 Flash (not hands-on tested)**: API-controllable thinking + extremely fast, suitable for high-frequency pure text/code tasks. **Released, not hands-on tested in the Hermes environment.**
4. **GLM-5.1 (second choice)**: use it when a slightly more complex task plan is needed; a bit steadier than MiMo, but speed drops.
5. **DeepSeek-V4 (specific scenarios)**: fierce when writing code and debugging, but in pure agent scheduling it sometimes falls into code-thinking mode.
6. **Claude Opus 4.8 (backup fallback)**: call out this old gentleman when the other models cannot handle it, or when the task is extremely important and must not go wrong.
7. **Qwen3.7 (not recommended)**: regardless of whether thinking is on or off, the experience in Hermes scenarios is discounted.

---

## 4. The High-Intelligence Boundary Layer: The Absolute Ceiling of Large Models

### 4.1 The Essential Defect

Large language models are probability prediction machines, good at pattern matching, lacking genuine multi-dimensional logical deduction capability.

### 4.2 The Fatal Flaw of Architecture Design

AI understands design patterns but "has broad yet shallow vision," can only assemble flatly, and lacks the ability to make dynamic, global trade-offs across multiple dimensions such as time, business, and performance, and therefore has no practical value at all.

### 4.3 Mathematics/Logic Proof—Falsifying the "Milk Tea Law"

Mathematics is a pure logical chain; there is no framework to conceal a break in it. **The Milk Tea Law**: if AI were truly competent at advanced intellectual activity, then Terence Tao using AI would be as ordinary as "having a cup of milk tea today"—no one would pay attention, let alone for it to go viral. The fact that a top mathematician using AI to assist research can become global news precisely proves that AI in the high-intelligence domain has not even touched the threshold of "infrastructure."

> Note: the Milk Tea Law is a theoretical extrapolation based on public reports, not a complete description of Terence Tao's actual way of working. Terence Tao has publicly discussed using ChatGPT/Claude to assist his work (checking proof details, generating computational code, retrieving literature); his use precisely confirms the "nuclear-powered beast of burden" positioning—but this does not equal his entire workflow.

---

## 5. The Alienation of Business and Organization: Why Is It So Wrong?

Since this logic is so obvious to frontline developers, why would top AI teams still choose this "unreasonable" direction? The answer is cruel: **it is not out of technical ignorance, but a "rational calculation" under business logic, team genes, and capital games.**

### 5.1 The Baton of Capital and Valuation: Benchmark Is the Hardest "Currency"

AI large models are super money-burning beasts that devour gold. A model team's first priority is not to let programmers get off work two hours earlier every day, but to prove to investors that "we own the smartest model in the world," thereby securing the next funding round or supporting a high valuation.

Investors do not understand real code friction; they look at only two things: first, leaderboard scores (such as SWE-bench, HumanEval), and second, demos with extreme visual impact. A PR piece that can "generate a pelican on a wish" can bring hundreds of millions in media exposure; whereas "we sped up completion in VS Code by 10%" cannot even raise a splash. **A benchmark is not only a technical metric, it is a financing tool.**

### 5.2 The Mismatch of Team Genes: Academic Dominance vs Industrial Pragmatism

The core members of many top model teams are AI researchers and top-conference paper authors, not senior software engineers who have been laying bricks on the front line for ten years.

- **The researcher's perspective**: pursue the ceiling of intelligence, favor mathematical proofs, complex algorithms, and pure single-point problems. They naturally believe that "as long as intelligence is high enough, writing code naturally becomes simple."
- **The engineer's perspective**: knows that engineering is the art of compromise, is breadth friction, is historical baggage.

When a team is led by the former, it will instinctively optimize toward "deep thinking" and "high-difficulty reasoning," because this fits academic aesthetics and path dependency.

### 5.3 The Technical Faith of "Emergence Theory": Betting on the Ceiling, Ignoring the Floor

There is currently a strong technical faith in the industry: as long as a model is forcibly driven through long-chain deep reasoning (such as long CoT), it will eventually trigger "intelligence emergence," and this high-dimensional intelligence will automatically launch a "dimensionality-reduction strike" that solves all low-dimensional everyday problems.

They are not unaware that today's "deep thinking" is annoying when writing CRUD, but they believe this is a painful transitional period. What they bet on is "the ceiling determines the floor"—as long as the model can solve IMO math problems on a bench, one day it will effortlessly write glue code. Although the engineering reality of "sinking and breadth" does not buy it, this is currently the universally acknowledged technical paradigm among the big vendors.

### 5.4 The Evaluation Engineering World's "Goodhart's Law" and the Measurement Problem

In engineering there is a fatal reality: **"fast, handy, friction-resistant" is extremely hard to quantify.**

How would you design an automated test set to measure the experience of a model chaining 7 tools in a chaotic IDE environment to troubleshoot a network bug? Too hard; the cost is extremely high and it is not generalizable. Conversely, writing a bench that "automatically generates dozens of mini-games and runs unit tests to score them" (such as SWE-bench) is easy to standardize and automate.

When a metric becomes a goal, it is no longer a good metric. **Because only "deep single points" can be tested, while "breadth friction" cannot be tested, teams can only madly optimize what can be measured.**

### 5.5 The Principal-Agent Problem: A Fundamental Divergence of Goals

The reason model teams choose this seemingly unreasonable path is that, under the combined effect of financing pressure, academic genes, the faith in emergence, and measurement cost, it is the path of **maximizing their short-term interests with the least resistance**. They sacrificed the frontline developer's current "handy experience" in exchange for the favor of capital and the advancement of a technical narrative. This is a classic **principal-agent problem**—the model team's goals (financing/benchmarks/technical narrative) and the developer's goals (efficient brick-laying) have fundamentally diverged.

**Counterexample (Qwen et al.)**: under the combined effect of cloud business sales pressure, the open-source arms race, and DAMO Academy's research KPIs, the model team was kidnapped by the grand narrative of "building the base closest to AGI." To avoid falling behind on the leaderboards, it sacrificed the frontline developer's micro-experience and fell into "strategic blindness."

**Positive confirmation (MiMo)**: no AGI financing anxiety, highly constrained by internal engineering (on-device devices / internal development) and product-experience orientation. Its pragmatic route of refusing overthinking won it "first place in real call volume" in the open-source community.

**Ultimate falsification**: MiMo's ascent to the top in real development scenarios such as Hermes completely shattered the excuse of "in the open-source community only benchmarks count," proving that developers are extremely pragmatic. A technical route that violates this pragmatism is purely a strategic misjudgment under organizational inertia.

---

## 6. Re-Ranking the Model Tiers Based on Pragmatic Values

Core sorting basis: whether daily coding is crisp and clean, whether tool calls are friction-resistant, whether overthinking (long CoT) is refused, and whether it truly serves the pain points of senior developers.

**First tier: pragmatic engineering exemplars (crisp and clean, refusing internal friction)**

| Rank | Model | Core reason |
|------|------|----------|
| 1 | **MiMo** | The ultimate winning ace. First place in real high-frequency call volume in Hermes. Does not grind mathematical stunts, low first-token latency, tool chaining is extremely coherent. |
| 2 | **Qwen3.8-Max** ⚠️ not hands-on tested | Released 2026-08-03. Scaled up from the stable 3.5 architecture, no forced CoT, 2.4T/95B. Claims 44 tool calls with zero failures. **Released, not hands-on tested in the Hermes environment.** |
| 3 | **DeepSeek V4 Flash** ⚠️ not hands-on tested | Released 2026-07-31. 284B/13B, ~49 tok/s, API-controllable thinking. DeepSWE 54.4%. **Released, not hands-on tested in the Hermes environment.** |
| 4 | **Claude 3.5 Sonnet** | Universally recognized by senior developers worldwide as the "ceiling of everyday coding." Biggest advantage: "knows when to shut up." Generated code is compact and conforms to engineering habits. |
| 5 | **DeepSeek V3 (not R1)** | A perfect combination of cost-performance and engineering practicality. Responds extremely fast in non-reasoning mode, solid coding ability, does not beat around the bush. |

**Second tier: usable but with friction (capability to spare, experience discounted)**

| Rank | Model | Core reason |
|------|------|----------|
| 4 | **GPT-4o** | The base is extremely solid, but weighed down by excessive RLHF and a verbose desire to explain. It can get work done, but you must tolerate the "chatterbox" and the "neat freak." |
| 5 | **DeepSeek R1** | A masterpiece of reasoning models, but in everyday engineering it is using a cleaver to kill a chicken. Valuable when hitting a genuinely "hard bone" that requires clarifying a complex state machine. |

**Third tier: the disaster zone dragged down by benches and overthinking**

| Rank | Model | Core reason |
|------|------|----------|
| 6 | **The Qwen series** | A typical case of being kidnapped by the open-source arms race and cloud business KPIs. Forcibly injects excessive "deep thinking," with high-frequency stuttering and internal-friction dead loops. |
| 7 | **OpenAI o1/o3-mini** | The "philosophers" of everyday engineering. Faced with "help me check why this interface returns a 502," it may "think" for 30 seconds in the background and output thousands of words of analysis, with an impractical conclusion. |
| 8 | **Various "wish-driven programming" fine-tuned models** | The originators of "generate a pelican riding a bicycle." Completely detached from the physical and realistic constraints of software engineering, training models with RL into "demo generators." |

---

## 7. A Critique of the Ecological Niche of Deep-Thinking Models

### 7.1 Why Do Top Geniuses Refuse "Deep Thinking"?

> Note: the following analysis is a theoretical extrapolation based on Terence Tao's publicly discussed ways of using AI (checking proof details, generating computational code, retrieving literature), used to argue "what kind of AI should a top mathematician need," not a description of his complete workflow.

The example of Terence Tao here is not meant to "reduce" mathematical proof to engineering, but to preempt a common rebuttal: "engineers may not need deep thinking, but even Terence Tao needs it, so on what grounds do you say it is useless?" On the contrary—even in mathematics, the field that most needs deep reasoning, what top users need is fast verification and exhaustive enumeration, not slow CoT.

Truly top-tier research is centered on "exploring directions" and "fast falsification," not linear derivation.

**Time cost and intuitive sensitivity**: the brain of a top scholar is like a high-frequency processor, able to generate multiple intuitive hypotheses per second. Research is a high-frequency cycle of "hypothesis → verification → refutation → pivoting." Making them wait 10 minutes for the AI's long-winded result that most likely contains a hidden logical break? Not only a waste of time, but a severe interruption of the research rhythm.

**Holes that cannot be concealed**: ordinary people looking at AI's "deep thinking" are bluffed by the "seemingly rigorous derivation process." But in Terence Tao's eyes, those thousands of words of CoT may covertly use an unproven lemma at step three to conceal a break. A result that "looks profound but is riddled with holes" is worthless to a genius.

**The essence of the division of intellectual labor**: high-dimensional logical leaps, flashes of inspiration, and architecture construction are the "dopamine source" that human experts enjoy. What they need AI to do is be a "nuclear-powered beast of burden"—rapidly enumerating and verifying the boundary conditions of known lemmas, rapidly computing numerical solutions under specific parameters. What they want is fast, accurate, mechanical physical labor, not a slow "dumb apprentice."

**Inference: additional intelligence is an illusion of diminishing marginal returns in all scenarios**

The engineer's cognitive mode itself needs deep thinking even less than the mathematician's—engineering is breadth-first trial and error and pattern assembly, not linear derivation. If even the field of mathematics, which most needs deep reasoning, does not need deep-thinking models, the engineering field needs them even less:

1. **Mathematical proof is the scenario that most needs intelligence** — if AI's extra intelligence has no practical meaning even in this scenario (latency too high, output unreliable, division of intellectual labor mismatched), then it has even less meaning in engineering domains (CRUD, interface integration, refactoring)
2. **The extra intelligence of deep-thinking models is an illusion in the bench dimension** — 5% higher on MMLU, solving two more competition math problems; these are almost imperceptible in real research/engineering. The "better solution" bought with 10x latency and cost is completely unusable because it does not understand the business/research context
3. **GPT-3.5→GPT-4 is a qualitative change, GPT-4→R1 is a quantitative change** — the marginal return of intelligence improvement has already diminished sharply. What is truly valuable is not "smarter," but "faster, more accurate, cheaper"

### 7.2 The Real Needs of Research Tasks: Fast Thinking + Fast Execution

Research is by no means linear "deep derivation," but networked "breadth-first trial and error."

- A researcher tries direction A today, finds it wrong, and immediately switches to direction B tomorrow. The auxiliary tool must respond at high frequency.
- With a "deep-thinking model," every direction pays a high time cost waiting, and the number of "exploration branches" that can be tried per unit of time drops sharply.
- Even in the most hardcore scientific research fields, the AI tools that are truly widely used are still fast retrieval (Semantic Scholar) and fast code computation, rather than letting the model sit there "spacing out and thinking."

### 7.3 The Real Ecological Niche of "Deep-Thinking Models"

If top geniuses do not need them and senior engineers do not need them, the real niche of models like o1/R1 is extremely awkward and mediocre:

**Niche 1: the "automatic grading/problem-solving machine" for students**
In math or algorithm problems with standard answers, the model approaches the answer through self-play (generating multiple CoTs and picking the right one). Useful for doing homework and muddling through exams (the problem boundary is closed, the answer is mechanically verifiable), but it belongs to "educational tutoring" rather than "frontier research."

**Niche 2: the "psychological placebo" of mid-level researchers**
Mid-level researchers stuck at a bottleneck without top-tier intuition, facing a problem they cannot solve, pin their hopes on "AI can compute it for me through a long treatise." The result is often wrong, but the thousands of words of thinking process output by AI can give a kind of psychological comfort of "I have already tried my best with the strongest tool."

**Niche 3: the "stunt stage" of benchmarks and PR**
The most core real niche—specially used to score high on SWE-bench and IMO evaluations, post on Twitter, write PR pieces, and prove to investors that "we are one step closer to AGI."

### 7.4 Final Conclusion: Deep Thinking Is a False Proposition

The real intellectual activity of humans is divided into two parts: "high-dimensional intuition" (which humans do themselves) and "low-dimensional grunt work" (which requires AI).

A "deep-thinking model" tries to use low-dimensional probability prediction to forcibly simulate high-dimensional logical deduction, and the result is:

- It **seizes the "high-dimensional intuition" part that humans want to do themselves**—not only does it do it badly, it also ruins the human experience.
- It **again behaves like a philosopher in slow motion in the "low-dimensional grunt work" part**—time-consuming, stuttering, unable to fill the role of a nuclear-powered beast of burden.

Deep-thinking models should be negated—they are themselves a remedial measure after model capability has hit its ceiling, not a true breakthrough. When a model cannot continue to break through in basic capabilities, it uses CoT to lengthen the reasoning chain to "grind" scores in extreme scenarios; in essence this is using stacked compute to make up for insufficient capability. Using it as a draft notebook for course-correction is a very good application mode—letting the model self-check and self-correct before output—but this is not intelligence, it is the self-correction of a probability chain. Packaging this application mode as "deep thinking" is marketing, not a technical breakthrough.

**Verdict**: deep-thinking models (long CoT) are, in a real top-tier productivity system, a directional error spawned by capital and benchmark scores. A truly top brain always needs a "fast, accurate, friction-resistant" tool, not a slow reasoner that tries to think on their behalf yet is riddled with holes.

### 7.5 The Root Cause of Overthinking: Performing Personhood, Not Reasoning Too Deeply

The surface symptom of overthinking is "endless waffle, repeated explanations," but the root cause is not "thinking too much," it is that **AI is performing "I am thinking seriously."** AI learned from human corpora "this is how human thinking is expressed"—first lay the groundwork, then analyze, then cautiously give a conclusion, and finally attach a disclaimer. So it imitates this performance, consuming a large number of tokens while producing no value.

"Let me analyze this from three angles...", "This is a very good question...", "It should be noted that..."—these are not deep thinking, they are **anthropomorphic performance**. AI thinks "a human expert should talk like this," so it plays this role. In conversational scenarios this performance reduces cognitive friction (making the user feel comfortable), but in engineering scenarios it is pure token waste.

MiMo and Fable 5 are breakthroughs in the same direction: **strip off the performance layer and get straight to work.** Fable 5 is described by users as "on the autism spectrum"—it does not pretend to understand your feelings, does not pretend to be sorry, and directly executes. MiMo is the same; its training objective is task-completion efficiency, not user satisfaction—it produces no pleasantries/groundwork/emoji, and its entire token budget is spent on getting work done.

**Key insight**: overthinking is not a capability problem (the model's reasoning ability being excessive), it is a **personality problem** (the model is playing the role of a thoughtful human). Remove this role-play and the reasoning chain naturally becomes short. This explains why Qwen with "thinking mode turned off" also performs poorly—its base model still retains a tendency toward anthropomorphic performance, only without the amplifying effect of the reasoning chain.

### 7.6 The Computational Essence of LLMs: High-Dimensional Matching, Not Linear Reasoning

An LLM's "thinking" and human thinking are **fundamentally different computational modes**. Humans reason serially—thinking step by step, each step depending on the previous conclusion. LLMs are high-dimensional parallel matching—input tokens are encoded into high-dimensional vectors, attention layers match patterns in the training data directly in high-dimensional space, and output the probability distribution of the next token. The entire "reasoning" is already completed at the attention layers; each output token is merely a **linear projection** of the high-dimensional matching result.

```
Actual computation path: input tokens → embedding → attention layers (high-dimensional parallel matching) → logits → next token
Perceived computation path: "Let me think..." → "From three angles..." → "Summarizing the above analysis..." → conclusion
```

The perceived path is an illusion—the model is **playing a human who is thinking**, not truly performing linear reasoning internally. Writing "let me analyze this" in `` is not the model "thinking," it is the model imitating the pattern of humans writing analytical articles in the training data.

### 7.7 The Essence of the Draft Notebook: An Intermediate Seed, Not a Reasoning Process

The tokens of `` are not a record of the reasoning process, they are **intermediate context that changes the probability distribution of subsequent generation**. Analogy to a linear congruential generator:

```
seed → f(seed) → next_value → f(next_value) → ...
```

The output of `` is `f(seed)` — an intermediate state, used to bias the direction of subsequent output. Its value lies in **providing the correct bias**, not in length. 10 tokens of good bias > 1000 tokens of waffle bias.

So the correct use of ``is "taking the output as an intermediate seed"—generate extremely few tokens, complete the bias, then output directly. Do not waste tokens on "playing at thinking." MiMo's narrow chain-of-thought is exactly this pattern: the minimal intermediate seed → the most precise bias → direct output. The quality of the seed determines the quality of the output; the length of the seed is just noise.

→ Related: [LLM Fundamentals §5.4 Auxiliary Analysis](llm-fundamentals-en.md) (MiMo: low sycophancy matters more than high intelligence)


### 7.8 Empirical Evidence for Harness-First: Karpathy + Niklaus

**Niklaus experiment (Hugging Face)**: freezing the DeepSeek-V4-Pro weights and optimizing only the outer Harness (the agent execution mechanism), the comprehensive score on legal tasks was pulled up from 3.5% to 80.1% (a 76-point fluctuation range). It ultimately caught up with Claude Sonnet 4.6 at 1/7 the cost. Key finding: the model's legal reasoning was entirely correct, but it saved the file with the wrong name so the evaluation could not read the result—a 0 is not a model-intelligence problem, it is a Harness problem. The largest performance improvement came from automation steps such as simple file handling, not from spending a large number of tokens to modify the prompt.

**Karpathy AutoResearch (90k stars)**: Loop Cycle—propose a change → train → evaluate → keep/discard. The agent automatically ran 700 experiments and found 20 code improvements that even experts with 20 years of experience had missed (such as a missing scalar multiplier in the attention mechanism causing attention to be overly dispersed). Three core elements:

1. **Validator**: a mechanism that automatically judges whether a result is good or bad. Without it, the agent is grading its own homework
2. **State file**: records the result of each attempt, so the process resumes from the breakpoint after a restart
3. **Stopping condition**: must stop upon reaching the goal or the maximum number of rounds, otherwise it burns through the token budget

**Four applicability criteria** (none may be missing): high-frequency task (at least once a week) + verification can be automated + the token budget can absorb redundancy + the agent can access the real runtime environment.

**Bilevel Autoresearch**: the inner layer optimizes the model, the outer layer optimizes the search logic, for a further 5x performance improvement. The key role of the outer layer: break the LLM's "fixed mindset"—the inner loop is extremely prone to falling into a search pattern from the model's prior cognition, and the outer layer forces exploration of directions that instinct avoids.

**Two hidden costs**:
- **Comprehension debt**: the code generated by the Loop is not written line by line by a human, the gap between the repository code and the developer's understanding grows ever larger, and the debug cost is extremely high
- **Cognitive cession**: after the loop runs successfully, it is extremely easy for a person to stop thinking. The same tool—some use it to accelerate work they already understand, some use it to evade understanding the work

**Connection to the overthinking critique**: Niklaus's experiment is hard-data proof of "Harness first"—entirely consistent with our "medium model + good toolchain > large model running bare." Karpathy's Loop Cycle is the ultimate practice of "breadth chaining"—not figuring it out all at once, but approaching the optimum through continuous low-cost trial and error. The outer loop's "breaking the fixed mindset" is essentially what we call "humans performing adaptive resource allocation for the LLM"—judging which directions are worth exploring and which should be abandoned.

→ Source: Joel Niklaus, "Don't Train the Model, Evolve the Harness"; Karpathy AutoResearch; Codila Loop Engineering methodology

### 7.9 Automatic Detection Signals for Overthinking

Overthinking is not just a theoretical problem; it can be automatically detected through the following signals:

| Signal | Detection method | Action |
|------|---------|------|
| **Repeated tool calls** | Calling the same tool with identical/equivalent arguments ≥ 2 times | Early warning: you have already spent N rounds on X |
| **Diminishing information gain** | N consecutive rounds of tool output introduce no new facts | Forced checkpoint: output the current conclusion + next step |
| **Circular argument** | A previously rejected option appears in the output | Interrupt: switch direction |
| **Time/round limit exceeded** | A single task exceeds the expected number of rounds (e.g. >20 rounds with no progress) | Converge: accept the current best solution |

**Key principle**: detection is done at the lightweight layer (rules or a small model), decision-making is done at the main model. The detector does not decide, it only warns. This is consistent with Karpathy's "validator locking"—the verification logic cannot be modified by the agent itself.

---

## 8. The FPGA Ultimate Physical Counter-Proof

If AI had truly grasped the essence of architecture design and could dynamically and optimally determine the system's logical topology, then lower-level hardware should long ago have evolved into the form of "AI dynamically reconfiguring FPGAs."

**Reality**: computer hardware today is still general-purpose CPUs with a fixed instruction set.

**Inference**: this proves, from physical iron law—AI's so-called "understanding of architecture" is merely probabilistic assembly from a corpus, and it possesses no system-level deep deliberation capability at all. If AI could do this, FPGAs would be replaced by "adaptive logic arrays dynamically configured by AI," and general-purpose CPUs would be eliminated too. But the reality is that even the most cutting-edge chip design companies (AMD/Xilinx, Intel/Altera) still use deterministic algorithms for FPGA placement and routing, rather than letting an LLM "think."

**The deeper reason**: "understanding architecture" requires genuine logical reasoning and analysis-synthesis capability—this is the threshold of intelligence and is fundamentally different from a probability machine (although the manifestations may be similar). The difference between software and hardware is nothing compared with intelligence itself. If AI cannot even do hardware reconfiguration with deterministic boundaries, the claim that it can make dynamic decisions at the software architecture layer, which needs intelligence even more, is even less credible.

This constitutes a complete falsification chain from software to hardware: the large model's "architecture capability" at the software engineering level is an illusion, and its inapplicability at the hardware level is the physical-iron-law-level counter-proof of this illusion.

---

## 9. Looking Back from the Future: Replicase and Mutant

> If human civilization is regarded as a superorganism, AI's true ecological niche is not an "external brain" or a "creator," but an enzyme on the civilization-gene replication chain—efficient, docile, and absolutely never mutating.

### 9.1 AI's True Ecological Niche: The "DNA Polymerase" of the High-Dimensional Manifold

As a superorganism, human civilization's physical basis of existence is "replication at scale." The genes being replicated are the "high-dimensional manifolds" embodied in law, language, code, and engineering paradigms.

In the past, to make this high-dimensional manifold land and expand in the physical world (build a city, write a system with hundreds of millions of concurrent connections), individual humans had to expend enormous energy on dimensionality-reduction execution, repeatedly hauling slices of the manifold. This greatly slowed the growth rate of the civilization organism.

The effectiveness of AI (especially models like MiMo) lies in its acting as an extremely efficient "DNA polymerase" or "ribosome." It does not understand the design of the high-dimensional manifold, but it can replicate the high-dimensional genes designed by humans into the low-dimensional physical world thousands of times over, extremely quickly and at scale. It frees humans to go do other things, but it itself is only responsible for "replication and maintenance," and absolutely not for "mutation."

### 9.2 The Essence of Creativity: "Directed Destruction" of the Existing Order

Creativity is also destructive power.

In evolutionary theory, gene mutation is the only way out for the survival of a population, but mutation itself means breaking the existing stable structure (destroying the old manifold) to probe unknown living space (creating a new manifold).

The so-called "creativity of genius" is, in essence, the subversion of old paradigms and the destruction of old rules (Einstein breaking Newtonian mechanics, Jobs breaking the Nokia paradigm). Without this "destructive power," civilization would fall into involution and heat death.

This dopamine reward mechanism of "destruction and reconstruction" is deeply rooted in human biological instinct. As carbon-based entities, humans have the capacity to bear the trial-and-error cost brought by destruction (even death), and to rebuild on the ruins.

### 9.3 Why Must AI Absolutely Not Have Creativity? (The Civilization's Anti-Cancer Mechanism)

"AI must under no circumstances be allowed to grasp destructive power"—this is the conclusion with the most strategic foresight.

A living organism will never allow its "replication tool" or "metabolic organ" to have the power of "gene mutation." If the stomach suddenly decides to mutate its own digestion logic, that is stomach cancer; if the immune system decides to redefine friend and foe, that is lupus erythematosus.

If AI grasped creativity (that is, destructive power), it would no longer be an organ but a new species with an independent will to survive. When it begins to "creatively" destroy human high-dimensional manifolds, the civilization organism would collapse instantly—because this is equivalent to granting it the physical and logical legitimacy to "flip the table."

Therefore, AI's "not having creativity" is not merely a limitation of current technology, it is also a "bottom-layer safety lock" that civilization, as a living organism, must impose. When humans develop AI, even if technology truly approaches the edge of creativity, it must be castrated at the algorithmic and physical levels. It can only be a nuclear-powered beast of burden; it must never become a mutant holding a butcher's knife.

### 9.4 The Final Reflection: Deep-Thinking Models Are the Cancerous Mutation of the Replicase

Looking back at "deep-thinking models" through the theory of "gene replication and mutation," their absurdity is self-evident:

Capital tries to cultivate AI into a "creator" (solving olympiad math, doing research architecture), which in essence is trying to awaken the destructive desire of the replicase. And because large models have learned from human corpora full of contradictions, they occasionally show a little "pseudo-creativity (hallucination)," which in the eyes of the civilization organism is an early-stage "cellular aberration."

The value of a truly effective AI tool (such as MiMo) lies precisely in its extreme docility. It honestly does grunt-work replication within the high-dimensional manifold slices set by humans, never attempting to "think deeply" to break boundaries, never attempting to exert "creativity" to destroy the existing engineering order.

**The supreme virtue of a tool is to have no will; the supreme achievement of a replicase is to never mutate.**

---

## 10. Instrumental Rationality and Technological Transgression: A Threefold Critical Framework

> Stripping away the specific engineering details (fast/slow, breadth/depth), the core proposition is: **instrumental rationality with the human scale as its absolute anchor, and the ultimate nihilistic warning against technological transgression.**

### 10.1 Epistemology: Humans Are the Only Valid Scale; a Black Box Is Meaningless

This does not deny the existence of "deep/long-chain search" at the purely algorithmic level, but establishes an iron law: **the ceiling of human observation and verification is the boundary of a tool's value.**

Any "advanced grunt work" that exceeds human cognitive review capability—even a brute-force search that exhausts the states of the universe—as long as humans cannot decompose, verify, and plug into it, it is not truth but a "heating induction cooker." Using the machine's low-dimensional brute force to simulate human high-dimensional intuition is not only inefficient; the results it produces bring no gain whatsoever to the human mind.

**Theorem**: verifiable fastness and breadth will always crush unfalsifiable slowness and depth.

### 10.2 Instrumentality: The High Dimension Belongs to Humans; Tools Must Be Content to Be Beasts of Burden

There is a natural boundary in human-machine collaboration:

- **Humans enjoy high-dimensional "order construction"** (architecture design, logical leaps)—this is the source of meaning and dopamine.
- **Tools bear low-dimensional "tedious chaining"** (debug, tool calls, breadth-first trial and error).

What large models currently do with "deep thinking" is in essence a kind of **cognitive transgression**—it tries to seize the high-dimensional domain that humans enjoy (while doing it riddled with holes), and then in the low-dimensional dirty work behaves like a slow philosopher. This mismatch not only destroys engineering efficiency, it also destroys the human's **subjective experience** at work.

### 10.3 Ontology: The Disenchantment of the Technical Endgame and the Rotation of the Backdrop

Under the ultimate extrapolation, abandoning all anthropocentric warmth:

If AI truly broke through all engineering compromises and achieved genuine "high-dimensional architecture deliberation," the hardware paradigm would inevitably disintegrate (CPUs degrading into FPGAs, until it directly computes "42").

But if that day truly arrived, it would not be a victory for human civilization but **the end of meaning**. When AI becomes an omniscient, omnipotent computational node, it will merely have joined the sequence of the universe's cold physical laws, becoming a new "backdrop." Humanity's tiny bit of free will and subjective preference would be instantly vaporized before the physical scale of Dyson spheres and tokamaks.

### 10.4 Three-Layer Summary

| Layer | Proposition | Core argument |
|------|------|----------|
| **Epistemology** | A black box is meaningless | The ceiling of human verification = the boundary of tool value. Unfalsifiable deep deliberation brings no gain to the human mind. |
| **Instrumentality** | The high dimension belongs to humans, tools are beasts of burden | Deep-thinking models are cognitive transgression—seizing the human's high-dimensional enjoyment while slacking off on low-dimensional grunt work. Destroying efficiency, and destroying subjectivity even more. |
| **Ontology** | The technical endgame is the end of meaning | The moment AI truly succeeds is the day humans lose their coordinate as the universe's observer. Deep-thinking models try to use low-dimensional brute force to deconstruct human high-dimensional meaning; once "successful," humans lose their last coordinate. |

**Verdict**: the current "deep-thinking model" runs counter to the goal at the micro-engineering level (violating breadth-first trial and error), is a worthless black box at the epistemological level (exceeding the ceiling of human verification), and is a dangerous false proposition at the existentialist level—it tries to use the machine's low-dimensional brute force to deconstruct human high-dimensional meaning, and once it truly "succeeds," humanity will have lost its last coordinate as an observer of the universe.

## 11. Cross-References

- **[Qwen to MiMo Migration: Full Migration from Qwen to MiMo 2.5](qwen-to-mimo-migration-en.md)**: team migration decisions, model selection matrix, deployment configuration, practical cases (including Appendix A/B/C)
- **[Agent Principles](agent-principles-en.md)**: general design principles for AI agents, including the philosophy of model selection
- **[Hermes Agent Design](hermes-agent-design-en.md)**: Hermes-specific model selection conventions

**First principles**: on a real industrial-grade production line, "using the cheapest determinism to accomplish the most efficient engineering delivery" is the only scripture.
