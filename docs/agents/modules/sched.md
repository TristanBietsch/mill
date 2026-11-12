# sched

**Path:** `internal/daemon/sched`  
**Status:** planned · first built in M0 (sequential), M1 (real scheduler)  
**Backend plan path:** `internal/engine` (run actor, reducer) + `engine/sched`, `engine/budget`, `engine/retry`, `engine/cache`  
**Depends on:** pipeline, store, parthost (via the part interface), expr, clock  
**Depended on by:** server, `--local` entry

## Purpose

Decides when each node-run starts: readiness, the ready queue, concurrency slots, budgets, retries, caching, cancellation and crash recovery.

## Owns

- Readiness and skip propagation.
- Slots: global, per-runner, per-agent, per-run.
- Budgets and retry backoff.
- Cancel/teardown path; recovery on restart.

## Invariants

- Slots acquired before start, released at terminal state on every path.
- Terminal node-runs never change; a retry is a new node-run.
- Non-idempotent live node-runs become `crashed` after a restart, never re-run.

## Spec

- [§15 Execution semantics](../../spec/15-execution.md#15-execution-semantics)
- [§15.7 Crash recovery](../../spec/15-execution.md#157-daemon-restarts-and-crash-recovery)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
