# Repository structure

mill is a **two-binary Go application**: `mill` (CLI + Bubble Tea TUI) and `milld` (daemon). Clients talk to the daemon over newline-delimited JSON on a local socket. Parts (plugins) talk to the daemon with the same envelope over stdio.

This document is the map for where docs and code go. **No code exists yet**: the code layout below is the backend plan's layout ([plans/backend/architecture.md](plans/backend/architecture.md) §2), accepted on 2026-11-11 and mirrored in [spec §27.4](spec/27-tech-stack.md#274-repository-layout). Once code exists, Go source wins on any conflict and this file gets fixed.

## Project kind

| Aspect | Detail |
|--------|--------|
| Binaries | `cmd/mill`, `cmd/milld` (planned) |
| Module | `github.com/TristanBietsch/mill` (repo of the same name, public; name check resolved, spec §30 item 12) |
| License | MIT ([LICENSE](../LICENSE)) |
| Public API | None outside the module (`internal/` only), except the parts SDKs under `sdk/` |
| Build | `CGO_ENABLED=0`, one static binary per platform |

## Top-level layout

Today:

```
mill/
├── README.md  CONTRIBUTING.md  CHANGELOG.md  SECURITY.md  AGENTS.md  TODO.md  LICENSE
├── docs/
│   ├── index.md            # documentation map
│   ├── design.md           # product and UX overview (one page)
│   ├── tech.md             # tech overview (one page)
│   ├── STRUCTURE.md        # this file
│   ├── spec/               # the specification (authoritative until code exists)
│   ├── plans/              # implementation plans (backend/, editor.md, …)
│   ├── agents/             # onboarding and module map for coding agents
│   └── *.md                # user reference (stubs until features ship)
└── mockups/                # HTML mockups of the TUI
```

Planned (backend plan architecture §2, spec §27.4):

```
mill/
├── cmd/
│   ├── mill/               # client: CLI + TUI (package main)
│   └── milld/              # daemon (package main)
├── internal/               # all library code
├── sdk/{go,ts,python}/     # parts SDKs
├── third_party/vt10x/      # terminal emulator fork (from rex)
├── plugins/                # official plugins (github, slack, linear, rex …), each a normal plugin
├── examples/               # example pipelines; also the end-to-end test suite
├── testdata/               # repo-root fixtures
├── docs/
├── Makefile  go.mod  .golangci.yml
```

## Domain tree (`internal/`, planned)

Library code lives outside `daemon/` unless it only exists because there's a daemon: the engine, store, runners, workspaces, secrets and triggers are also used by `mill run --local` and the weekend prototype. Layering rules (what may import what) are in [plans/backend/architecture.md](plans/backend/architecture.md) §3.

```
internal/
├── core/  errs/  clock/  paths/  mlog/     # base: plain data, the §29 error object, time, paths, logging
├── wire/
│   ├── jsonl/              # envelope + codec, shared by the daemon and parts protocols
│   ├── protocol/           # daemon intents and events
│   └── client/             # socket dial, handshake, request/ack
├── partproto/              # parts protocol messages, Part interface, transports, conformance suite
├── pipeline/               # parse, lint, edge inference, compile
│   ├── ir/                 # the compiled graph
│   ├── pipe/               # pipe-expression parser
│   ├── edit/               # edit operations and quick fixes (CLI, daemon, TUI)
│   ├── yamledit/           # text-splice YAML edits
│   └── layout/             # slots, pins, edge routing into a cell grid
├── expr/  schema/          # CEL environment; output shorthand → JSON Schema
├── engine/                 # run actor, reducer, readiness
│   └── sched/ budget/ retry/ cache/
├── store/                  # run dirs, event log, items, logs
│   └── cas/ index/ lock/
├── parthost/               # part registry and resolution, process pool, health, PartCall
├── parts/builtin/          # sh, exec, check, data, control, agent*, git, gh, http, mcp, file …
├── agents/  agents/detect/ # agent registry; terminal-agent state detection
├── runner/                 # local, pty, container, ssh; process groups per OS
├── workspace/              # worktrees, temp dirs, cleanup, retry snapshots, path safety
├── secrets/  secrets/redact/
├── trigger/                # trigger host + one package per kind
├── config/  config/settings/
├── luaapi/                 # the mill.* Lua API, sandboxes, LuaLS type generation
├── plugins/                # resolver, installer, manifest, permissions, lockfile, pack, import/export
├── daemon/
│   ├── server/             # socket handlers, per-connection queues
│   ├── boot/               # wiring: lock, config, recovery, engine, triggers, server
│   └── status/             # status file for mill prompt and completion
├── daemonctl/              # find, start and reach milld
├── gateway/                # HTTP + WebSocket front door (later)
├── surface/
│   ├── cli/                # subcommands, selectors, exit codes, output modes; cli/core output kit; cli/local
│   ├── tui/                # boot splash, board, run popup, log, editor, settings
│   │   └── canvas/  editor/  settings/
│   ├── sound/              # soundsets, event mixer, debounce, silent fallback
│   └── mcpserver/          # mill mcp serve
└── testutil/               # test env, golden files, fakes, echo agent harness
```

