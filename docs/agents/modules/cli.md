# cli

**Path:** `internal/surface/cli` + `surface/cli/core` (output kit from rex) + `surface/cli/local` (the `--local` entry) (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M0 (`mill run`), M1 (`--local`, lint, explain), M2/M5 (full CLI)  
**Spec v2 path (superseded):** `internal/surface/cli`  
**Depends on:** wire/client, pipeline, the errors package; `engine`/`store` only from the `--local` entry  
**Depended on by:** cmd

## Purpose

Every `mill` subcommand: argument parsing, selectors, output modes (pretty, plain, JSON, JSONL), exit codes, confirmations. The CLI is a complete product, not a fallback for the TUI.

## Owns

- Selectors: full id, `pipeline#n`, `pipeline#last`, short hex id, `run/node[n]`.
- Output kit: tables with stable columns, `--json`, `--fields`, relative vs RFC 3339 times.
- Exit codes derived from error classes.
- `--yes`, `--dry-run`, confirmation in a terminal only.
- `mill explain`, `mill doctor`, `mill prompt`, completion.

## Invariants

- Data on stdout, diagnostics on stderr.
- With `--json`, failures print the error object on stdout and exit non-zero.
- New table columns only on the right.
- No approve or reply commands (hands-off).

## Spec

- [§17 CLI reference](../../spec/17-cli.md#17-cli-reference)
- [§14 Piping](../../spec/14-piping.md#14-piping-and-unix-composability)
- [§29 Errors](../../spec/29-errors.md#29-errors-and-diagnostics)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
