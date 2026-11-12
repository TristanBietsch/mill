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

**Toolchain.** `go 1.27` in `go.mod` with `toolchain go1.27.4`; the patch floor is raised promptly for security fixes. `CGO_ENABLED=0` for every target.

**Platforms.** macOS and Linux are tier 1. Windows is **tier 2** in v0: it builds, runs the unit and integration tests and a `--local` smoke test in CI from the first engine milestone, while PTY (terminal agents) and the container runner are best-effort until the hardening milestone. Targets: `darwin/arm64`, `darwin/amd64`, `linux/amd64`, `linux/arm64`, `windows/amd64`, `windows/arm64`.

**Name, repository, distribution.** The name stays `mill`. Source: [github.com/TristanBietsch/mill](https://github.com/TristanBietsch/mill) (public), module path `github.com/TristanBietsch/mill`. Homebrew's core formula `mill` is an unrelated Scala build tool, so mill ships from its own tap: `brew install tristanbietsch/tap/mill`. License: MIT.

**Code from rex.** Plumbing that rex already has (JSONL codec, socket server, daemon autostart, paths, locks, PTY console, Lua dispatcher, settings types, CLI output kit, test helpers, CI) is copied and diverges. Each copied file starts with a header naming the rex source commit and path. Whether to extract a shared module is revisited after v0.1.

### 27.2 Libraries

| Area | Choice |
|---|---|
| TUI | Bubble Tea v2, Lip Gloss, Bubbles; a custom cell-grid canvas (evaluate Charm's `cellbuf`) |
| Expressions | `cel-go` |
| Lua | `gopher-lua`, plus a typed binding layer for the `mill.*` API |
| YAML | `go.yaml.in/yaml/v4` (the maintained successor of the archived `yaml.v3`) for parsing with positions and comments. No Go YAML library re-emits a file byte for byte, so edits are **text splices**: an edit replaces one value's byte range or inserts a line at the right indentation, then the result is re-parsed and compared with the intended document, with golden tests on hand-formatted files |
| Storage | JSONL files; SQLite through `modernc.org/sqlite` (pure Go, keeps builds CGO-free), WAL mode, rebuilt rather than migrated |
| PTY | `creack/pty`, ConPTY |
| Files and schedules | `fsnotify`; a standard cron parser |
| MCP | The official Go MCP SDK |
| WASM parts (later) | `wazero` (pure Go) |
| HTTP gateway | Standard library `net/http`, WebSocket for the remote API |
| Containers | Docker and Podman through their CLIs (no heavy SDK dependency) |
| Audio | `ebitengine/oto` for playback with pure-Go WAV/OGG decoding; sounds embedded in the binary |
| CLI | Standard library `flag` plus the output kit copied from rex (formats, printer, selectors, golden tests); no CLI framework |
| Testing | Standard `testing`, `testify`, `goleak` (no leaked goroutines), `go-internal/testscript` for CLI tests |
| Releases | GoReleaser: binaries, checksums, keyless signature on the checksum file, the `tristanbietsch/tap` Homebrew tap, Scoop, `.deb`/`.rpm` |

### 27.3 Non-Go pieces

| Piece | Stack | When |
|---|---|---|
| Parts SDKs | Go, TypeScript, Python (thin wrappers around the JSONL protocol) | v0.5 |
| Web run viewer | Go-served HTML plus the SVG from `mill render --format svg`. No JavaScript framework. | v1 |
| Web editor (only if there's demand) | TypeScript, React, React Flow, talking to the gateway | later |

Every client speaks the same wire protocol, so adding a non-Go frontend later never touches the daemon.

### 27.4 Repository layout

Single Go module, two binaries. Library code doesn't live under `daemon/` unless it only exists because there's a daemon: the engine, store, runners, workspaces, secrets and triggers are used by `mill run --local` and the weekend prototype too. Docs follow the same structure as the code: reference docs per surface, how-tos per task, and module docs for contributors and coding agents. The full tree, with layering rules, is in `docs/plans/backend/architecture.md` §2–3.

```
cmd/
  mill/                     client: CLI + TUI
  milld/                    daemon (calls daemon/boot)
internal/
  core/                     ids, states, items, refs, budgets, selectors (plain data)
  errs/                     the §29 error object, codes, classes, explain pages, exit codes
  clock/ paths/ mlog/       time (real and fake), config/data/state/socket paths, logging
  wire/jsonl/               envelope and codec, shared by the daemon and parts protocols
  wire/protocol/            daemon intents and events
  wire/client/              socket dial, handshake, request/ack
  partproto/                parts protocol messages, Part interface, transports, conformance suite
  pipeline/                 parse, lint, edge inference, compile
  pipeline/ir/              the compiled graph
  pipeline/pipe/            pipe-expression parser
  pipeline/edit/            every edit operation and quick fix (shared by CLI, daemon and TUI)
  pipeline/yamledit/        text-splice YAML edits under pipeline/edit
  pipeline/layout/          slots, sidecar pins, edge routing into a cell grid (editor, run popup, mill render)
  expr/  schema/            CEL environment and helpers; output shorthand → JSON Schema
  engine/                   run actor, reducer, readiness
  engine/sched/ engine/budget/ engine/retry/ engine/cache/
  store/                    run dirs, event log, items, logs
  store/cas/ store/index/ store/lock/   artifact store, SQLite index, locks
  parthost/                 part registry and resolution, process pool, health, PartCall
  parts/builtin/            sh, exec, check, data, control, agent*, git, gh, http, mcp, file …
  agents/ agents/detect/    agent registry and merge; terminal-agent state detection
  drafter/                  sentence → pipeline proposal for the wizard: local Ollama client, model resolver, keyword fallback, output validation
  runner/                   local, pty, container, ssh (remote, external later); process groups
  workspace/                worktrees, temp dirs, cleanup, retry snapshots, path safety
  secrets/ secrets/redact/  broker and backends; redactor
  trigger/                  trigger host, dedupe, concurrency policy; one package per kind
  config/ config/settings/  Lua loader, layers and precedence, settings registry, settings.lua writer
  luaapi/                   the mill.* Lua API, sandboxes, hooks, LuaLS type generation
  plugins/                  manifest, resolver, installer, permissions, lockfile, pack, import/export
  daemon/server/ daemon/boot/ daemon/status/   socket handlers, wiring, status file
  daemonctl/                find, start and reach milld
  gateway/                  HTTP and WebSocket front door (later)
  surface/cli/              subcommands, selectors, exit codes, output modes; surface/cli/local runs --local
  surface/tui/              boot splash (splash.go, splash_steps.go), board, run popup, log, editor, settings
  surface/tui/canvas/       cell-grid renderer, zoom levels, hit-testing
  surface/tui/editor/       editor model: canvas, inspector, pipeline panel, palette, panels, keymap
  surface/tui/settings/     settings page generated from the registry
  surface/sound/            soundsets, event mixer, debounce, silent fallback
  surface/mcpserver/        mill mcp serve
  testutil/                 test env, golden files, fakes, echo agent harness
sdk/
  go/ ts/ python/           parts SDKs
third_party/vt10x/          terminal emulator fork (from rex)
plugins/                    official plugins (github, slack, linear, rex …), each a normal plugin
examples/                   example pipelines (also integration tests)
docs/
  index.md design.md tech.md quickstart.md cli.md tui.md pipeline-format.md parts-protocol.md plugin-format.md
  lua-api.md protocol.md agents.md runners.md triggers.md settings.md paths.md security.md glossary.md
  howto/ (write-a-plugin, write-a-lua-part, add-an-agent, import-export, pipe-recipes, headless-ci, webhooks, mcp)
  agents/ (module map and handoff notes for coding agents)
```

Gate for every change: `go vet`, race tests, lint on Linux, macOS and Windows, coverage floors per package, `govulncheck`. Conventional Commits. A changelog entry for every user-visible change.


---

[← spec index](index.md)
