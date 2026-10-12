# Automaton, State Machine and Reducer: Two Registers of the Same Term

> **Languages:** [English](automaton-vs-state-machine-en.md) (primary) · [中文](automaton-vs-state-machine.md)

## In One Sentence

**An automaton and a "state machine" are formally the same object; but in the engineering context of "state machine pattern / library names", "state machine" carries a narrower meaning, referring to the kind that is "an entity's own named states + operations restricted by state".** The reducer of a replicated state machine (RSM) belongs to the former, not the latter; usages of the "process / multi-entity" kind such as agent orchestration are workflow graphs, not state machines.

## Formal Register: Automaton = State Machine

The same mathematical object — a deterministic automaton `M = (Q, Σ, δ, q0, F)`:

- `Q` — the set of states (can be infinite; see pushdown automata, Turing machines).
- `Σ` — the input alphabet.
- `δ` — the transition function `δ : Q × Σ → Q`.
- `q0` — the initial state.
- `F` — the set of accepting states.

"Automaton" is the **computability perspective** (what language it recognizes, what it computes); "state machine" is the **business perspective** (modeling the state transitions of a domain). The two are two names for the same object, and `δ : Q × Σ → Q` is exactly the **reducer** in programming.

`Q` can be an infinite set (the stack of a pushdown automaton and the configurations of a Turing machine are both infinite states), so "states have no clear boundary, operations are unrestricted" does not disqualify it from being an automaton.

## Engineering Register: "State Machine" Refers Only to the (Single-Entity) Restricted-Operation Kind

In the "state machine pattern" (State pattern) and state machine libraries (xstate, spring-statemachine, etc.), the term "state machine" has crystallized into a narrower, more practical meaning:

- The multiple modes of residence / states of **a single subject (entity)**;
- Each state defines **which operations are legal** (transition guards);
- Triggering an operation jumps to another named state — **state transition + restricted operations**.

This is a **restricted subclass** of the automaton (workflow-style FSM / labeled transition system), and it is also where the term "state machine" truly carries information in engineering. It must be **single-entity** — the cross-entity "who is doing what" is not a state machine; see the next section, "State Machine vs. Workflow Graph".

## State Machine vs. Workflow Graph: Single Entity and Process

"Restricted operations" is not yet the complete criterion — the deeper axis is **single entity vs. process**:

- **Center**
  - State machine: the multiple modes of residence / states of a single entity.
  - Workflow graph: the step order of a process / flow / routing.
- **Node semantics**
  - State machine: the modes of the same object (order: pending payment / paid / canceled).
  - Workflow graph: steps / roles / components (who is doing what).
- **Transition**
  - State machine: the state transition of an entity, preserving that entity's identity.
  - Workflow graph: the flow of steps, which can span multiple entities / participants.
- **Typical**
  - State machine: protocols, orders, booth lifecycles.
  - Workflow graph: BPMN / petri-net / orchestration graphs.

The states of a state machine are the modes of residence of **the same thing**; the nodes of a workflow graph are **steps / workers**, not the "states" of the same thing — so a workflow graph is inherently not a state machine, and this distinction has nothing to do with "restricted operations" (formally it corresponds to FSM vs. workflow-net / petri-net).

**AI agent orchestration** (LangGraph `StateGraph`, OpenAI/Anthropic workflows) is a typical **workflow graph** — nodes = LLM calls / tools / sub-agents, edges = routing, spanning steps and even multiple entities, not a "single-entity state machine". What can truly be called a "state machine" is the **lifecycle of a single agent itself** of the kind found inside a ReAct loop or tool call (think → act → observe).

## The Vacuousness Test: Why Not Everything Can Be a "State Machine"

If "state machine" is used in a general sense — a web application is also called a state machine (interfaces = operations, database = state), an orchestration graph is also called a state machine — then almost everything qualifies, and the term loses its discriminating power as a result. This is the decisive vacuousness test: **"state machine" should be reserved for the "single-entity, restricted-operation" kind**; use **workflow graph** for processes / multiple entities, and use **reducer** for the general `fold commands into a large data state`.

## The Position of RSM (Replicated State Machine)

The `apply` (reducer) of a replicated state machine is the transition function δ of a (large-state-space, possibly infinite) deterministic automaton — **it is formally an automaton**. But it does **not** have the features of a "state machine" in the engineering register: no named states, no transition matrix of "which operations this state can perform", no jumping between discrete states. Therefore, in the engineering register, the accurate word to describe it is **reducer**.

Looking at the four concepts by dimension (automaton / engineering "state machine" / workflow graph / reducer):

- **State**
  - Automaton: an arbitrary set, possibly infinite.
  - Engineering "state machine": discrete named states (finite).
  - Workflow graph: steps (which can span entities).
  - reducer: large data state.
- **Transition**
  - Automaton: `δ : Q×Σ→Q`, defined over the entire alphabet.
  - Engineering "state machine": legal operations restricted by state + jumps.
  - Workflow graph: edges + conditional routing.
  - reducer: folding of arbitrary commands.
- **Typical**
  - Automaton: web applications, raft, everything.
  - Engineering "state machine": protocols, UI, orders.
  - Workflow graph: agent orchestration, BPMN.
  - reducer: replicated state machine `apply`.

## Conclusion

- **Formal register**: automaton = state machine; RSM is an automaton, and `apply` is its transition function.
- **Engineering register**: "state machine" refers to the **single-entity**, restricted-operation workflow FSM; RSM does not count (use **reducer**), and cross-entity / process does not count either (use **workflow graph**).
- Putting the three words in place: **state machine** = a single entity's own states; **workflow graph** = process / multiple entities; **reducer** = fold over a large data state.
- The two registers are each true — do not use the formal register to deny the narrow meaning of the engineering register, nor the reverse.

## Cross-References

- [Consensus Protocol](consensus-protocol-en.md): the replicated state machine implementation perspective (reducer, log store, snapshot compaction)
- [KV Storage Engine](kv-storage-engine-en.md): Raft principles and boundaries