The spec's earlier tree (with `sched`, `store`, `runner`, `trigger`, `workspace` and `secrets` under `daemon/`, and the pipe parser under `surface/`) is superseded. Module docs in [agents/modules/](agents/modules/overview.md) give the backend path first.

### Rationale (one line each)

| Directory | Why it exists |
|-----------|----------------|
| `core/`, `errs/`, `clock/`, `paths/`, `mlog/` | Base layer every package may import |
| `wire/`, `partproto/` | One envelope and codec shared by clients, daemon and parts |
| `pipeline/`, `expr/`, `schema/` | The file format and expressions, independent of how runs execute |
| `engine/`, `store/`, `runner/`, `workspace/`, `secrets/`, `trigger/` | Execution as a library, hosted by `milld` and by `mill run --local` |
| `parthost/`, `parts/builtin/`, `plugins/` | Everything mill can do comes from parts; built-ins use the public protocol |
| `config/`, `luaapi/` | Lua config and the one extension language |
| `daemon/`, `daemonctl/`, `gateway/` | Only what exists because there's a daemon |
| `surface/` | Everything the user runs directly (`mill`, the TUI, sound, the MCP server) |

## Where new code goes (planned)

```
Need to…                              → Put it in…
───────────────────────────────────────────────────────────────────────
Add a CLI subcommand                    → surface/cli/ (+ register in cmd/mill)
Add TUI key, view or popup              → surface/tui/
Change a wire message                   → wire/protocol/ (+ docs/agents/data/schemas.md, spec §16)
Change the parts protocol               → partproto/ (+ spec §9.3)
Add a daemon intent handler             → daemon/server/
Add a built-in part                     → parts/builtin/ (+ spec §10, its manifest flags)
Add a runner or trigger                 → runner/ or trigger/ (they are parts too)
Change scheduling, retries or budgets   → engine/ (+ spec §15)
Change the pipeline format or lint      → pipeline/ (+ spec §7, bump `mill:` only on a meaning change)
Add a CEL helper                        → expr/
Add a setting                           → config/settings/ only
Add a built-in agent entry              → agents/
Add a Lua API function                  → luaapi/ (+ LuaLS types)
Add an error code                       → errs/ (+ explain page + golden test, spec §29.11)
Add a sound                             → surface/sound/
```

## Conventions

- **Go filenames:** `snake_case.go` and `*_test.go`.
- **Go directories:** lowercase, match the package name. Exception: none planned (`milld` has no hyphen).
- **Tests:** co-located `*_test.go`; CLI scripts as `testdata/script/*.txtar`; end-to-end in `examples/`. Tiers in [CONTRIBUTING](../CONTRIBUTING.md#test-tiers-planned).
- **Shared types:** no `internal/types/`. Plain shared data lives in `core/`, wire shapes in `wire/protocol`. Promote a type only when two or more domains share it.

Full conventions: [agents/CONVENTIONS.md](agents/CONVENTIONS.md).

## Documentation

| Path | What goes there | Audience |
|------|-----------------|----------|
| `README.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `SECURITY.md` | Front door, rules, user-visible changes, threat model | Everyone |
| `AGENTS.md` (root) | Pointer to `docs/agents/HANDOFF.md` | Agents |
| [docs/index.md](index.md) | Map of every doc and the authority order | Everyone |
| [docs/spec/](spec/index.md) | The specification, one file per section. Changed only on purpose, by the user or with their accept | Everyone |
| [docs/plans/](plans/README.md) | How to build something: milestones, tasks, decisions. Naming in plans/README.md | Implementers |
| `docs/*.md` | User reference per surface (cli, tui, pipeline-format, …). Describes shipped behavior only | Users |
| `docs/howto/` | One task per file, written when the feature ships | Users |
| `docs/errors/` | Generated explain pages, one per M-code (spec §29.5). Never hand-edited | Users |
| [docs/agents/](agents/README.md) | Onboarding, module map, interfaces, schemas, decisions | Agents, contributors |
| `mockups/` | Visual reference for the TUI | Designers, TUI implementers |

Rules:

- A user doc describes what ships. Until then it is a stub pointing to the spec.
- A behavior change updates, in the same PR: the user doc, the module doc, `CHANGELOG.md`, and the spec when the change departs from it.
- Agent docs are navigation. They link to user docs and the spec instead of copying their tables.

## Verification (planned)

```sh
make build
make test
make check   # vet + race tests + lint (host + windows) + coverage floors + govulncheck + errors-check + layering
```

Smoke: `mill run examples/hello.mill.yaml --local`, `mill status`, `milld -version`.
