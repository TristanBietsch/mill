# Modules overview

Every planned package, from [spec §27.4](../../spec/27-tech-stack.md#274-repository-layout). **Status of all: planned.** No code exists.

Each module doc has the same fields: path, status and first milestone, the backend plan's path for it, dependencies, purpose, what it owns, invariants, spec links. When code lands, the doc is rewritten from source in rex's module-doc shape.

## Map

| Layer | Modules |
|-------|---------|
| Entrypoints | [cmd](cmd.md) |
| Surface | [cli](cli.md), [tui](tui.md), [canvas](canvas.md), [tui-settings](tui-settings.md), [sound](sound.md), [pipe](pipe.md) |
| Wire | [protocol](protocol.md), [client](client.md) |
| Daemon | [server](server.md), [sched](sched.md), [store](store.md), [runner](runner.md), [trigger](trigger.md), [workspace](workspace.md), [secrets](secrets.md) |
| Config and Lua | [config](config.md), [luaapi](luaapi.md), [settings](settings.md) |
| Parts | [plugins](plugins.md), [parthost](parthost.md), [builtin](builtin.md), [agents](agents.md) |
| Language | [pipeline](pipeline.md), [expr](expr.md) |
| Outside `internal/` | [sdk](sdk.md), [examples](examples.md) |

Official plugins (`plugins/` at the repo root: github, slack, linear, rex …) are normal plugins and get a doc each when they exist.

## Dependency direction (planned)

```
cmd → cli, tui → client → protocol
cmd (milld) → server → sched → parthost → builtin → (runner, workspace, secrets via Open)
                     → store
                     → trigger
          pipeline → expr
          config → luaapi, settings
          plugins → luaapi
```

Lower layers never import higher ones; built-ins never import the scheduler or store. Full rules: [../ARCHITECTURE.md](../ARCHITECTURE.md#package-boundaries).

## What the backend plan changes

[../../plans/backend/architecture.md](../../plans/backend/architecture.md) §2 refines this layout. Pending the user's accept; until then the spec's paths above are the contract.

| Change | Reason given |
|--------|--------------|
| `daemon/sched` → `engine` (+ `engine/sched`, `budget`, `retry`, `cache`) | `--local` and the prototype must run without a daemon |
| `daemon/store`, `runner`, `trigger`, `workspace`, `secrets` → top-level `internal/` | Same |
| `surface/pipe` → `pipeline/pipe` | The daemon compiles pipe expressions (`Compile` intent) |
| `catalog/settings` → `config/settings`; `catalog/agents` → `agents`, `agents/detect` | Grouped with their loaders |
| New: `core` | Shared plain types: ids, states and transitions, items, refs |
| New: `errs` | The §29 error object, codes, classes, explain pages, exit-code map |
| New: `clock`, `paths`, `mlog` | Fake time, one source of paths, file logging |
| New: `wire/jsonl` | Payload-agnostic codec shared by both protocols |
| New: `partproto` | Parts protocol types, transports, conformance suite |
| New: `schema` | Node `output` shorthand → JSON Schema |
| New: `daemonctl`, `daemon/boot`, `daemon/status`, `gateway`, `surface/mcpserver`, `testutil` | Daemon start/find, wiring, prompt status file, HTTP front door, `mill mcp serve`, test helpers |
| New: `third_party/vt10x` | rex's terminal emulator fork for PTY screens |

If the user accepts the backend layout, rename the module docs to match and add docs for the new packages.
