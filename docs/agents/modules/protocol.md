# protocol

**Path:** `internal/wire/jsonl` (envelope + codec, copied from rex) and `internal/wire/protocol` (payloads) (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M2  
**Spec v2 path (superseded):** `internal/wire/protocol`  
**Depends on:** the errors package  
**Depended on by:** client, server, parthost, cli, tui

## Purpose

The envelope, intent and event payload types, error codes and the JSONL codec shared by the daemon protocol and the parts protocol.

## Owns

- Envelope `{v, kind, type, id, data}`.
- Codec: 16 MiB line cap, base64 for bytes, unknown fields ignored.
- Intent and event structs; protocol error codes.

## Invariants

- One object per line; `v` must match.
- Additive changes only without a version bump.

## Spec

- [§16 Wire protocol](../../spec/16-wire-protocol.md#16-daemon-wire-protocol)
- [§5.4 Invariants](../../spec/05-architecture.md#54-invariants)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
