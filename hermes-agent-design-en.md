# Hermes Agent — Usage, Configuration and Operations Notes

> **Languages:** [English](hermes-agent-design-en.md) (primary) · [中文](hermes-agent-design.md)

*Hermes-specific usage conventions, configuration details, and project-specific operations protocols.*

## Model Selection
- Don't use an overly intelligent model for routine tasks. `qwen3.6-plus` is already overkill.
- `flash`-tier models should suffice for most agent work — faster and cheaper.
- Small post-trained models (10B tier) can handle complex agent tasks efficiently.

→ For the detailed model selection matrix and migration decisions, see [LLM Overthinking Critique](llm-overthinking-critique-en.md) and [Qwen to MiMo Migration](qwen-to-mimo-migration-en.md)

## Behavior Patterns
- Conversational messages with a user prefix (e.g. `O: ...`) usually require: analyzing the viewpoint, filling in the omitted logic, verifying the correctness of the reasoning.
- Session transcripts for recovery: `# <<<` = user message, `# >>>` = assistant response.
