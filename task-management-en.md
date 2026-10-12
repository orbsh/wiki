# Task Management Architecture: The scratch Improvement Plan and Agent Integration

> **Languages:** [English](task-management-en.md) (primary) · [中文](task-management.md)
> **Status**: Improvement Plan / Design Analysis (2026-08-28)

---

## 0. Current State of scratch (Source)

scratch is an existing tool:

- Local: `~/Configuration/nushell/scripts/scratch`
- Online: <https://github.com/orbsh/scratch.nu>

This document has two layers of intent: **① the improvement plan for scratch**; **② the agent integration model** — which may eventually be integrated into the agent together with [Graph Memory](graph-memory-en.md).

## 1. Core Form (scratch Model)

scratch **stores to a database**:

- Each entry is a **record** (record).
- **Tree organization**.
- There is also a **tag system**, and tags can likewise be organized into a **tree**.

## 2. scratch vs neorg Comparison

### scratch: Structured == Tree == Mind Map

Can be **extended into a network structure**:

- Rust rewrite
- Use KV storage

### neorg: Textual == Editable

- Cannot accommodate much metadata
- Metadata requires considering visual presentation
- Syntax must be designed
- **Visual presentation and syntax design are coupled** → poor extensibility (changing the presentation often drags the syntax along)

### scratch's Interface Shortcomings

- Usable only in the terminal; no interface that can be modified in real time.
- Development directions:
  - Build a web version
  - Combine with the launcher
  - cli interface

## 3. Improvement Plan

- **Extend the structured form into a network structure**: Rust rewrite; use KV storage.
- **Interface shortcomings**: web version / combine with launcher / cli interface (a real-time editable interface + keyboard-workflow access).
- **Visual presentation**: both the structured and the textual forms **need** visual presentation designed (it is not a cost unique to either side). The difference lies in the degree of coupling — **the textual form (neorg) couples visual presentation with syntax design, giving poor extensibility**; the structured form (scratch), on this basis, is easier to decouple.

## 4. Agent Integration Model

- Consider the integration model between scratch and the agent: the task tree + tag tree can serve as the agent's task/memory structure.
- **Coordination positioning**: the task graph is not only a personal task list, but also a **multi-agent coordination medium** (a shared task view replacing mutual message passing) and an **organizational ledger** (if work goes through the graph, tracking costs nothing) — for the argument see [Graph Memory](graph-memory-en.md) §Task Networking.
- **Endgame**: integrated into the agent together with [Graph Memory](graph-memory-en.md) (the task structure merged into the agent's memory/reasoning orchestration).
