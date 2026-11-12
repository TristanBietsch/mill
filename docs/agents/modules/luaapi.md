# luaapi

**Path:** `internal/luaapi` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M1 (config phase), v0.5 (Lua parts)  
**Spec v2 path:** same (dispatcher copied from rex)  
**Depends on:** gopher-lua  
**Depended on by:** config, parthost, plugins

## Purpose

The `mill.*` Lua API for config, hooks, commands and Lua parts, the sandbox profiles, and LuaLS type generation.

## Owns

- Config-phase functions (`mill.setup`, `mill.agent`, `mill.plugins`, …).
- Runtime functions and hook dispatch.
- Sandbox profiles: trusted, data-only, part (capability-scoped).

## Invariants

- Config phase finishes within 2 s.
- Hooks have a per-call time limit; a failing hook is logged and skipped.

## Spec

- [§19.4 Lua API](../../spec/19-config-lua.md#194-lua-api)
- [§19.5 Safety](../../spec/19-config-lua.md#195-execution-model-and-safety)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
