# trigger

**Path:** `internal/trigger` + one sub-package per kind (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M4 (cron, poll, run), M7 (webhook, fswatch, git)  
**Spec v2 path (superseded):** `internal/daemon/trigger`  
**Depends on:** pipeline, store (trigger state), clock  
**Depended on by:** server

## Purpose

Runs trigger parts and turns their events into runs, with dedupe, debounce and the per-pipeline concurrency policy.

## Owns

- Arm and disarm on load and reload.
- `queue`, `skip`, `replace`, `parallel: N`.
- Persistent state (last fire, seen keys, catch-up).

## Invariants

- Untrusted pipelines never arm triggers.

## Spec

- [§13 Triggers](../../spec/13-triggers.md#13-triggers)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
