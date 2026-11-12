# builtin

**Path:** `internal/parts/builtin` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M0 (`sh`, `agent`, `stdout`), M1 (data, control), M3 (git, gh)  
**Spec v2 path:** same (agent logic split into `internal/agents` and `agents/detect`)  
**Depends on:** the part interface, runner and workspace via `Open`, expr  
**Depended on by:** parthost

## Purpose

Every built-in part: execution (`sh`, `exec`, `check`, `lua`), data (`map`, `filter`, `jq`, …), control flow (`if`, `assert`, `handoff`, `loop`, …), agents (`agent`, `agent.fanout`, `agent.judge`, `agent.review`, `agent.loop`, `agent.extract`), code and git, I/O and network.

## Owns

- One sub-package or file per part, each with its manifest flags.
- Structured agent output: headless JSON, `MILL_OUTPUT` file, extraction, validation.

## Invariants

- No built-in imports the engine or store.
- `git.commit`/`git.push` use the user's signing setup and never bypass it.
- No part can ask a person anything.

## Spec

- [§10 Built-in parts](../../spec/10-builtin-parts.md#10-built-in-parts-catalog)
- [§11.5 Patterns](../../spec/11-agents.md#115-patterns-built-into-parts)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
