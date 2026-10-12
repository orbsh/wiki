# Workplace Communication: IM, Email and the AI Moderator

> **Languages:** [English](workplace-communication-en.md) (primary) · [中文](workplace-communication.md)
> Created 2026-06-29

**Core thesis**: The essential problem of IM is that it synchronizes communication that could be asynchronous, requiring everyone to be present in real time — in essence an inefficient meeting. Communication efficiency does not depend on the tool but on the quality of information organization — reduce RTT (round-trip time), give enough information at once, reduce ritual and performance. The decision layer needs deep thinking; the execution layer is handled by AI.

---

## I. The Structural Problems of IM

### The Essence of Synchronization

IM turns communication that could be asynchronous into synchronous: it requires everyone to be present, much like a meeting. If someone is absent, it cannot proceed — extremely inefficient.

Typical scenario: "Are you there?" — if the person is away, you wait for them to come back before saying anything; but by the time they come back, you may have forgotten many details, or even forgotten what you were going to do.

### Cascading Interruption Storm

IM's real-time interruptions efficiently destroy rhythm and reduce overall efficiency. A phone call can serve as a stronger signal, but the other party may need to focus — IM's real-time interruptions destroy focus, and interruptions are themselves cascading: you interrupt me, I interrupt him, and the whole team's deep-thinking time is fragmented.

### Root Causes

| Root cause | Manifestation |
|:---|:---|
| **Insufficient thinking ability**, or the execution layer does not need to think | Instant replies substitute for deep thinking |
| **Hierarchy** | Interruptions are often one-directional — superiors interrupt subordinates, not mutually |
| **Deliberation is not encouraged or respected** | Reply speed is treated as attitude; depth of thought is not rewarded |

### Emergency Coordination Does Not Count as "Communication"

Fire alarms, failure alerts, and the like are the socialized form of triggers — one-way notification, not mutual alignment toward consensus. They should not fall into the category of "communication." The definition of communication is: the process by which two or more parties mutually align and reach consensus. Emergency notification is trigger-response, not communication.

## II. Correcting Common Attributions

| Mistaken attribution | Why it is not the main cause |
|:---|:---|
| "The mobile leap" | Forum-style discussion was briefly dominant only in the early days when there was no alternative; it is not the reason IM became mainstream |
| "Sense of distance" | Hierarchical societies prefer distance, but authority comes from position, not thinking ability; reply speed and tone convey "attitude" more easily than content |
| "Preference for colloquial culture" | Take "approving by written note," for example: it is essentially a matter of quality — evading deep thinking — not a cultural choice |
| "Group collaboration is efficient" | Real-time communication is more prone to performance and ritual (a screen full of +1). Forum-style defaults to approval, raising objections only when there are any; no performance needed |

### Tools Are Culture

Tools do not merely transmit information; they also shape the organization's way of thinking. The CLI gave rise to Unix culture (composition, transparency, programmability), and the GUI gave rise to Windows culture (encapsulation, graphics). vim/emacs users and IDE users have different modes of thinking. As a daily work tool, IM's cultural shaping power cannot be ignored — it is not just a communication channel; it also shapes the organization's expectations of "communication" itself.

## III. Reduce RTT, Give Enough Information at Once

The engineering core of communication efficiency: fewer round trips = less total time.

High-quality communication = a logical chain composed of evidence/hypotheses + reasoning. Give the background, the phenomenon, the thought process, and the code all at once, saving all the round trips of "What did you use?" "What exactly does it look like?" "What have you tried?"

**Reduce ritual and performance**: a screen full of +1, "acknowledged," "okay" increase RTT without adding information. Forum-style discussion defaults to approval, raising objections only when there are any; no need to perform agreement.

## IV. Structured Thinking vs. Rapid Dumping

Zhang Xinxu observed that "writing the problem clearly solves it faster than chatting," but his attribution was based on a framework popular at the time — Kahneman's "Thinking, Fast and Slow" (a 2011 bestseller).

This attribution is mistaken. The difference is not in speed but in organization:

- **Structured thinking**: long total time, but active thinking and high reasoning density — fast in internal processing, slow in output organization
- **Rapid dumping**: fast output (replies within seconds), but shallow internal processing, with no organization or verification

| | RTT | Total time | Internal processing |
|---|---|---|---|
| Structured thinking | High (reply within a day) | Low (15 minutes of organizing + 1 round of RTT) | Deep, active |
| Rapid dumping | Low (reply within seconds) | High (30 minutes gone by) | Shallow, fragmented |

