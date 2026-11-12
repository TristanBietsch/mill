# Repository structure

mill is a **two-binary Go application**: `mill` (CLI + Bubble Tea TUI) and `milld` (daemon). Clients talk to the daemon over newline-delimited JSON on a local socket. Parts (plugins) talk to the daemon with the same envelope over stdio.

This document is the map for where docs and code go. **No code exists yet**: the code layout below is the plan from [spec §27.4](spec/27-tech-stack.md#274-repository-layout). Once code exists, Go source wins on any conflict and this file gets fixed.

## Project kind

| Aspect | Detail |
|--------|--------|
| Binaries | `cmd/mill`, `cmd/milld` (planned) |
| Module | `github.com/tristanbietsch/mill` (assumed; spec §30 item 12 asks for a name check first) |
| Public API | None outside the module (`internal/` only), except the parts SDKs under `sdk/` |
| Build | `CGO_ENABLED=0`, one static binary per platform |

## Top-level layout

Today:

```
mill/
├── README.md  CONTRIBUTING.md  CHANGELOG.md  SECURITY.md  AGENTS.md  TODO.md
├── docs/
│   ├── index.md            # documentation map
│   ├── STRUCTURE.md        # this file
│   ├── spec/               # the specification (authoritative until code exists)
│   ├── plans/              # implementation plans (backend/, editor.md, …)
│   ├── agents/             # onboarding and module map for coding agents
│   └── *.md                # user reference (stubs until features ship)
└── mockups/                # HTML mockups of the TUI
```

Planned (spec §27.4):

```
mill/
├── cmd/
│   ├── mill/               # client: CLI + TUI (package main)
│   └── milld/              # daemon (package main)
├── internal/               # all library code
├── sdk/{go,ts,python}/     # parts SDKs
├── plugins/                # official plugins (github, slack, linear, rex …), each a normal plugin
├── examples/               # example pipelines; also the end-to-end test suite
├── testdata/               # repo-root fixtures
├── docs/
├── Makefile  go.mod  .golangci.yml
```

## Domain tree (`internal/`, planned)

```
internal/
├── surface/                # what the user runs directly
│   ├── cli/                # subcommands, selectors, exit codes, output modes
│   ├── tui/                # boot splash, board, run popup, log, editor, settings
│   │   ├── canvas/         # cell-grid renderer, zoom levels, hit-testing
│   │   └── settings/       # settings page generated from the registry
│   ├── sound/              # soundsets, event mixer, debounce, silent fallback
│   └── pipe/               # pipe-expression parser
├── wire/
│   ├── protocol/           # envelope, intents, events, codec (shared with parts protocol)
│   └── client/             # socket dial, handshake, request/ack
├── daemon/
│   ├── server/             # intent handlers, per-connection queues
│   ├── sched/              # ready queue, slots, budgets
│   ├── store/              # event log, items, artifacts, index
│   ├── runner/             # local, pty, container, ssh, remote, external
│   ├── trigger/            # trigger host, dedupe, concurrency policy
│   ├── workspace/          # worktrees, temp dirs, cleanup, path safety
│   └── secrets/            # broker, redactor
├── config/                 # Lua loader, layers and precedence, settings.lua writer, reload
├── luaapi/                 # the mill.* Lua API, sandboxes, LuaLS type generation
├── plugins/                # resolver, installer, manifest, permissions, lockfile, pack, import/export
├── parthost/               # process pool, Lua part host, lifecycle, routing
├── parts/builtin/          # sh, exec, check, data, control, agent*, git, gh, http, mcp, file …
├── pipeline/               # parse, lint, edge inference, compile, YAML editing, layout
├── expr/                   # CEL environment and helpers
└── catalog/
    ├── settings/           # settings registry
    └── agents/             # agent registry + merge
```

**Pending layout decision.** The backend plan refines this tree ([plans/backend/architecture.md](plans/backend/architecture.md) §2): it moves `sched`, `store`, `runner`, `trigger`, `workspace` and `secrets` out of `daemon/` (so `mill run --local` can use them without a daemon), moves the pipe parser to `pipeline/pipe`, and adds `core`, `errs`, `clock`, `paths`, `mlog`, `engine`, `partproto`, `agents`, `daemonctl` and `gateway`. Until the user accepts that, the spec tree above is the contract. Module docs in [agents/modules/](agents/modules/overview.md) list both paths.

### Rationale (one line each)

| Directory | Why it exists |
|-----------|----------------|
| `surface/` | Everything the user runs directly (`mill`, the TUI, pipe expressions) |
| `wire/` | One envelope and codec shared by clients, daemon and parts |
| `daemon/` | Long-lived work: scheduling, storage, processes, triggers |
| `config/`, `luaapi/` | Lua config and the one extension language |
| `plugins/`, `parthost/`, `parts/builtin/` | Everything mill can do comes from parts; built-ins use the public protocol |
| `pipeline/`, `expr/` | The file format and expressions, independent of how runs execute |
| `catalog/` | Registries that drive several surfaces from one definition |

## Where new code goes (planned)

```
Need to…                              → Put it in…
───────────────────────────────────────────────────────────────────────
Add a CLI subcommand                    → surface/cli/ (+ register in cmd/mill)
Add TUI key, view or popup              → surface/tui/
Change a wire message                   → wire/protocol/ (+ docs/agents/data/schemas.md, spec §16)
Add a daemon intent handler             → daemon/server/
Add a built-in part                     → parts/builtin/ (+ spec §10, its manifest flags)
Add a runner or trigger                 → daemon/runner/ or daemon/trigger/ (they are parts too)
Change the pipeline format or lint      → pipeline/ (+ spec §7, bump `mill:` only on a meaning change)
Add a CEL helper                        → expr/
Add a setting                           → catalog/settings/ only
Add a built-in agent entry              → catalog/agents/
Add a Lua API function                  → luaapi/ (+ LuaLS types)
Add an error code                       → the errors package (+ explain page + golden test, spec §29.11)
Add a sound                             → surface/sound/
```

## Conventions

- **Go filenames:** `snake_case.go` and `*_test.go`.
- **Go directories:** lowercase, match the package name. Exception: none planned (`milld` has no hyphen).
- **Tests:** co-located `*_test.go`; CLI scripts as `testdata/script/*.txtar`; end-to-end in `examples/`. Tiers in [CONTRIBUTING](../CONTRIBUTING.md#test-tiers-planned).
- **Shared types:** no `internal/types/`. Wire shapes live in `wire/protocol`. Promote a type there only when two or more domains share it.

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
