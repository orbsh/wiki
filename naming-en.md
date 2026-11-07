# On Naming

Naming is not packaging; it is part of the model. A wrong name puts a wrong model in the reader's head, and nothing raises an error. This document records three things: what a split vocabulary costs, why names drift, and what can be extracted from existing examples.

## 1. The cost of a split vocabulary

Code and documentation are two carriers of one model. When the document says A and the code says B, the model exists twice, and the reader has to maintain the concordance himself; that table enters no repository and no one's long-term memory, so it is rebuilt every time.

AI is worst off: it has no memory across sessions and rebuilds its understanding from the words every time it arrives. When one word names two concepts it re-guesses on every encounter, and the cost of a wrong guess lands in the implementation, where people notice it only much later.

For people the cost is mental switching. On reading the word they must first decide which one it means this time, and attention moves off the problem onto the vocabulary. The overhead is invisible, but it is paid every time, and it grows linearly with the length of the document.

The cost compounds. Writing the ambiguous word is cheapest at the moment of writing; every later reading, every discussion, every new colleague or model that joins pays again — while a single refactor is paid once.

Hence: ambiguity that one refactor can remove should not be kept alive with long-lived comments, and it should not be smoothed over with modifiers. "Resident session" only hides the ambiguity; the bare word still names two things in two places.

## 2. Why names drift

Concepts drift; names do not move by themselves. After the architecture evolves, what sits under a name has changed: the granularity changed (from "a whole service" to "an instance inside a service"), the implementation changed (from a subprocess to an in-process VM), the role changed (from "one conversation" to "a reusable execution context").

After the drift the name is not wrong — it is imprecise. Sentences still parse; the meaning has shifted. This is the hardest kind of error to spot, because nothing raises an error.

One real trajectory: in aura and effector, session went through three stages. On 2026-09-16 it first meant a nushell PTY shell session (the Unix sense — that PTY subprocess really was a process); the same day it was generalised to mean the resident context of every carrier (in-process VM, in-process module, subprocess, side by side), while krystallizer had always used it for a conversation. Three concepts, one word, and one clarification before every discussion.

The test is not whether the name still looks right, but whether its boundary still holds. Once a name needs explaining, the debt has already been taken on.

## 3. Examples

### actor and process: same core, different boundaries

Their intension is close: each is an independent unit of execution, carrying its own state, being scheduled, sharing no memory with its peers.

The difference is decisive: an actor has a mailbox (asynchronous messages), processes messages one at a time, and is location-transparent — the caller does not know where it is, and does not need to.

So instead of coining a modifier such as "asynchronous process", the field took an entirely different name. The difference between the names carries the difference between the models: seeing actor, you know the way to talk to it is to send a message, not to call it.

### pod and container: the image says half a sentence more

Kubernetes writes the etymology into the definition itself: "A Pod (as in a pod of whales or pea pod) is a group of one or more containers, with shared storage and network resources", and "always co-located and co-scheduled".

In runtime intension a pod is roughly a group of containers; but the word adds the image of peas in a pod — several beans under one shell, sharing it, scheduled together.

So one word says two things: it contains several (structure), and it cannot be subdivided (scheduling granularity). "Container group" would say only the first, and nothing about "this is the smallest deployable unit".

### One exercise from this stack: session → executor

It started with an underspecified word: the same session named three different things on three layers (a conversation, a resident runtime, a framed channel).

There was a detour: rename it residency / resident. Rejected, because that names a lifetime — an optimisation (saving activation cost), not the essence. The essence is "executing atomic logic transparently".

process was considered too, and rejected for a different reason: within this stack the word is already used literally (out-of-process subprocesses), and using a literally-occupied word as an abstraction repeats the original mistake elsewhere.

The choice was executor, separated from effector by one relation: effector is the execution plane (frame encoding, nodes dialling in, the sandbox); executor is a unit of execution inside that plane (the living body of one instance). The former has N of the latter; not the reverse.

The price: five repositories, five hundred-odd code hits, one wire field, and matching edits in documents and ADRs. The return: no later reading needs a disambiguation step.

