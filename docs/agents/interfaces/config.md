# Configuration (agents)

Contract summary. Reference: **[spec §19](../../spec/19-config-lua.md#19-configuration-lua)** (Lua config) and **[spec §20](../../spec/20-paths.md#20-files-and-paths)** (paths). User docs (when shipped): [../../settings.md](../../settings.md), [../../paths.md](../../paths.md), [../../lua-api.md](../../lua-api.md). Status: planned.

## Files

| File | Purpose | Trust |
|------|---------|-------|
| `~/.config/mill/init.lua` | Your config, fully programmable | Trusted |
| `~/.config/mill/settings.lua` | Managed by `mill config set` and the TUI; plain `return { … }` | Trusted, data only |
| `~/.config/mill/lua/` | Your modules for `require` | Trusted |
| `<repo>/.mill/mill.lua` | Project config: plugins, agents, defaults | Data-only until `mill trust .` |
| `plugin.lua` | Plugin manifest | Always data-only |

## Layers (lowest to highest)

1. Built-in defaults
2. `settings.lua`
3. `init.lua`
4. `.mill/mill.lua`
5. Environment (`MILL_MAX_CONCURRENT_AGENTS=2`)
6. Command-line flags

`mill config get <key> --why` names the winning layer. A failed reload keeps the previous config and reports file and line ([§19.2](../../spec/19-config-lua.md#192-load-order-and-precedence)).

## Lua API surface

Config phase: `mill.setup`, `mill.plugins`/`mill.plugin`, `mill.agent`, `mill.mcp`, `mill.runner`, `mill.secrets`, `mill.part` with `mill.node`/`mill.trigger`/`mill.sink`, `mill.pipeline`, `mill.on`, `mill.soundset`, `mill.command`, `mill.keymap`, `mill.theme`. Helpers and runtime functions: [§19.4](../../spec/19-config-lua.md#194-lua-api).

Limits: config phase within 2 s; hooks time-limited per call; a failing hook is logged and skipped ([§19.5](../../spec/19-config-lua.md#195-execution-model-and-safety)).

## Settings

One registry defines every setting (id, type, default, section, help, validation). Unknown keys warn with the closest valid key; wrong types are errors with file and line. Table: [§19.6](../../spec/19-config-lua.md#196-settings-registry). Code: [../modules/settings.md](../modules/settings.md).

## Paths

| Path | Contents |
|------|----------|
| `~/.config/mill/` | `init.lua`, `settings.lua`, `lua/`, `pipelines/` |
| `~/.local/share/mill/runs/<run-id>/` | `run.json`, `events.jsonl`, items, raw logs, transcripts |
| `~/.local/share/mill/artifacts/` | Content-addressed artifacts |
| `~/.local/share/mill/cache/` | Node output cache |
| `~/.local/share/mill/index.db` | SQLite index (rebuildable) |
| `~/.local/share/mill/plugins/` | Installed plugins, user lock |
| `~/.local/state/mill/` | `milld.log`, `logs/tui.log`, `tui-state.json` |
| `$XDG_RUNTIME_DIR/mill/milld.sock` | Socket |
| `<repo>/.mill/` | `mill.lua`, `pipelines/`, `prompts/`, `mill.lock`, layout sidecars |

## Environment

| Variable | Effect |
|----------|--------|
| `MILL_CONFIG_DIR`, `MILL_DATA_DIR`, `MILL_STATE_DIR`, `MILL_SOCKET` | Override paths (spec §20) |
| `MILL_<SETTING>` | Setting override, layer 5 |
| `MILL_DEBUG=1` | Stack traces in errors |
| `MILL_ASCII=1` | Words instead of glyphs |
| `NO_COLOR` | Plain output |
| `MILL_OUTPUT`, `MILL_RUN_ID`, `MILL_NODE_ID`, `MILL_WORKSPACE`, `MILL_ARTIFACTS` | Set by mill for agent and step processes ([§11.3](../../spec/11-agents.md#113-getting-structured-output-from-an-agent)) |

## Decided (2026-11-11)

- Socket: `$XDG_RUNTIME_DIR/mill/milld.sock` on Linux, else `~/.cache/mill/milld.sock` (always on macOS) (backend D20, spec §20).
- `XDG_*` is honored on macOS and Linux (backend D21, spec §20).
- Environment passed to steps: the `env.passthrough` allowlist plus pipeline and node `env` (backend D30, [spec §12.1](../../spec/12-runners.md#121-environment)).
