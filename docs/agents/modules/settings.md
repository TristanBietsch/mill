# settings

**Path:** `internal/catalog/settings`  
**Status:** planned · first built in M1  
**Backend plan path:** `internal/config/settings` (types copied from rex)  
**Depends on:** none  
**Depended on by:** config, cli (`mill config`), tui-settings

## Purpose

The single settings registry: id, type, default, section, help text and validation for every setting.

## Owns

- Setting definitions.
- Unknown-key suggestions (`did you mean`).
- Normalization and validation (for example `trusted_dirs` path rules).

## Invariants

- One definition drives `mill.setup{}`, `settings.lua`, `mill config` and the TUI page.

## Spec

- [§19.6 Settings registry](../../spec/19-config-lua.md#196-settings-registry)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
