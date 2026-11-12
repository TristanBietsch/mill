# tui-settings

**Path:** `internal/surface/tui/settings` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in v0.2  
**Spec v2 path:** same  
**Depends on:** settings (registry)  
**Depended on by:** tui

## Purpose

The TUI settings page, generated from the settings registry. Writes go to the managed `settings.lua`.

## Owns

- Rendering sections and rows from registry metadata.
- Editing and validation feedback.

## Invariants

- No setting is defined here; the registry is the only definition.

## Spec

- [§18.10 Settings](../../spec/18-tui.md#1810-settings)
- [§19.6 Settings registry](../../spec/19-config-lua.md#196-settings-registry)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