**The connection to the LLM capability structure**: fragmented processing is, conversely, a comparative advantage of AI. Suppose an LLM's fragmented-processing capability is 1/10 of its structured processing, while a human's fragmented-processing capability is 1/1000 of their structured processing — relatively speaking, AI retains its capability in fragmented scenarios far better than humans do. The LLM workflow itself is proof: submit the user's fragmented conversation all together and quickly judge the user's needs. The AI moderator exploits precisely this advantage, organizing IM's fragmented conversations into structured summaries and bridging the gap between "rapid dumping" and "structured thinking."

## V. Decision Layer vs. Execution Layer

The premise for discussing communication problems is the **decision layer** — requirements organization, architecture decisions, product direction; these require deep thinking. The decision layer should think deeply and should not be interrupted by IM's synchronization.

The execution layer's work is done by AI — code changes, testing, format conversion, document organization. AI excels at mechanical work within the normal-distribution range and is poor at judgment work in the power-law tail.

**Division of labor**:
- Decision layer: deep thinking, using structured thinking, with the AI moderator assisting information organization
- Execution layer: AI completes the mechanical work; humans only do acceptance

## VI. The AI Moderator: A Reconciliation for the Synchronization Problem

Since IM chaos is a reality and the volume of communication cannot be reduced (10 decision-makers = 45 channels), then:

### Three-Layer Architecture

| Layer | Responsibility | Tool |
|:---|:---|:---|
| **AI moderator** | Organize chaotic conversations, flag conflicts, synchronize consensus anchors | IM channel + continuous AI analysis |
| **Loop execution** | Code changes, testing, PR, CI/CD | Gitea + CI + containers |
| **Human judgment** | Make the final call on requirements, review PRs, accept results | Channel discussion + PR review |

The value of the AI moderator is not "reducing communication" but **countering the inefficiency of synchronization**:
- **Organize fragmented conversations into structured summaries**: force everyone into "structured thinking" mode
- **Reduce information asymmetry**: everyone sees the AI's organized results; no need to sync via "who attended which meeting"
- **Reduce repetitive communication**: consensus anchors are continuously updated; no need for circular rehashing
- **Flag disagreements**: when AI finds a conflict of opinions it pulls it directly onto the table; no need for private back-and-forth
- **Counter ritualization**: AI tracks "what was said" rather than "who replied with +1"

### The Solution to the "Equal Treatment" Problem

LLMs treat all utterances equally, but in actual conversations different utterances carry different weight:

| Solution | Applicable scenario |
|:---|:---|
| **Time decay** | More recent utterances carry higher weight (requirements evolve) |
| **Consensus detection** | Multiple people say similar things → high confidence; only one person says it → low confidence |
| **Conflict detection** | A says "be fast," B says "be stable" → flag in red for the relevant people to confirm |
| **Speaker-level ranking** | Decision-makers' utterances carry higher weight |
| **Confirmation vote** | For key decisions, initiate a vote among the relevant people to confirm |

## VII. The Relationship to the LLM Cognitive Framework

| Communication problem | Corresponding LLM limitation | AI moderator's solution |
|:---|:---|:---|
| Fragmented conversation cannot extract requirements | Egalitarianism (treating all tokens alike) | Time decay + consensus detection |
| Missing implicit requirements | High vision, low intelligence (can recognize patterns but lacks judgment) | Confidence annotation + human confirmation |
| Circular rehashing wastes time | Semi-mechanical (fast but not rigorous) | Consensus anchor updates |
| Tracking requirement evolution | No metacognition (cannot judge "this requirement has changed") | Conflict detection + initiate confirmation |
| Ritualized replies (+1, acknowledged) | — | AI tracks content rather than reply behavior |

---

## Cross-References

| Document | Related argument |
|:---|:---|
| [LLM Fundamentals](llm-fundamentals-en.md) | Three kinds of intelligence, adaptive allocation, absence of metacognition |
| [Loop Engineering Analysis](loop-engineering-analysis-en.md) | The three-layer architecture of AI moderator + loop execution |
| [Karpathy AI Coding Methodology](karpathy-ai-coding-methodology-en.md) | Structured thinking vs. rapid dumping, structural constraints |

## References

- [Zhang Xinxu, "How to Ask Questions to Advance into a Frontend Master?"](https://www.zhangxinxu.com/wordpress/2015/05/how-to-ask-web-front-question/): the starting point of this article's discussion. From the angle of email questions, Zhang Xinxu proposed the principles of "reduce RTT, give enough information at once, reduce ritual"; this article extends them into a structural analysis of workplace communication.