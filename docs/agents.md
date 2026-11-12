# Agents

Agent registry entries: how mill runs an agent CLI unattended and reads its output, usage and errors.

Status: **not written yet.** Written when the feature ships: v0 ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §11](spec/11-agents.md#11-agents): registry fields, headless and terminal modes, structured output, patterns
- [spec §29.7](spec/29-errors.md#297-agent-errors-are-normalized): normalizing agent errors to classes

When written, this page covers: every registry field, the built-in entries, adding your own agent, `mill agent test`.
