# Tech overview

One page on how mill is built. The spec's [§27](spec/27-tech-stack.md#27-tech-stack-go-and-repository-layout) is the reference; the backend plan ([plans/backend/00-overview.md](plans/backend/00-overview.md)) has the detail. No code exists yet.

## Stack

| Item | Choice |
|---|---|
| Language | Go 1.27 (`toolchain go1.27.4`), `CGO_ENABLED=0`, one static binary per platform |
| Module | `github.com/TristanBietsch/mill`, public at [github.com/TristanBietsch/mill](https://github.com/TristanBietsch/mill), MIT |
| Platforms | macOS and Linux tier 1; Windows tier 2 in v0 (built and tested in CI; PTY and containers best-effort) |
| Distribution | GoReleaser; Homebrew tap `tristanbietsch/tap/mill`, Scoop, `.deb`/`.rpm` |

Key libraries ([plans/backend/tech-stack.md](plans/backend/tech-stack.md) has versions, licenses and the alternatives):

| Area | Library |
|---|---|
| Expressions | `cel-go` |
| Lua config and parts | `gopher-lua` |
| YAML | `go.yaml.in/yaml/v4`, edits as text splices |
| JSON Schema | `santhosh-tekuri/jsonschema/v6` |
| SQLite index | `modernc.org/sqlite` (pure Go) |
| TUI | Bubble Tea v2, Lip Gloss v2, Bubbles v2 |
| PTY | `creack/pty`, ConPTY via `charmbracelet/x/xpty`, rex's `vt10x` fork |
| MCP | the official Go MCP SDK |
| Files and schedules | `fsnotify`, `gronx` |
| Audio | `ebitengine/oto/v3` |
| CLI | stdlib `flag` plus the output kit copied from rex |
| Tests | `testing`, `testify`, `goleak`, `testscript` |

## Two binaries

| Binary | Role |
|---|---|
| `mill` | Client: CLI, TUI, pipe expressions. Dials the daemon and starts it if needed. `mill run --local` hosts the engine in-process instead. |
| `milld` | Daemon: loads pipelines, hosts parts, arms triggers, schedules node-runs, supervises child processes, writes the run store. |

The engine is a library that both `milld` and `--local` host, so the weekend prototype is the first slice of the real thing ([plans/backend/architecture.md](plans/backend/architecture.md)).

## Storage

Files are the truth ([§20](spec/20-paths.md#20-files-and-paths), [plans/backend/storage.md](plans/backend/storage.md)):

- One append-only `events.jsonl` per run; run state is a fold over it. Crash recovery replays it ("crash, don't guess").
- Items as per-port JSONL files; artifacts in one global content-addressed store.
- A SQLite index that is only a cache (`mill doctor --reindex` rebuilds it).
- Secret values are redacted before anything is written.

## Protocols

- **Daemon protocol:** one JSON object per line, `{v, kind, type, id, data}`, over a Unix domain socket on every OS (`$XDG_RUNTIME_DIR/mill/milld.sock` on Linux, else `~/.cache/mill/milld.sock`). [§16](spec/16-wire-protocol.md#16-daemon-wire-protocol), [plans/backend/protocols.md](plans/backend/protocols.md).
- **Parts protocol:** the same envelope over a part's stdio, plus an optional `ctx` field for long-lived parts. Built-ins run in-process behind the same interface, proven equal by a conformance suite. [§9.3](spec/09-parts-and-plugins.md#93-parts-protocol-v1).

## Code from rex

rex already solves much of the plumbing (codec, socket server, daemon autostart, PTY console, Lua dispatcher, CLI kit, test helpers, CI). mill copies those files and lets them diverge; each copied file names the rex commit it came from. A shared module is reconsidered after M5.

## Build order

Milestones M0 (weekend prototype) to M11, each ending in something runnable: [plans/backend/milestones.md](plans/backend/milestones.md). The first dogfood pipeline is `nightly-vuln-sweep`, run from OS cron at the end of M0.

## Where to go next

- Package layout and layering: [plans/backend/architecture.md](plans/backend/architecture.md), [STRUCTURE.md](STRUCTURE.md).
- Engine semantics: [plans/backend/engine.md](plans/backend/engine.md), [spec §15](spec/15-execution.md#15-execution-semantics).
- Security: [plans/backend/security.md](plans/backend/security.md), [spec §22](spec/22-security.md#22-security-and-trust).
- Testing and the gate: [plans/backend/testing.md](plans/backend/testing.md).
- What it looks like: [design.md](design.md).
