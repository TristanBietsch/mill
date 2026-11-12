# agents

**Path:** `internal/catalog/agents`  
**Status:** planned · first built in M0 (hard-coded entries), M1 (registry and merge)  
**Backend plan path:** `internal/agents` + `agents/detect`  
**Depends on:** luaapi (entries from `mill.agent`)  
**Depended on by:** builtin (`agent*`), runner (pty)

## Purpose

The agent registry: built-in entries, merge by id with user and plugin entries, argv building, headless output parsing, usage, error normalization, terminal state detection and `auto_answer`.

## Owns

- Built-in entries: Claude Code, Codex CLI, Gemini CLI, Aider, OpenCode, Ollama, `echo`.
- Merge rules: scalars replace, `models` merge by id.
- Error patterns → classes.

## Invariants

- Folder-trust prompts are answered only in `trusted_dirs`.
- An unknown question fails with `M6304`; nothing waits.

## Spec

- [§11.1 Agent registry](../../spec/11-agents.md#111-agent-registry-millagent-in-lua)
- [§29.7 Agent errors](../../spec/29-errors.md#297-agent-errors-are-normalized)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
