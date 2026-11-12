# config

**Path:** `internal/config`  
**Status:** planned · first built in M1  
**Backend plan path:** `internal/config` + `config/settings`  
**Depends on:** luaapi, settings (registry), paths  
**Depended on by:** server, cli, tui

## Purpose

Loads Lua config in layers (defaults, `settings.lua`, `init.lua`, `.mill/mill.lua`, env, flags), writes the managed `settings.lua`, reloads on change, answers `--why`.

## Owns

- Layering and precedence.
- Reload that keeps the previous config on error.
- `mill config get --why`, `list`, `check`.

## Invariants

- A failed reload never takes the daemon down.
- Untrusted project config is data-only.

## Spec

- [§19 Configuration](../../spec/19-config-lua.md#19-configuration-lua)
- [§19.2 Load order](../../spec/19-config-lua.md#192-load-order-and-precedence)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
