# server

**Path:** `internal/daemon/server` + `daemon/boot` + `daemon/status` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M2  
**Spec v2 path (superseded):** `internal/daemon/server`  
**Depends on:** protocol, sched, store, pipeline, parthost, trigger, config  
**Depended on by:** cmd (milld)

## Purpose

The socket listener and intent handlers. Holds per-connection ordered event queues and the handshake.

## Owns

- Accept loop, peer uid check, bounded per-connection queues.
- Intent → engine/store calls.
- Snapshot on `Hello`, broadcast fan-out.
- Status file for `mill prompt`.

## Invariants

- No broadcast before `Hello` completes.
- Unknown intents get `bad_intent`, never a crash.

## Spec

- [§16.3 Intents](../../spec/16-wire-protocol.md#163-intents)
- [§5.2 Components](../../spec/05-architecture.md#52-components)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
