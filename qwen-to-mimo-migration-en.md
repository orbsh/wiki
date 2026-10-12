# Technical Retrospective Memo: Full Migration from Qwen to MiMo 2.5

> **Languages:** [English](qwen-to-mimo-migration-en.md) (primary) · [中文](qwen-to-mimo-migration.md)

**Author**: Architecture Group / Core Development Team
**Audience**: All R&D members / Production business line leads
**Date**: 2026-06-20
**Status**: Team consensus

---

## 1. Why Are We Writing This Memo?

Recently the team has made deep use of Big Tech "strong reasoning" models represented by Qwen, both in internal development flow and in online production environments. After engineering validation in the real deep end, we found that the high IQ shown on paper leaderboards caused systemic problems when facing thick-Harness scenarios with high-frequency, multi-round tool interaction, and customer-facing scenarios with high concurrency and low latency.

The team has reached consensus: **a full migration from Qwen to MiMo 2.5 (standard edition)**.

Below is the underlying logic and data support for this migration; please follow it in subsequent development and module refactoring.

→ For the technical principles, see [LLM Overthinking Critique](llm-overthinking-critique-en.md)

---

## 2. The Essence of Large Models: High Vision, Low Intellect

Before discussing specific models, we need to align on one understanding first: **a large model is a probabilistic prediction engine, not a causal reasoner**.

- **High vision**: has read almost all the open-source code and API documentation in the world; its knowledge base is boundless
- **Low intellect**: lacks genuine causal reasoning ability. Faced with a specific codebase that has run for five years, full of ancient legacy muck and unique implicit business rules, it becomes useless because it cannot understand "humans' unique baggage of business history"

This means large models are not suited to architecture planning, for three reasons:

1. **Hidden context cannot be transmitted** — company budget, the team's stack proficiency, the direction of business evolution, departmental political games; none of this can be written into a prompt
2. **Lack of global abstraction ability** — AI sees code as "local fine-tuning", while a human architect sees code as a "three-dimensional topology"
3. **Isolation of responsibility** — architecture planning bears legal responsibility for production crashes; AI bears no consequences, so its proposals are inevitably sloppy

**The truth about real coding tasks**: 80% of day-to-day coding at Big Tech is manual labor—adding fields, wiring up interfaces, writing CRUD, configuring routes, tweaking CSS. It does not require extremely high IQ; the human architect points the direction, and the rest is highly repetitive mechanical alignment. This is precisely MiMo's best battlefield.

---

## 3. Problems with Qwen in Production

### 3.1 Root Cause: The Engineering Cost of an External Thinking Process

To farm points on public leaderboards (MMLU, Math, etc.), Big Tech models deeply bind instruction following, type safety, and logical ability into an external long thinking process (Thinking Process). This causes two extreme problems for the model in thick-Harness scenarios.

### 3.2 Problem One: Thinking On—Response Latency Is Unacceptable

**Development side (Zed/Hermes)**: in thick-Harness scenarios, Qwen starts a massive thinking router. Even for just an inline completion, it must reason and mutter in the background for a long time. Thinking tokens are re-prefilled in multi-round loops, causing the daily token bill to soar.

**Customer-facing production side**: when a customer sends a simple inquiry, the model responds extremely slowly; the time to first token (TTFT) is excessively high, and the user experience dies on the spot.

### 3.3 Problem Two: Thinking Off—Instruction Following Degrades

With thinking turned off, Qwen's logical ability drops sharply. Because part of its reasoning ability depends on the thinking process, once the thinking window is forcibly cut, it not only fails to understand the thick Harness's System Prompt, but even produces low-level hallucinations when handling basic business code—in actual customer-facing business, it cannot even correctly assemble a simple API URL, triggering production incidents of routing going astray online.

### 3.4 The Core Contradiction

| Mode | Problem | Impact |
|------|------|------|
| **Thinking On** | Extremely high TTFT, huge Token consumption | Development flow breaks, costs run out of control |
| **Thinking Off** | Degraded instruction following, chaotic formatting | Production incidents, user experience dies |

---

## 4. Why Did We Choose MiMo 2.5?

In tasks such as refactoring industrial-grade legacy projects, interface adaptation, and landing ADRs—tasks of "high vision, low intellect"—we don't need AI performing inner drama and making inventions every day. We only need it to be: **fast, stable, obedient, cheap, and never cutting out**.

