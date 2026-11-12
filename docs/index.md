# Documentation index

mill is a terminal-native, unattended pipeline engine for agents, code and automations. Two binaries: `mill` (CLI and TUI) and `milld` (daemon). They talk JSONL over a local socket.

Status: spec stage. No code yet.

## Authority

Which document wins when two disagree:

| Stage | Order (first wins) |
|-------|--------------------|
| Now (no code) | 1. [spec](spec/index.md) · 2. plans in [plans/](plans/README.md) · 3. everything else |
| Once code exists | 1. Go source · 2. spec · 3. plans · 4. everything else |

When code and spec disagree, the code wins and the spec gets fixed in the same PR (or a follow-up noted in the PR). Do not change code to match the spec without the user's intent. A plan may propose a spec change; it applies only once the user accepts it and the spec is edited.

## By audience

| Audience | Start here |
|----------|------------|
| First-time reader | [README](../README.md), then [spec summary](spec/01-summary.md) |
| Coding agent | [agents/HANDOFF.md](agents/HANDOFF.md) |
| Contributor | [CONTRIBUTING](../CONTRIBUTING.md), [STRUCTURE.md](STRUCTURE.md), [agents/](agents/README.md) |
| Implementer of an area | [plans/](plans/README.md), then the spec section |
| Security reviewer | [SECURITY](../SECURITY.md), [spec §22](spec/22-security.md) |

## Spec

The spec is split into one file per section under [spec/](spec/index.md). Start at [spec/index.md](spec/index.md).

| File | Section |
|------|---------|
| [spec/index.md](spec/index.md) | Title, status, what changed in v2, contents |
| [01-summary.md](spec/01-summary.md) | What mill is |
| [02-principles.md](spec/02-principles.md) | The 16 principles that decide arguments |
| [03-goals.md](spec/03-goals.md) | Goals and non-goals |
| [04-concepts.md](spec/04-concepts.md) | Concepts, glossary, states |
| [05-architecture.md](spec/05-architecture.md) | Processes, components, design decisions, invariants |
| [06-data-model.md](spec/06-data-model.md) | Items, node modes, artifacts, workspaces |
| [07-pipeline-format.md](spec/07-pipeline-format.md) | `*.mill.yaml` format, edge inference, layout sidecar, lint |
| [08-expressions.md](spec/08-expressions.md) | `${{ }}` CEL expressions |
| [09-parts-and-plugins.md](spec/09-parts-and-plugins.md) | Parts protocol, plugins, import/export, lockfile |
| [10-builtin-parts.md](spec/10-builtin-parts.md) | Built-in parts catalog |
| [11-agents.md](spec/11-agents.md) | Agent registry, unattended agents, structured output |
| [12-runners.md](spec/12-runners.md) | Where node-runs execute |
| [13-triggers.md](spec/13-triggers.md) | What starts runs |
| [14-piping.md](spec/14-piping.md) | mill as a filter, pipe expressions |
| [15-execution.md](spec/15-execution.md) | Scheduling, retries, cache, budgets, teardown, recovery |
| [16-wire-protocol.md](spec/16-wire-protocol.md) | Daemon wire protocol |
| [17-cli.md](spec/17-cli.md) | CLI reference, exit codes, selectors, output modes |
| [18-tui.md](spec/18-tui.md) | TUI reference |
| [19-config-lua.md](spec/19-config-lua.md) | Lua config, API, settings registry |
| [20-paths.md](spec/20-paths.md) | Files and paths |
| [21-secrets.md](spec/21-secrets.md) | Secrets |
| [22-security.md](spec/22-security.md) | Security and trust |
| [23-integrations.md](spec/23-integrations.md) | MCP, HTTP, n8n, Zapier, rex |
| [24-observability.md](spec/24-observability.md) | Logs, traces, cost |
| [25-hosted.md](spec/25-hosted.md) | Optional hosted layer |
| [26-examples.md](spec/26-examples.md) | Example pipelines |
| [27-tech-stack.md](spec/27-tech-stack.md) | Go, libraries, repository layout |
| [28-roadmap.md](spec/28-roadmap.md) | Phases v0 to v1, weekend prototype |
| [29-errors.md](spec/29-errors.md) | Error rules, M-codes, classes, `mill explain` |
| [30-open-questions.md](spec/30-open-questions.md) | Open questions |
| [appendix-a-soundsets.md](spec/appendix-a-soundsets.md) | Reference soundsets |
| [appendix-b-walkthrough.md](spec/appendix-b-walkthrough.md) | Install to result walkthrough |

## Reference (user docs)

Each file is a stub until its feature ships. The stub points to the spec section that is the reference until then.

| Document | Purpose |
|----------|---------|
| [quickstart.md](quickstart.md) | Install to first run |
| [cli.md](cli.md) | `mill` commands, flags, exit codes, selectors |
| [tui.md](tui.md) | Board, keys, run popup, editor |
| [pipeline-format.md](pipeline-format.md) | `*.mill.yaml` keys and rules |
| [parts-protocol.md](parts-protocol.md) | Writing an executable part |
| [plugin-format.md](plugin-format.md) | `plugin.lua`, packages, lockfile |
| [lua-api.md](lua-api.md) | The `mill.*` Lua API |
| [protocol.md](protocol.md) | Daemon wire protocol |
| [agents.md](agents.md) | Agent registry entries |
| [runners.md](runners.md) | Runners |
| [triggers.md](triggers.md) | Triggers |
| [settings.md](settings.md) | Settings keys |
| [paths.md](paths.md) | Files, directories, environment variables |
| [security.md](security.md) | Trust, secrets, sandboxes for users |
| [glossary.md](glossary.md) | Terms (complete) |

## Guides (planned)

No files yet. Planned in [spec §27.4](spec/27-tech-stack.md#274-repository-layout) under `howto/`: `write-a-plugin`, `write-a-lua-part`, `add-an-agent`, `import-export`, `pipe-recipes`, `headless-ci`, `webhooks`, `mcp`. Each is written when its feature ships.

## Plans

| Document | Purpose |
|----------|---------|
| [plans/README.md](plans/README.md) | Naming, numbering, plan format, index of plans |
| [plans/backend/00-overview.md](plans/backend/00-overview.md) | Backend and tech-stack plan: engine, storage, protocols, milestones M0–M11 |
| [plans/editor.md](plans/editor.md) | TUI pipeline editor plan |

## Agent docs

| Document | Purpose |
|----------|---------|
| [agents/HANDOFF.md](agents/HANDOFF.md) | Start here: state, gate, open work |
| [agents/README.md](agents/README.md) | Module map, where to start per task |
| [agents/AGENTS.md](agents/AGENTS.md) | Authority, safe vs careful edits, never-do list |

## Layout and design

| Document | Purpose |
|----------|---------|
| [STRUCTURE.md](STRUCTURE.md) | Where each kind of doc and code goes |
| [agents/ARCHITECTURE.md](agents/ARCHITECTURE.md) | Processes and package boundaries |
| [`mockups/`](../mockups/) | HTML mockups of the TUI (v1). Visual reference only; the spec wins on behavior |
