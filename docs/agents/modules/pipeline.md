# pipeline

**Path:** `internal/pipeline` + `pipeline/ir`, `pipeline/pipe`, `pipeline/yamledit`, `pipeline/layout` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M0 (subset), M1 (lint), M5 (YAML edits), v0.3 (layout)  
**Spec v2 path (superseded):** `internal/pipeline`  
**Depends on:** expr, the errors package  
**Depended on by:** sched, server, cli, tui

## Purpose

Parses `*.mill.yaml` with positions, lints, infers edges, and compiles to an immutable execution graph. Also surgical YAML edits for `mill node`/`edge` and the editor, and graph layout.

## Owns

- Parse with line and column for every key.
- Lint rules and their M-codes (`M2xxx`, `M3xxx`).
- Edge inference.
- Compile at a fixed content hash.
- Comment-preserving edits; layout sidecar read and write.

## Invariants

- A run executes the pipeline version it started with.
- A broken pipeline is never saved without `--force`.

## Spec

- [§7 Pipeline format](../../spec/07-pipeline-format.md#7-pipeline-file-format)
- [§7.4 Edge inference](../../spec/07-pipeline-format.md#74-edge-inference)
- [§7.7 Validation](../../spec/07-pipeline-format.md#77-validation-mill-lint)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
