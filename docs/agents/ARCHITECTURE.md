# Architecture

Agent-oriented summary of [spec §5](../spec/05-architecture.md#5-architecture). Nothing here is built yet. For depth (Go interfaces, goroutine ownership, run lifecycle diagrams) read [../plans/backend/architecture.md](../plans/backend/architecture.md).

## Process model

```
┌──────────────────┐   JSONL / local socket   ┌──────────────────────────────┐
│ mill             │ ◄──────────────────────► │ milld                        │
│  CLI · TUI ·     │   Intent / Event         │  server (handlers, queues)   │
│  pipe expressions│                          │  pipeline loader             │
└──────────────────┘                          │  scheduler (slots, budgets)  │
        │ --local: same engine, in-process    │  trigger host                │
        ▼                                     │  part host ──► parts         │
   run store on disk                          │  runner layer ──► processes  │
                                              │  run store, workspaces,      │
                                              │  secrets broker, expr engine │
                                              │  gateway (optional, HTTP/WS) │
                                              └──────┬───────────────────────┘
                                                     │ stdio JSONL (parts protocol)
                                                     ▼
                                     part processes · step processes · agent CLIs
```

| Process | Role |
|---------|------|
| `mill` | Client. CLI subcommands, TUI, pipe expressions. Dials the daemon and auto-starts it. `--local` runs one pipeline in-process without a daemon |
| `milld` | Daemon. Loads pipelines, hosts parts, runs triggers, schedules node-runs, supervises child processes, writes the run store |
| Part processes | Long-lived or per-call children of `milld` speaking the parts protocol |
| Step processes | Programs started by runners for `exec`-style nodes and agents |

## Components (inside `milld`)

| Component | Owns | Planned package (spec §27.4, backend layout) |
|-----------|------|------------------------------|
| Server | Socket, handshake, intent handlers, per-connection ordered queues | `daemon/server` |
| Pipeline loader | Parse, lint, compile `*.mill.yaml` and pipe expressions; watch and reload | `pipeline`, `pipeline/pipe` |
| Scheduler | Ready queue, concurrency caps, budgets, when a node-run starts | `engine`, `engine/sched` |
| Part host | Start, pool, health-check, stop parts; route messages | `parthost` |
| Runner layer | Process groups, PTYs, containers, SSH sessions | `runner` |
| Trigger host | Trigger parts → new runs, with dedupe, debounce, concurrency policy | `trigger` |
| Run store | Event log, items, artifacts, rebuildable SQLite index | `store` |
| Workspace manager | Worktrees, temp dirs, volumes, path safety | `workspace` |
| Secrets broker | Resolve by reference, inject where declared, register for redaction | `secrets` |
| Expression engine | CEL `${{ }}`, sandboxed Lua transforms | `expr`, `luaapi` |
| Gateway (optional) | HTTP/WebSocket: webhooks, remote API | `gateway` |

## Main flow: a run, trigger to result

1. A trigger fires (or `mill run` sends `StartRun`).
2. The loader compiles the pipeline at a fixed content hash; parts resolve from the lockfile; edges are inferred; lint runs. The hash goes on the run.
3. The scheduler queues ready node-runs. One starts only when it holds every slot it needs: global, per-runner, per-agent, run budget.
4. The part host opens a context on the part with resolved `with`, workspace, granted secrets, budget and a scratch dir. Items flow in; items, logs, progress, usage and artifacts flow out.
5. Every change is appended to the run's `events.jsonl`; items and raw output are written beside it; the index is updated.
6. Clients subscribed over the socket get `RunUpdated`, `NodeRunUpdated` and related events.
7. The run reaches a terminal state. Outputs go to stdout when mill is used as a filter. Workspaces follow their cleanup policy.

## Package boundaries

Rules from the accepted layout ([backend plan §2–3](../plans/backend/architecture.md), spec §27.4):

1. Lower layers never import higher ones. Base types and errors → language (pipeline, expr, protocol) → services (store, runner, secrets, config) → execution (scheduler, part host, built-ins, triggers) → hosts (daemon server, gateway) → surface (CLI, TUI).
2. The scheduler and engine see parts only through the parts interface. Built-ins are registered by the part host, never imported by the engine.
3. Built-in parts never import the engine or the store. Everything they need arrives in the `Open` context.
4. The TUI never reads run directories; it goes through the daemon. The CLI may read run files read-only (`mill runs --store`, `mill logs` with the daemon down).
5. Only the daemon boot code and the `--local` entry construct an engine.

`engine`, `store`, `runner`, `trigger`, `workspace` and `secrets` live outside `daemon/` so `--local` can use them without the daemon (accepted 2026-11-11, ADR-044).

## Invariants

From [spec §5.4](../spec/05-architecture.md#54-invariants). Each will have a test.

1. One JSON object per line on every socket and every part's stdio; `v` must match.
2. No broadcast reaches a connection before its `Hello` completes.
3. Binary data on the wire is base64.
4. Run logs are append-only; corrections are new events.
5. Raw step output is stored as-is (minus redacted secrets; backend plan S1). Display uses sanitized copies.
6. A node-run in a terminal state never changes state.
7. Slots are acquired before a node-run starts and released at its terminal state, on every path.
8. Cancel and delete share one teardown path. No transient `failed` during teardown.
9. Resolved secret values never appear in run logs, items, artifacts or client events.
10. Reloading pipelines never changes a run in progress.

## Concurrency (planned)

From the backend plan: one run actor goroutine per live run is the only writer of its event log; one scheduler loop owns every slot; components talk over channels. Every timer goes through an injected clock so tests use fake time.

## External systems

| System | Through | Role |
|--------|---------|------|
| Agent CLIs | agent registry + runners | Child processes, headless or in a PTY |
| MCP servers | part host | Tools exposed as `mcp:<server>/<tool>` nodes; `mill mcp serve` exposes mill |
| git, `gh` | built-in parts | Worktrees, commits, PRs |
| Docker / Podman | `container` runner | Isolation |
| Secret stores | secrets backend parts | env, keychain, 1Password, Vault, age file |
| rex | official `mill/rex` plugin only | Handoff target, runner, trigger. Core knows nothing about rex |
