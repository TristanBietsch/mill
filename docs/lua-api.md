# Lua API

The `mill.*` API for config, hooks, commands and Lua parts.

Status: **not written yet.** Written when the feature ships: v0 (config phase: `mill.setup`, `mill.agent`) and v0.5 (Lua parts) ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §19.4](spec/19-config-lua.md#194-lua-api): config-phase, helper and runtime functions, hook events
- [spec §19.5](spec/19-config-lua.md#195-execution-model-and-safety): what runs where, sandboxes, time limits
- [spec §19.3](spec/19-config-lua.md#193-example-initlua): example `init.lua`

When written, this page covers: every function with signature and example; generated LuaLS types (`mill lua-types`).
