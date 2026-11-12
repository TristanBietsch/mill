# mill — agent documentation

**New agent? Start with [HANDOFF.md](HANDOFF.md)** (onboarding order, state, gate, open work).

Read this file next. For directory layout and "where does new code go?", see [`../STRUCTURE.md`](../STRUCTURE.md).

mill is a Go pipeline engine: `mill` (CLI, TUI, pipe expressions) talks to `milld` (daemon) over newline-delimited JSON on a local socket; `milld` runs pipelines made of parts that speak the same envelope over stdio. **No code exists yet.** Every module below is planned, in the accepted backend layout ([spec §27.4](../spec/27-tech-stack.md#274-repository-layout)).

## Modules

All status: **planned**. Paths follow the backend plan's layout, accepted 2026-11-11 ([modules/overview.md](modules/overview.md)).

| Module | Path | Description |
|--------|------|-------------|
| [cmd](modules/cmd.md) | `cmd/mill`, `cmd/milld` | Entrypoints |
| [cli](modules/cli.md) | `internal/surface/cli` | Subcommands, selectors, exit codes, output modes |
| [tui](modules/tui.md) | `internal/surface/tui` | Boot splash, board, run popup, editor |
| [canvas](modules/canvas.md) | `internal/surface/tui/canvas` | Cell-grid graph renderer |
| [tui-settings](modules/tui-settings.md) | `internal/surface/tui/settings` | Settings page from the registry |
| [sound](modules/sound.md) | `internal/surface/sound` | Soundsets, mixer, silent fallback |
| [pipe](modules/pipe.md) | `internal/pipeline/pipe` | Pipe-expression parser |
| [protocol](modules/protocol.md) | `internal/wire/jsonl`, `internal/wire/protocol` | Envelope and codec; intents and events |
| [client](modules/client.md) | `internal/wire/client` | Dial, handshake, request/ack |
| [server](modules/server.md) | `internal/daemon/server` | Intent handlers, per-connection queues |
| [sched](modules/sched.md) | `internal/engine` (+ `engine/sched`, `budget`, `retry`, `cache`) | Run actor, ready queue, slots, budgets |
| [store](modules/store.md) | `internal/store` | Event log, items, artifacts, index |
| [runner](modules/runner.md) | `internal/runner` | local, pty, container, ssh, remote, external |
| [trigger](modules/trigger.md) | `internal/trigger` | Trigger host, dedupe, concurrency policy |
| [workspace](modules/workspace.md) | `internal/workspace` | Worktrees, temp dirs, cleanup, path safety |
| [secrets](modules/secrets.md) | `internal/secrets` | Broker, redactor |
| [config](modules/config.md) | `internal/config` | Lua loader, layers, reload |
| [luaapi](modules/luaapi.md) | `internal/luaapi` | `mill.*` API, sandboxes, LuaLS types |
| [plugins](modules/plugins.md) | `internal/plugins` | Resolver, installer, lockfile, import/export |
| [parthost](modules/parthost.md) | `internal/parthost`, `internal/partproto` | Part processes, Lua part host, routing; protocol types and conformance suite |
| [builtin](modules/builtin.md) | `internal/parts/builtin` | Built-in parts |
| [pipeline](modules/pipeline.md) | `internal/pipeline` | Parse, lint, edges, compile, YAML edits, layout |
| [expr](modules/expr.md) | `internal/expr` | CEL environment and helpers |
| [settings](modules/settings.md) | `internal/config/settings` | Settings registry |
| [agents](modules/agents.md) | `internal/agents` | Agent registry and merge |
| [sdk](modules/sdk.md) | `sdk/{go,ts,python}` | Parts SDKs |
| [examples](modules/examples.md) | `examples/` | Example pipelines, end-to-end suite |

## Where to start

| Task | Start here |
|------|------------|
| Weekend prototype (M0) | [HANDOFF.md](HANDOFF.md) §6, [../plans/backend/milestones.md](../plans/backend/milestones.md), [modules/pipeline.md](modules/pipeline.md), [modules/expr.md](modules/expr.md) |
| Pipeline format or lint | [interfaces/pipeline-format.md](interfaces/pipeline-format.md), [modules/pipeline.md](modules/pipeline.md) |
| Engine, scheduling, retries, budgets | [modules/sched.md](modules/sched.md), [../plans/backend/engine.md](../plans/backend/engine.md) |
| Run store, event log | [data/schemas.md](data/schemas.md), [modules/store.md](modules/store.md), [../plans/backend/storage.md](../plans/backend/storage.md) |
| Wire protocol | [interfaces/wire.md](interfaces/wire.md), [modules/protocol.md](modules/protocol.md), [../plans/backend/protocols.md](../plans/backend/protocols.md) |
| Parts protocol or a built-in part | [interfaces/parts-protocol.md](interfaces/parts-protocol.md), [modules/builtin.md](modules/builtin.md), [modules/parthost.md](modules/parthost.md) |
| Agents | [modules/agents.md](modules/agents.md), [../plans/backend/execution.md](../plans/backend/execution.md) |
| CLI command | [interfaces/cli.md](interfaces/cli.md), [modules/cli.md](modules/cli.md) |
| Config, settings, Lua | [interfaces/config.md](interfaces/config.md), [modules/config.md](modules/config.md), [modules/luaapi.md](modules/luaapi.md) |
| Secrets, trust, sandboxes | [modules/secrets.md](modules/secrets.md), [../plans/backend/security.md](../plans/backend/security.md), [../../SECURITY.md](../../SECURITY.md) |
| TUI board or editor | [modules/tui.md](modules/tui.md), [../plans/editor.md](../plans/editor.md), `mockups/` |
| Errors | [CONVENTIONS.md](CONVENTIONS.md#errors), [../spec/29-errors.md](../spec/29-errors.md) |
| Tests and CI | [../plans/backend/testing.md](../plans/backend/testing.md), [WORKFLOWS.md](WORKFLOWS.md) |

## Other docs

| File | Contents |
|------|----------|
| [HANDOFF.md](HANDOFF.md) | Start here: current state, conventions, open work |
| [AGENTS.md](AGENTS.md) | Authority, safe vs careful edits, never-do list |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Processes, components, package boundaries, invariants |
| [CONVENTIONS.md](CONVENTIONS.md) | Go style, errors, exit codes, output, naming, docs voice |
| [DECISIONS.md](DECISIONS.md) | ADR log of decisions already in the spec |
| [GLOSSARY.md](GLOSSARY.md) | Process and plan terms (product terms: [../glossary.md](../glossary.md)) |
| [WORKFLOWS.md](WORKFLOWS.md) | Plan → worktree → TDD → gate → review → PR |
| [interfaces/cli.md](interfaces/cli.md) | CLI contract |
| [interfaces/config.md](interfaces/config.md) | Config files, layers, paths, env |
| [interfaces/pipeline-format.md](interfaces/pipeline-format.md) | Pipeline file contract |
| [interfaces/wire.md](interfaces/wire.md) | Daemon wire protocol contract |
| [interfaces/parts-protocol.md](interfaces/parts-protocol.md) | Parts protocol contract |
| [data/schemas.md](data/schemas.md) | Item envelope, event log, pipeline file, lockfile, layout sidecar |

## Human documentation

| File | Contents |
|------|----------|
| [../index.md](../index.md) | Documentation map and authority order |
| [../spec/index.md](../spec/index.md) | The specification |
| [../plans/README.md](../plans/README.md) | Plans |
| [../glossary.md](../glossary.md) | Product glossary |

Spec wins until code exists; then Go source wins.