## 4. Rules

Four principles set the direction; five operational rules apply them.

### 1. Image over abstraction

An abstract word expresses too much: it makes sense in every context, so the context supplies the boundary. In conversation the context is strong and the word is legible; the technical field is exactly the cross-context case — an application is itself an assembly of results from several fields, and the same word means a different thing in each. session in this project is that case: it means the resident runtime in aura and effector, a conversation in krystallizer, and the framed channel in induction. Each reading is fine on its own; together they are a disaster.

That is why the field reaches for borrowed words with a concrete image: pod instead of "container group" — the peas in a pod; actor instead of "asynchronous execution unit" — a performer on a stage. An image carries its own boundary, needs no context, and asks the reader to learn no jargon first.

### 2. Intension over extension

Intension is what it is; extension is which cases it covers. Name by extension and the name follows the use: one component runs conversations today and tools tomorrow, and the name drifts with the feature. Name by intension and a change of use needs no rename. In practice: the name says what it is, not how it is optimised, and not which mechanism it takes part in — residency named an optimisation (saving activation cost), session named "the stretch of time one interaction lasted"; both describe a use, so neither is stable.

This principle leans toward cohesion: one name carries only its own core.

### 3. Orthogonality (decoupling)

One concept, one word; one word, one concept: two names' extensions neither overlap nor contain one another. When names share no stretch of meaning, the reader need not decide which one is meant. Modifiers do not carry the job of distinguishing — a qualifier that cuts through the overlap only hides it; the bare word still names two things in two places.

Principle 2 outranks principle 3, as cohesion outranks decoupling in architecture: first tighten each name to its own core, then discuss whether the names overlap. Cutting a concept in two in order to remove an overlap yields two synonyms, not orthogonality — the reader converts between two words to recover the one concept, and the disambiguation cost has merely moved. A name whose core is unclear will overlap however it is divided.

### 4. Affinity

The first sieve for a technical word is whether it can be said and remembered; precision comes after. actor and pod spread so well half because of the image and half because of the length: one or two syllables, already in everyday English, no compound term to learn first. The counter-example is asynchronous process — three roots, nine syllables, and one still reaches for the abbreviation async to write it comfortably; and once async is common, asynchronous process, async fn and "async function" start blurring into one another, so the reader resolves them from context.

Nor can an image be invented by feel. Suppose actor had been named "blue thread": on its own that is not absurd (blue is not arbitrary in blue-green deployment), but it collides head-on with the existing green thread, and that family has no basis in meaning for its members — green just about reads as "light, eco-friendly", blue reads as nothing at all. With two of them memory still works; with more there is nothing to tell them apart, because the difference is not in the words, and the reader can only memorise — when the whole point of the name was to spare him that.

So an image must be readable from the concept itself (the peas in a pod read out co-located and indivisible); an image that is not readable is only a code name, and a code name should be short — and should not come in families.

### Operational rules

1. The condition for a new word: it shares a core with the old one but has a different boundary, and the decisive property can be stated (the actor's mailbox, the pod's peas). If it cannot be stated, it is the same concept, and the old word should be reused.
2. A new word must add an image. When two words have nearly the same intension they are not two concepts but a duplicate name — either merge them, or establish that they sit on two layers that never swap (effector / executor is of this kind: "plane versus unit" is what makes it hold).
3. Run the collision check in the reader's context: near-synonyms appearing side by side in one module have the highest priority; homographs that live in different repositories or layers and never share a screen have the lowest. Words coined one after another in a single family are the most dangerous — green, blue and red threads collide not as strings but as the distinctions the reader must infer.
4. Pay for the rename up front. Locate the concept first (which layer, who owns it), then pick the word; once picked, change identifiers, wire fields and document titles in one pass, leaving no dual-write period. A rename is paid once; disambiguation is paid forever.
5. Prefer a primitive the domain already has. induction calls the framed channel a link, and the name carries the constraint "a line between two ends" without further explanation.

This document is about choosing words; "one concept, one word" is a charter-level constraint, and it is written in Fractal's coding discipline.