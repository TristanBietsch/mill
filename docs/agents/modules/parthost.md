# parthost

**Path:** `internal/parthost`  
**Status:** planned · first built in M0 (in-process only), v0.5 (external parts)  
**Backend plan path:** `internal/parthost` + `internal/partproto` (message types, transports, conformance suite)  
**Depends on:** protocol, luaapi, plugins  
**Depended on by:** sched, server (`PartCall`)

## Purpose

Resolves parts by id, starts and pools part processes, health-checks them, hosts Lua parts, and routes messages between the scheduler and parts.

## Owns

- Resolution order (built-ins, user plugins, project plugins, inline, PATH, MCP).
- Process pool and `Ping` health checks.
- Context multiplexing for long-lived parts.

## Invariants

- Built-ins are registered here and reached only through the part interface.
- Unknown message types are ignored; unsupported features answer `unsupported`.

## Spec

- [§9.3 Parts protocol](../../spec/09-parts-and-plugins.md#93-parts-protocol-v1)
- [§9.4 Resolution order](../../spec/09-parts-and-plugins.md#94-resolution-order)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
