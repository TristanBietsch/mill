## 20. Files and paths

XDG on Linux and macOS: mill honors `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME` and `XDG_RUNTIME_DIR` on both, with the defaults below when they're unset. Windows uses `%APPDATA%` and `%LOCALAPPDATA%` equivalents. `MILL_CONFIG_DIR`, `MILL_DATA_DIR`, `MILL_STATE_DIR` and `MILL_SOCKET` override.

| Path | Contents |
|---|---|
| `~/.config/mill/` | `init.lua`, `settings.lua` (managed), `lua/`, `pipelines/` |
| `~/.local/share/mill/runs/<run-id>/` | `run.json`, `events.jsonl`, `items/<node>/<n>.jsonl`, `logs/<node-run>.log` (raw output), `transcripts/` |
| `~/.local/share/mill/artifacts/` | Content-addressed artifact store |
| `~/.local/share/mill/cache/` | Node output cache |
| `~/.local/share/mill/index.db` | SQLite index (rebuild with `mill doctor --reindex`) |
| `~/.local/share/mill/plugins/` | Installed plugins, plus the user `lock` |
| `~/.local/state/mill/milld.log` | Daemon log |
| `~/.local/state/mill/logs/tui.log` | TUI log (opened by the boot splash's `log.init`) |
| `~/.local/state/mill/tui-state.json` | Saved selection, view and filter (read by `state.restore`) |
| `$XDG_RUNTIME_DIR/mill/milld.sock` | Socket on Linux when `XDG_RUNTIME_DIR` is set |
| `~/.cache/mill/milld.sock` | Socket everywhere else (always on macOS, which never sets `XDG_RUNTIME_DIR`). A path longer than the OS allows (about 104 bytes on macOS) fails with `M7002`; set `MILL_SOCKET` to a shorter one |
| `~/.local/share/mill/triggers/<pipeline>/<trigger>.json` | Trigger state that survives restarts: last fire, seen dedupe keys, queued fires (§13) |
| `~/.local/share/mill/trust.json` | Trust records: path and SHA-256 of every trusted pipeline, project config and bundle (§22) |
| `<repo>/.mill/` | `mill.lua` (project config), `pipelines/`, `prompts/`, `mill.lock`, layout sidecars, `samples/<pipeline>/` (pinned samples, §18.7.12), `patterns/` (project patterns) |
| `~/.config/mill/patterns/` | Your editor patterns (§18.7.6) |
| `~/.local/state/mill/swap/` | Unsaved editor buffers, for recovery (§18.7.13) |


---

[← spec index](index.md)
