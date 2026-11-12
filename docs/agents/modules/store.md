# store

**Path:** `internal/daemon/store`  
**Status:** planned · first built in M0 (run dir), M1 (event log), M2 (index)  
**Backend plan path:** `internal/store` + `store/cas`, `store/index`, `store/lock` (out of `daemon/` so `--local` can use it)  
**Depends on:** protocol types, secrets (redactor), paths  
**Depended on by:** sched, server, cli (read-only `--store`)

## Purpose

The run store: per-run append-only `events.jsonl`, item files, raw logs, the content-addressed artifact store, and the rebuildable SQLite index.

## Owns

- Run directories and their layout.
- Event appends and durability.
- Items per node-run and port.
- Artifacts by SHA-256.
- Index rebuild (`mill doctor --reindex`).

## Invariants

- Append-only; corrections are new events.
- Redaction is the only transformation before persistence.
- The index is a cache.

## Spec

- [§20 Paths](../../spec/20-paths.md#20-files-and-paths)
- [§6.3 Artifacts](../../spec/06-data-model.md#63-artifacts)
- [§5.4 Invariants](../../spec/05-architecture.md#54-invariants)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
