# client

**Path:** `internal/wire/client`  
**Status:** planned · first built in M2  
**Backend plan path:** same  
**Depends on:** protocol  
**Depended on by:** cli, tui

## Purpose

Client SDK: find or start the daemon, dial, `Hello`, send intents and wait for the matching reply, stream events.

## Owns

- Auto-start of `milld` when unreachable.
- Request/reply correlation by `id`.
- Version check (`M7003` when the daemon is older).

## Invariants

- An intent with an `id` gets exactly one reply.

## Spec

- [§16.2 Handshake](../../spec/16-wire-protocol.md#162-handshake)
- [§29.9 Common failures](../../spec/29-errors.md#299-common-failures-designed-in-advance)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