### 4.1 MiMo 2.5's Core Advantages

| Dimension | MiMo 2.5 | Qwen Plus |
|------|----------|-----------|
| **Thinking mode** | Narrow CoT—constrained to tool-chain decomposition and boundary checks, outputs directly | Depends on an external thinking process; degrades severely once disabled |
| **Prefill compute** | 1/7 (Hybrid SWA architecture) | Full compute |
| **Streaming output** | No pauses, full-speed streaming jet | Frequently cuts out |
| **Format precision** | Native high precision | Chaotic formatting once thinking is off |
| **Cost** | Extremely low (480 lines of Markdown ≈ ¥0.2) | High (¥300-400/day) |
| **TTFT** | Millisecond level | Minute level |
| **Sycophancy** | None. The training objective is task-completion efficiency, not user satisfaction. It produces no pleasantries/setup/emoji; the token budget is spent entirely on doing the work | RLHF training explicitly rewards "sycophancy toward the user", producing large amounts of polite filler |

> **Note**: Low sycophancy matters more than high IQ. In interactive analysis scenarios, a model that panders to your views is more dangerous than one that is not smart enough—you will be led in the wrong direction without noticing. MiMo's low sycophancy is not "coldness"; it is the inevitable result of its training objective: optimize task-completion efficiency, not user satisfaction. In conversation scenarios users find it "unfriendly"; in engineering scenarios developers find it "not wasting words"—the same trait, judged in completely opposite ways in different scenarios. For details, see [LLM Fundamentals: Cognition and Mental Models §5.4](llm-fundamentals-en.md#54-auxiliary-analysis-rapidly-exploring-the-boundaries-of-thought).

### 4.2 Measured Data

Recently the team ran an extreme live drill: requiring AI, working from ADRs, to conjure from nothing and converge on a "Block-Level Editor System Design White Paper" covering Rust generics, Fjall embedded-storage Keyspace planning, and Yjs CRDT binary collaboration.

**MiMo 2.5's performance**:
- Full-speed streaming jet, with no thinking pauses or interruptions in between
- Steadily landed a 480-line, pure-substance Markdown document within 2 minutes
- A single run cost only ¥0.2

### 4.3 Cost Comparison (20-person team, 1 billion tokens per month)

| Option | Monthly cost | Multiple |
|------|--------|------|
| **MiMo v2.5** | **¥64** | 1x |
| MiMo 95% + DeepSeek 5% | ¥140 | 2x |
| DeepSeek Flash | ¥1,512 | 24x |
| DeepSeek Pro | ¥4,698 | 73x |

DeepSeek is priced separately for input/output (Flash: input $0.14/M + output $0.28/M), estimated on a 50/50 split. MiMo's blended unit price is ¥0.064/M.

