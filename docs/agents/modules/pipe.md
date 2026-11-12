# pipe

**Path:** `internal/pipeline/pipe` (backend plan moves it: the daemon compiles pipe expressions) (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in v0.1  
**Spec v2 path (superseded):** `internal/surface/pipe`  
**Depends on:** pipeline  
**Depended on by:** cli, tui (`λ` prompt), server (`Compile` intent)

## Purpose

Lexer and parser for one-line pipe expressions (`a | b | c`). Produces the same spec as YAML.

## Owns

- Grammar: `part-id key=value "positional"`, `sh '…'`, `agent id/model "prompt"`, `each`/`batch`, `assert`, `as name`.
- `--save` to YAML, `--dry-run` graph.

## Invariants

- Pipe expressions are linear; branches and joins need YAML.

## Spec

- [§14.2 Pipe expressions](../../spec/14-piping.md#142-pipe-expressions)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
