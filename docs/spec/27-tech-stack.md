## 27. Tech stack (Go) and repository layout

### 27.1 Language: Go

mill is written in **Go**. The decision is made; this section records why.

| Need | Why Go fits |
|---|---|
| A daemon supervising many processes, streams, timers and sockets at once | Goroutines and channels match the scheduler, the part host and item streaming almost one-to-one. |
| Single static binaries for macOS, Linux and Windows | `CGO_ENABLED=0` cross-compiles from one machine. `go install` works out of the box. |
| Expressions | `cel-go` is the reference CEL implementation. |
| Lua config, Lua parts and transforms | `gopher-lua` is pure Go, sandboxable, and fast enough for config and glue. |
| TUI | Bubble Tea and Lip Gloss: a mature ecosystem with the same client/model architecture mill's TUI needs. |
| PTYs for terminal-only agents | `creack/pty` on Unix and ConPTY on Windows. |
| Iteration speed | Fast builds, simple tooling, easy for contributors and coding agents to work in. |

The bottleneck is waiting on agents, processes and the network, not CPU, so Go's runtime costs don't matter here. The one area where another language would have an edge is the canvas (Rust's Ratatui renders into a cell buffer). mill handles that with its own small cell-grid canvas component inside Bubble Tea (§18.7).

Minimum Go version: the current stable release at the time of v0, raised promptly for security fixes.

### 27.2 Libraries

| Area | Choice |
|---|---|
| TUI | Bubble Tea, Lip Gloss, Bubbles; a custom cell-grid canvas (evaluate Charm's `cellbuf`) |
| Expressions | `cel-go` |
| Lua | `gopher-lua`, plus a typed binding layer for the `mill.*` API |
| YAML (comment-preserving edits) | `yaml.v3` node API |
| Storage | JSONL files; SQLite through a pure-Go driver (keeps builds CGO-free) |
| PTY | `creack/pty`, ConPTY |
| Files and schedules | `fsnotify`; a standard cron parser |
| MCP | The official Go MCP SDK |
| WASM parts (later) | `wazero` (pure Go) |
| HTTP gateway | Standard library `net/http`, WebSocket for the remote API |
| Containers | Docker and Podman through their CLIs (no heavy SDK dependency) |
| Audio | `ebitengine/oto` for playback with pure-Go WAV/OGG decoding; sounds embedded in the binary |
| Testing | Standard `testing`, `testify`, `goleak` (no leaked goroutines), `go-internal/testscript` for CLI tests |
| Releases | GoReleaser: binaries, checksums, signatures, Homebrew tap, Scoop, `.deb`/`.rpm` |

### 27.3 Non-Go pieces

| Piece | Stack | When |
|---|---|---|
| Parts SDKs | Go, TypeScript, Python (thin wrappers around the JSONL protocol) | v0.5 |
| Web run viewer | Go-served HTML plus the SVG from `mill render --format svg`. No JavaScript framework. | v1 |
| Web editor (only if there's demand) | TypeScript, React, React Flow, talking to the gateway | later |

Every client speaks the same wire protocol, so adding a non-Go frontend later never touches the daemon.

### 27.4 Repository layout

Single Go module, two binaries. Docs follow the same structure as the code: reference docs per surface, how-tos per task, and module docs for contributors and coding agents.

```
cmd/
  mill/                     client: CLI + TUI
  milld/                    daemon
internal/
  surface/cli/              subcommands, selectors, exit codes, output modes
  surface/tui/              boot splash (splash.go, splash_steps.go), board, run popup, log, editor, settings
  surface/tui/canvas/       cell-grid renderer, zoom levels, hit-testing
  surface/tui/editor/       editor model: canvas, inspector, pipeline panel, palette, panels, keymap
  surface/tui/settings/     settings page generated from the registry
  surface/sound/            soundsets, event mixer, debounce, silent fallback
  surface/pipe/             pipe-expression parser
  wire/protocol/            envelope, intents, events, codec (shared with parts protocol)
  wire/client/              socket dial, handshake, request/ack
  daemon/server/            intent handlers, per-connection queues
  daemon/sched/             ready queue, slots, budgets
  daemon/store/             event log, items, artifacts, index
  daemon/runner/            local, pty, container, ssh, remote, external
  daemon/trigger/           trigger host, dedupe, concurrency policy
  daemon/workspace/         worktrees, temp dirs, cleanup, path safety
  daemon/secrets/           broker, redactor
  config/                   Lua loader, layers and precedence, settings.lua writer, reload
  luaapi/                   the mill.* Lua API, sandboxes, LuaLS type generation
  plugins/                  resolver, installer, manifest, permissions, lockfile, pack, import/export
  parthost/                 process pool, Lua part host, lifecycle, routing
  parts/builtin/            sh, exec, check, data, control, agent*, git, gh, http, mcp, file …
  pipeline/                 parse, lint, edge inference, compile
  pipeline/edit/            every edit operation and quick fix over the yaml.v3 tree (shared by CLI and TUI)
  pipeline/layout/          slots, sidecar pins, edge routing into a cell grid (editor, run popup, mill render)
  expr/                     CEL environment and helpers
  catalog/settings/         settings registry
  catalog/agents/           agent registry + merge
sdk/
  go/ ts/ python/           parts SDKs
plugins/                    official plugins (github, slack, linear, rex …), each a normal plugin
examples/                   example pipelines (also integration tests)
docs/
  index.md quickstart.md cli.md tui.md pipeline-format.md parts-protocol.md plugin-format.md
  lua-api.md protocol.md agents.md runners.md triggers.md settings.md paths.md security.md glossary.md
  howto/ (write-a-plugin, write-a-lua-part, add-an-agent, import-export, pipe-recipes, headless-ci, webhooks, mcp)
  agents/ (module map and handoff notes for coding agents)
```

Gate for every change: `go vet`, race tests, lint on Linux, macOS and Windows, coverage floors per package, `govulncheck`. Conventional Commits. A changelog entry for every user-visible change.


---

[← spec index](index.md)