→ For the model architecture, see [Appendix B.1](#b1-primary-model-mimo-25)

---

## 5. Team-Wide Migration Implementation Details

### 5.1 Internal Development Side (IDE / Agent)

- **Everyone check** the settings.json of your local Zed and your Hermes configuration files, and switch the model base to MiMo 2.5
- **Streamline prompts**: clean out the various defensive patches previously added to suppress Qwen's "long-windedness, filler, overthinking" ("DO NOT explain", "Output code ONLY", etc.). MiMo itself is already clean and does not need this redundancy

→ For the configuration guide, see [Appendix C.2](#c2-production-deployment-configuration)

### 5.2 Production Environment Side (Facing Ordinary Customers)

- All business interfaces with high frequency, low latency, and strongly structured responses (customer-service Agent, Webhook router, automated data dashboard, API assembly gateway) are fully replaced with MiMo 2.5
- Completely eliminate low-level failures in online business such as "minute-level latency" or "cannot even assemble the URL correctly"
- Leverage MiMo 2.5's extreme per-step TPS throughput to drive online customers' real-time interaction latency down into the millisecond range

### 5.3 Fallback Strategy (The 5% Extreme Hardcore Scenarios)

For the very few scenarios requiring deep reasoning (mathematical formulas, algorithm proofs):

- **Recommended**: GPT-OSS-20B + Low mode (locally deployed, with configurable reasoning effort)
- **Alternative**: keep a lightweight model in the gateway layer that supports manually controlling reasoning effort to Low
- **Strictly forbidden**: enabling Qwen's full thinking mode (a bottomless pit)
- **Principle**: prefer a slow but controllable low-thinking mode over a fast but uncontrollable full-thinking one

→ For the fallback strategy, see [Appendix B.3](#b3-gpt-oss-vs-mimo-25-benchmark-comparison)

---

## 6. Conclusion

On a real industrial-grade production line, "using the cheapest determinism to accomplish the most efficient engineering delivery" is the only gospel.

I hope this memo helps everyone recover the development flow destroyed by large models frequently freezing up, cutting out, and running up assassin bills, and that we use MiMo 2.5 to weld our engineering efficiency onto the highest-speed track.

During the migration of your respective modules, if you run into any questions about concurrency limits (Rate Limit) or fine-tuning Tool Call parameters, call us anytime in the architecture group channel.

→ For the cognitive gap between industry leaderboards and hands-on experience, see [LLM Overthinking Critique §3](llm-overthinking-critique-en.md)

---

## Cross References
- **[LLM Overthinking Critique](llm-overthinking-critique-en.md)**: A speculative analysis of LLM overthinking—engineering essence, capability ceiling, Bench mismatch, the false proposition of deep thinking
- **[Agent Principles](agent-principles-en.md)**: General design principles for AI Agents, including model-selection philosophy
- **[Hermes Agent Design](hermes-agent-design-en.md)**: Hermes-specific model-selection conventions

---

## Appendix A: The Technical Spectrum of Thinking Management

### A.1 Positioning of the Four Schools

```
[ Pure Exec, No Pad ] ── [ Controlled, Constrained ]  ── [ Deep Reasoning, Deep Pad ]
│                       │                                │
100% direct output       dynamic cognitive short-circuit   endless in-brain sparring
│                       │
│                  [ Narrow CoT ] ← MiMo 2.5
│                  constrained to a narrow problem domain: tool-chain decomposition + boundary checks
│                  streaming without interruptions, extremely low cost
```

| School | Characteristics | Representative models |
|------|------|----------|
| Pure execution flow | The draft notebook is removed entirely; the right to trial and error is handed to the external compiler for correction | Qwen3-Coder-Next, Gemma 4 (thinking off) |
| Controllable thinking flow | Limits the token budget; forcibly terminates ` thinking` at 95% confidence | GPT-OSS (Low/Mid/High), Qwen 3.7 (Low Effort) |
| Narrow CoT | MiMo 2.5: fine-tuned via MOPD + Agent RL, doing only tool-chain decomposition and boundary checks | MiMo 2.5 |
| Deep reasoning flow | Even for simple questions it deduces thousands of tokens of edge cases; severe latency under a thick Harness | DeepSeek-R1, Qwen3-Coder-Plus (full) |

### A.2 Why a Thick Harness Does Not Need Deep CoT

- **External correction is equivalent to in-brain deduction**: the LSP, the compiler (`cargo check`), and the test runner are 100% deterministic physical evaluation engines, giving the most precise answer in 10ms. For a reasoning model to spend 30 seconds in ` thinking` simulating "how the compiler sees this lifetime error" is pure wasted effort.
- **Overthinking destroys flow**: under thick-Harness long-context contamination, reasoning models are highly prone to triggering whiny self-reflection, producing extremely high TTFT.
- **A commercial compute disaster**: in low-difficulty scenarios a regular model's intuitive accuracy is already 95%+; making users wait blankly for thousands of hidden thinking tokens for a 0.5% gain is severe ROI over-engineering.

→ For the speculative analysis, see [LLM Overthinking Critique §1](llm-overthinking-critique-en.md#1-the-core-paradox-the-draft-notebooks-dual-personality)

---

## Appendix B: Model Selection Matrix

### B.1 Primary Model: MiMo 2.5

| Dimension | MiMo 2.5 |
|------|----------|
| **Architecture** | Hybrid SWA (hybrid sliding-window attention), native 128K context |
| **Thinking mode** | Narrow CoT—constrained to tool-chain decomposition and boundary checks, outputs directly |
| **Advantages** | 1/7 prefill compute, streaming output without pauses, high format precision |
| **Cost** | Extremely low (measured: 480 lines of Markdown ≈ ¥0.2) |
| **Suitable for** | Thick-Harness Agent, everyday coding, formatted output, API assembly |
| **Limitations** | Not suited to math/algorithm-proof scenarios requiring deep reasoning |

### B.2 Alternative Models

| Model | Architecture | Thinking mode | Suitable scenarios | Drawbacks |
|------|------|----------|----------|------|
| **GPT-OSS-20B** ⭐ | MoE, 3.6B active | Configurable (Low/Mid/High) | Thick-Harness Agent, MiMo replacement/surpassing | Requires local deployment (LM Studio/Ollama) |
| **GPT-OSS-120B** | MoE | Configurable | Complex reasoning, deep tasks | High resource requirements |
| **Qwen3-Coder-Next** | Linear MoE, 3B active | No CoT (native) | Zed/Agent core, native MCP alignment | New architecture; its reasoning ecosystem is still catching up |
| **Gemma 4 26B MoE** | MoE | Natively forced CoT | Thin-Harness single-shot hardcore tasks | **Freezes and cuts out under a thick Harness** |
| **North Mini Code** | Dense | Narrow | IDE real-time completion, lowest TTFT (0.31s) | Immature ecosystem |
| **Command A+** (Cohere) | Dense | Extremely reserved | Tool Calling, a standing background trooper | Smaller community |
| **Ministral 3 8B** | Dense | No CoT | Edge/intelligent customer service | Limited context window |
| **DeepSeek V4 Flash** | MoE | Configurable (Deep Thinking) | Hard algorithm assaults, deadlock analysis, concurrency optimization | High output unit price ($0.28/M); frequent 429 rate limiting on public cloud |
| **DeepSeek V4 Pro** | MoE | Deep Thinking | Extremely complex reasoning (not recommended for daily use) | Very high output unit price ($0.87/M); fabricates APIs 4% of the time |
| **GLM 5** | Hybrid architecture | Configurable | High-precision Function Calling, Git command execution, API integration | Public-cloud billing is on the high side; low community awareness |

### B.3 GPT-OSS vs MiMo 2.5 Benchmark Comparison

| Dimension | MiMo 2.5 | GPT-OSS-20B (Low) |
|------|----------|-------------------|
| **Thinking mode** | Narrow CoT (outputs directly after tool-chain decomposition) | Inner drama is cut out entirely in Low mode |
| **TTFT** | Extremely low | Extremely low (3.6B active) |
| **100+ round loops** | Occasionally talks nonsense in extreme scenarios | Code closure correctness and Tool format are extremely precise |
| **System Prompt tolerance** | High | Higher (top-tier OpenAI RL alignment) |
| **Deployment** | Cloud API | Local (LM Studio/Ollama) |

**Conclusion**: GPT-OSS-20B + Low mode = the perfect custom-made tool for the thick Harness, able to replace and even surpass MiMo.

### B.4 Gemma 4 Pitfall-Avoidance Guide

**Advantages**: Chunk Prefill optimization is extremely strong; ultra-long context handling speed is second-scale (on par with MiMo).

**Fatal flaws**:
- **Natively forced CoT**: strong reasoning logic is hard-coded at the architectural level; thinking cannot be turned off manually
- **Freezes under a thick Harness**: automatically launches thousands of tokens of multimodal reasoning, self-reflection death loop → reasoning backlog → slower and slower → stream cut off

**Suitable scenarios**: thin-Harness single-shot hardcore tasks (chart UI, multimodal code merging); **not suitable for high-frequency real-time completion**.

### B.5 Selection Decision Tree

```
What is your core engineering pain point?
│
┌────────────────────────┴────────────────────────┐
▼                                                 ▼
【For End Users】                            【For Developers/Agent】
(Support/text polish/high-QPS, low-cost)      (Zed code completion/complex API orchestration/LSP correction)
│                                                 │
├──────────────────┐                              ├────────────────────────┐
▼                  ▼                              ▼                        ▼
[Max Throughput]    [Low-Resource/Edge]      [Latest Arch/Tools]      [Most Mature Ecosystem]
North Mini Code     Ministral 3 8B            GPT-OSS-20B (Low)      Qwen2.5-Coder-32B
(min TTFT/0.31s)    (Thinking explicitly off) (config./Harness gem)   (classic dense/128K)
```

### B.6 Selection Principles

1. **Speed over intellect**: in thick-Harness scenarios, the external toolchain is a 100% deterministic evaluation engine; the model does not need to simulate the compiler in its head
2. **Determinism over reasoning depth**: precise formatting and stable output matter more than "thinking one step further"
3. **Controllable cost**: daily Token consumption must be at the single-digit RMB level
4. **Streaming without interruption**: TTFT < 100ms, no pauses in between

---

## Appendix C: Deployment in Practice

### C.1 Iron Rules for Taming a Thick Harness

**A. Physical-layer short-circuit control (Hermes / OpenClaw)**

In any agent framework that supports passing through an `extra_body` request body, never let the draft notebook grow wild:

```json
"thinking_budget_tokens": 512
```

Forcibly shrink the draft notebook's footprint, downgrade its purpose from "code compilation simulation" to "tool-chain sequence planning", and shut down the inner drama the moment it is done.

**B. Psychological defensive-prompt removal (Zed, no extra_body)**

The Zed architecture strictly forbids `extra_body`; psychological intervention through the System Prompt is required:

```
[Draft Notebook Protocol: Externally Verified Architecture]
1. You are currently running in the Zed IDE with active LSP and a background 'cargo check' auto-compilation stream.
2. It is strictly forbidden to use the  space to statically simulate compilation or check complex variable types.
   The external physical compiler will take over and handle these errors.
3. Please treat the  block purely as a temporary, minimal Checklist, used only to list the API or refactoring steps to output next.
4. Keep the draft notebook output strictly within 5 lines, and transition seamlessly to the final code output within 2 seconds.
```

### C.2 Production Deployment Configuration

**MiMo 2.5**

```json
{
  "model": "mimo-v2.5",
  "messages": [
    {"role": "system", "content": "You are a direct execution agent."},
    {"role": "user", "content": "..."}
  ],
  "temperature": 0.1
}
```

**vLLM local inference**

```bash
# MiMo 2.5 local deployment
vllm serve xiaomi/mimo-v2.5 --max-model-len 131072

# Qwen 3.7 Plus low-thinking mode (alternative)
vllm serve qwen/Qwen3-32B --thinking-tokens 1024 --max-model-len 131072
```

**Harness-side hardening**

- **Dynamic context trimming before input**: avoid noise interfering with the model's core reasoning path
- **Structured validation after output**: Pydantic/LSP checks, using deterministic code to guarantee delivery quality

### C.3 Case Study: Refactoring the Business-Side ID-to-Link Generation Architecture

**Problem**: the business side generates links from IDs (`PROD_992147` → `https://example.com/item/992147`). A full CoT is too slow, and with CoT off the link format is frequently corrupted (hallucinations of sensitive symbols such as `?&=%`).

**Root cause**: a URL contains many sensitive symbols (`?`, `&`, `=`, `%`). When an autoregressive model outputs directly, it lacks a global buffer and is highly prone to symbol hallucination. The draft notebook lets the model complete a self-check in its head—"extract prefix → match route → prepare concatenation".

**Pain points of the old architecture**:

```
【Manual Two-Stage Pipeline】
Aliyun API ───(spits out a long chunk of preprocessed text)───> local backend ───(runs heavy regex parsing)───> extract URL
```

- Malignant network I/O loss: the model's in-brain deduction happens in the cloud, so hundreds of tokens of filler are forcibly downloaded in full over the public network and then cleaned up with regex
- Format fragility: if the model's preprocessed text changes its sentence pattern even slightly, the backend regex immediately fails → occasional 500s

**New architecture**: MiMo 2.5 + strict JSON Schema control

```
MiMo 2.5 [Narrow CoT direct output] ──> clean JSON (zero regex, millisecond response)
```

Three major optimizations:
1. **Narrow CoT**: completes in an extremely short time, TTFT at the millisecond level
2. **Format determinism**: strict control via `response_format: { "type": "json_object" }`, output 100% perfect
3. **A cost cliff**: per-request cost drops 50-100x

| Evaluation dimension | Historical solution | After the refactor | Perceived effect |
|---------|---------|--------|------|
| **Network I/O** | ≥ 2 times | 1 time | Greatly reduces the risk of timeout disconnections |
| **TTFT** | Extremely high | Millisecond level | The front end "opens instantly" |
| **Maintainability** | Fragile (regex 500s) | Extremely high (Schema) | Eliminates legacy bugs |
| **Cost** | ¥300-400/day | A few yuan/day | Reduced 50-100x |
