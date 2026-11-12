# sdk

**Path:** `sdk/go`, `sdk/ts`, `sdk/python`  
**Status:** planned · first built in v0.5  
**Backend plan path:** `sdk/go` first (thin wrapper over `partproto`)  
**Depends on:** the parts protocol only  
**Depended on by:** third-party parts

## Purpose

Thin helpers that handle the envelope, manifest and lifecycle so a part is a handful of handlers.

## Owns

- Per-language envelope codec and lifecycle loop.

## Invariants

- SDKs add no behavior the protocol lacks.

## Spec

- [§9.3 Parts protocol](../../spec/09-parts-and-plugins.md#93-parts-protocol-v1)
- [§27.3 Non-Go pieces](../../spec/27-tech-stack.md#273-non-go-pieces)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
