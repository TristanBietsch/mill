# Glossary

mill's terms, alphabetical. Source: [spec §4](spec/04-concepts.md#4-concepts-and-glossary), plus terms defined elsewhere in the spec (linked). See [index.md](index.md) for the documentation map.

## agent

An agent CLI described in the agent registry (Claude Code, Codex, Gemini CLI, Aider, OpenCode, Ollama, or anything that runs in a terminal). mill runs it unattended. Also the `agent` part that runs one. See [spec §11](spec/11-agents.md#11-agents).

## artifact

A file produced by a node (a diff, a report, an image, a build output). Stored once by SHA-256 and referenced from items. Downstream nodes get a read-only local path. See [spec §6.3](spec/06-data-model.md#63-artifacts).

## assert

A node that checks a condition on items and fails, routes or drops them when it is false. The automated replacement for an approval.

## auto_answer

Agent registry rules that answer prompts a terminal-only agent is known to show (for example a folder-trust dialog in a trusted folder). Any other question fails the node with `M6304`. See [spec §11.4](spec/11-agents.md#114-agents-that-need-a-terminal).

## board

The TUI's one screen: every pipeline, one line each, in FAILED, RUNNING and READY sections. Everything else opens over it. See [spec §18.2](spec/18-tui.md#182-the-board).

## boot splash

What a bare `mill` shows while it starts the TUI: one `[ STATUS ] step · finding` row per boot step, then the board. See [spec §18.13](spec/18-tui.md#1813-boot-splash).

## budget

Limits for a run: tokens, money, wall time, node count, concurrent agents. Hitting one stops the run with reason `budget` (exit 9). See [spec §15.5](spec/15-execution.md#155-budgets).

## bundle

A plugin that mostly contains pipelines. What `mill export` produces and `mill import` consumes.

## cache

Reuse of a node-run's stored output when its key (part, config, inputs) matches. A reused node-run has state `cached`. See [spec §15.4](spec/15-execution.md#154-caching).

## destructive

A part capability flag for actions with outside effects: push, merge, send, delete, pay. A pipeline may use a destructive part only if `policy.allow_destructive` lists it. `--dry-run` skips them. See [spec §22](spec/22-security.md#22-security-and-trust).

## edge

A connection from one port to another. It carries a stream of items. Usually inferred from references. See [spec §7.4](spec/07-pipeline-format.md#74-edge-inference).

## error class

The category of an error (`user`, `not_found`, `transient`, `rate_limited`, `budget`, …). Retries, exit codes and the TUI key off the class, never the message. See [spec §29.6](spec/29-errors.md#296-error-classes).

## error code (M-code)

`M` plus four digits, grouped by area (`M3012` unknown node in expression). Never reused or renumbered. `mill explain <code>` prints its page. See [spec §29.4](spec/29-errors.md#294-error-codes).

## gateway

The optional HTTP and WebSocket front door of `milld`: webhook triggers and the remote API. Off by default; binds to localhost when on.

## handoff

A node that passes work to a person or another tool (an issue comment, a message, a ticket, a rex session) and ends that branch. mill never waits for the answer. If the answer should continue the work, it arrives later as a trigger.

## hands-off

The first principle: a run never waits on a person. No approvals, questions or takeovers inside mill. See [spec §2](spec/02-principles.md#2-principles).

## io

How an exec-style node sees items: `envelope` (the full item), `plain` (only `data`, one JSON value per line) or `text` (raw lines). See [spec §6.1](spec/06-data-model.md#61-items).

## item

One unit of data on an edge: a JSON value (`data`), plus optional files, a workspace reference (`ws`) and lineage metadata (`meta`). See [spec §6.1](spec/06-data-model.md#61-items).

## layout sidecar

`<name>.mill.layout.json` next to a pipeline file. Holds canvas positions so moving a box never changes the pipeline file. Safe to delete. See [spec §7.6](spec/07-pipeline-format.md#76-layout-sidecar).

## lockfile

`mill.lock` (project) or the user lock: exact plugin versions and hashes of archives and executables. See [spec §9.11](spec/09-parts-and-plugins.md#911-lockfile).

## mill, milld

The two binaries. `mill` is the client (CLI, TUI, pipe expressions). `milld` is the daemon: it loads pipelines, hosts parts, runs triggers, schedules node-runs and writes the run store. See [spec §5.1](spec/05-architecture.md#51-processes).

## mode

How a node consumes items: `batch` (one invocation gets all items, the default), `each` (one node-run per item) or `stream` (one long-lived invocation). See [spec §6.2](spec/06-data-model.md#62-node-modes).

## node

One step in a pipeline. It `uses` a part and configures it `with` settings.

## node-run

One execution of one node within a run. Nodes in `each` mode produce one per item. A retry is a new node-run.

## part

One capability the engine uses: a node type, trigger, runner, secrets backend, sink or artifact store. Built-ins and plugins use the same parts protocol. See [spec §9](spec/09-parts-and-plugins.md#9-parts-and-plugins).

## part host

The `milld` component that starts, pools, health-checks and stops part processes and routes messages to them. See [spec §5.2](spec/05-architecture.md#52-components).

## pipe expression

A one-line `a | b | c` pipeline written in the shell (`mill pipe '…'`). Compiles to the same graph as YAML. See [spec §14.2](spec/14-piping.md#142-pipe-expressions).

## pipeline

A named, versioned graph defined in a `*.mill.yaml` file. Its id is its `name`. See [spec §7](spec/07-pipeline-format.md#7-pipeline-file-format).

## plugin

An installable, shareable package containing parts, MCP server and agent definitions, pipelines, prompts and config defaults. See [spec §9.5](spec/09-parts-and-plugins.md#95-plugin-package-format).

## policy

A pipeline's safety rules: which destructive capabilities it may use (`allow_destructive`), allowed runners, network rules, `fail_fast`.

## port

A named input or output on a node. The defaults are `in` and `out`. Branching parts add more (`true`/`false`, `case.*`, `else`, `error`).

## run

One execution of a pipeline. Has an id, a per-pipeline number, inputs, a state, node-runs, artifacts and a budget ledger. Runs a fixed pipeline version (content hash).

## run store

Where runs live on disk: an append-only `events.jsonl` per run, items, raw logs, artifacts, and a SQLite index that can always be rebuilt from the files. See [spec §20](spec/20-paths.md#20-files-and-paths).

## runner

Where a node-run executes: local process, PTY, container, SSH host, remote `milld`, or an external session manager. See [spec §12](spec/12-runners.md#12-runners).

## selector

How a run is referenced on the CLI: full id, `pipeline#number`, `pipeline#last`, a short hex id (4+ digits), or `run/node[n]` for a node. See [spec §17.3](spec/17-cli.md#173-selectors).

## soundset

A named set of sounds for TUI and run events (`millrace`, `foundry`, `chiptune`, `bell`, `off`). See [spec Appendix A](spec/appendix-a-soundsets.md#appendix-a-reference-soundsets).

## trigger

Something that starts runs: manual, cron, webhook, file watch, git, poll, stdin, MCP call, another run's event. See [spec §13](spec/13-triggers.md#13-triggers).

## trust

Explicit approval, recorded by file hash, for pipelines, plugins and project config that came from elsewhere. `mill trust <file|dir>`. A changed file needs trust again.

## workspace

An isolated working directory (git worktree, temp dir or container volume) owned by a run or a node. Travels on items as `ws`. See [spec §6.4](spec/06-data-model.md#64-workspaces).

## States

Runs and node-runs share one vocabulary. Terminal states never change; only a new node-run (a retry) can follow one. Source: [spec §4 States](spec/04-concepts.md#states).

| State | Run | Node-run | Meaning |
|---|---|---|---|
| `pending` | | ✓ | Not ready yet (waiting on inputs) |
| `queued` | ✓ | ✓ | Ready, waiting for capacity |
| `working` | ✓ | ✓ | Executing |
| `retrying` | | ✓ | Failed with a retryable error and waiting out its backoff |
| `done` | ✓ | ✓ | Finished successfully (terminal) |
| `failed` | ✓ | ✓ | Finished with an error (terminal) |
| `skipped` | | ✓ | On a branch not taken, or `when` was false (terminal) |
| `cached` | | ✓ | Output reused from the cache; not executed (terminal) |
| `cancelled` | ✓ | ✓ | Stopped by a person, a policy or a budget (terminal) |
| `crashed` | ✓ | ✓ | The daemon died while it was live and it could not be resumed or safely re-run (terminal) |
