# Decisions

Choices already made in the spec, as a log an agent can check before undoing something. Each entry: date, status, context, decision, consequences, source.

All entries below were recorded on 2026-11-11 from spec v2. Status `accepted` means the spec states it as decided. Open choices are not here: they live in [../plans/backend/open-decisions.md](../plans/backend/open-decisions.md) and [spec §30](../spec/30-open-questions.md#30-open-questions). To add a decision: append the next number, never renumber; to reverse one, add a new entry that supersedes it and mark the old one `superseded by ADR-NNN`.

## ADR-001: Hands-off runs

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Pipelines are only useful as automation if they finish on their own. rex already covers agent work that needs a person.
- **Decision:** A run never waits on a person. No approvals, inboxes, questions or takeovers. Decisions inside a run come from checks, judges, reviews and policies. Work that needs a human goes through a `handoff` node, which records where it went and ends the branch. An answer that should continue the work arrives later as a trigger and starts a new run.
- **Consequences:** No approve/reply commands or input intents. Terminal agents that ask anything unknown fail with `M6304`. `mill run` in a pipe always finishes. `mill status` cannot mean "needs input".
- **Source:** [spec §2](../spec/02-principles.md#2-principles) principle 1, [§5.3](../spec/05-architecture.md#53-design-decisions), [§11.2](../spec/11-agents.md#112-unattended-by-design).

## ADR-002: Two binaries over a local socket

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Runs must outlive the TUI, many clients must connect, and everything must be scriptable.
- **Decision:** `mill` (CLI, TUI, pipe expressions) and `milld` (daemon). `milld` owns child processes, triggers, schedules and disk. `mill` dials a local socket and starts the daemon if it is not running. `mill run --local` runs one pipeline in-process for CI and cron.
- **Consequences:** A client crash never takes down a run. The TUI renders; the daemon owns. Every TUI screen has a CLI equivalent.
- **Source:** [spec §5.1](../spec/05-architecture.md#51-processes), [§5.3](../spec/05-architecture.md#53-design-decisions), principle 4.

## ADR-003: Go

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** A daemon supervising many processes, streams and timers; static binaries for three OSes; CEL, Lua and a TUI ecosystem.
- **Decision:** Go, `CGO_ENABLED=0`, one module, two binaries. Libraries: Bubble Tea and Lip Gloss, `cel-go`, `gopher-lua`, a YAML node API, pure-Go SQLite, `creack/pty` and ConPTY, `fsnotify`, the official Go MCP SDK, `oto`.
- **Consequences:** The canvas needs its own cell-grid component inside Bubble Tea. Exact library versions are a backend plan matter (open decisions D2–D4, D28).
- **Source:** [spec §27.1](../spec/27-tech-stack.md#271-language-go), [§27.2](../spec/27-tech-stack.md#272-libraries).

## ADR-004: Boring formats

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** People and LLMs must read and write every format mill uses.
- **Decision:** JSONL on the wire and in run logs, YAML for pipelines, Lua for config, UTF-8 text, exit codes. No custom binary formats. Binary data on the wire is base64.
- **Consequences:** The protocol is debuggable with `socat` and `jq`. Line length is capped (16 MiB).
- **Source:** [spec §2](../spec/02-principles.md#2-principles) principle 16, [§5.4](../spec/05-architecture.md#54-invariants).

## ADR-005: One JSONL envelope for the daemon and parts

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Two protocols (client ↔ daemon, daemon ↔ part) would need two codecs and two sets of rules.
- **Decision:** One object per line, `{v, kind, type, id, data}`, for both. `v` must match the protocol version. Unknown fields are ignored; unknown types get an error, never a crash.
- **Consequences:** One codec. Wire and parts protocol versions are separate constants that evolve separately. See [ADR-017](#adr-017-additive-evolution).
- **Source:** [spec §16.1](../spec/16-wire-protocol.md#161-envelope), [§9.3](../spec/09-parts-and-plugins.md#93-parts-protocol-v1).

## ADR-006: Parts are processes, not in-process plugins

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Extensions in any language, with crash isolation and independent upgrades.
- **Decision:** Every step type, trigger, runner, agent, secrets backend and sink is a part. Executable parts are child processes speaking the parts protocol over stdio. Lua and (later) WASM parts implement the same lifecycle through function calls.
- **Consequences:** The core is a scheduler plus a part host. In-process speed matters only for a few hot built-ins.
- **Source:** [spec §5.3](../spec/05-architecture.md#53-design-decisions), [§9](../spec/09-parts-and-plugins.md#9-parts-and-plugins), principle 7.

## ADR-007: Built-ins use the public parts protocol

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** A protocol that built-ins bypass goes incomplete.
- **Decision:** Built-in parts get no private internals. Any built-in can be replaced by a user part with the same id.
- **Consequences:** If a built-in needs a capability, the protocol grows it. The backend plan runs built-ins in-process behind the same interface, proven equal by a conformance suite (open decision D12).
- **Source:** [spec §5.3](../spec/05-architecture.md#53-design-decisions), [§10](../spec/10-builtin-parts.md#10-built-in-parts-catalog).

## ADR-008: Event-sourced run log; files are the truth

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Faithful replay, crash recovery and an audit trail.
- **Decision:** Each run has an append-only `events.jsonl`. Nothing rewrites history; corrections are new events. Raw step output is stored as-is. The SQLite index is a cache rebuilt from files (`mill doctor --reindex`).
- **Consequences:** State is a fold over events. Deleting `index.db` loses nothing. Any run can be inspected or replayed exactly.
- **Source:** [spec §5.3](../spec/05-architecture.md#53-design-decisions), [§5.4](../spec/05-architecture.md#54-invariants) invariants 4–5, principles 5 and 11.

## ADR-009: YAML pipelines plus a layout sidecar

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Pipelines must round-trip losslessly through the canvas editor and diff cleanly in git.
- **Decision:** Pipelines are `*.mill.yaml` (plain data). Canvas positions live in `<name>.mill.layout.json`, safe to delete or gitignore. Lua-defined pipelines are allowed but read-only in the canvas and export as YAML.
- **Consequences:** Moving a box never touches the pipeline file. CLI and TUI edits go through `SavePipeline` with a base hash and keep comments and key order.
- **Source:** [spec §5.3](../spec/05-architecture.md#53-design-decisions), [§7.6](../spec/07-pipeline-format.md#76-layout-sidecar), [§19.7](../spec/19-config-lua.md#197-pipelines-in-lua-optional).

## ADR-010: CEL for expressions

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Templating needs to be safe to evaluate inside the daemon.
- **Decision:** `${{ … }}` is CEL: no side effects, always terminates, typed. A string that is one whole expression keeps its type.
- **Consequences:** No arbitrary code in pipeline expressions. Logic beyond CEL goes in a `lua` transform node.
- **Source:** [spec §8](../spec/08-expressions.md#8-expressions-and-templating), [§5.3](../spec/05-architecture.md#53-design-decisions).

## ADR-011: Lua for config, inline scripts and small parts

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Config benefits from logic and comments; one extension language is easier than several.
- **Decision:** All config is Lua (`init.lua`, managed `settings.lua`, project `.mill/mill.lua`, `plugin.lua`). Layers load in a fixed order. Your own config is trusted; project config runs data-only until `mill trust .`; plugin manifests are always data-only.
- **Consequences:** A failed reload keeps the previous config running. One settings registry drives `mill.setup{}`, `settings.lua`, `mill config` and the TUI.
- **Source:** [spec §19](../spec/19-config-lua.md#19-configuration-lua), [§5.3](../spec/05-architecture.md#53-design-decisions).

## ADR-012: Agents described as data, no vendor SDKs

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Agent CLIs change fast; mill must not depend on any vendor.
- **Decision:** Agent CLIs are registry entries (`mill.agent "id" { … }`): command, models, prompt delivery, headless or terminal mode, detection, usage, error patterns. Built-ins ship embedded; entries merge by id.
- **Consequences:** A new agent needs no code. Agent errors normalize to classes, so retry rules work for every agent.
- **Source:** [spec §11.1](../spec/11-agents.md#111-agent-registry-millagent-in-lua), [§29.7](../spec/29-errors.md#297-agent-errors-are-normalized), principle 8.

## ADR-013: One package format for plugins and pipeline bundles

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Sharing a pipeline should bring its dependencies with it.
- **Decision:** A plugin is a directory or `.millpkg` with a `plugin.lua` manifest. A bundle is a plugin that mostly holds pipelines. Install, trust and lock work the same for both.
- **Consequences:** `mill export` and `mill import` reuse the plugin flow. The lockfile pins versions and hashes for both.
- **Source:** [spec §9.5](../spec/09-parts-and-plugins.md#95-plugin-package-format), [§9.9](../spec/09-parts-and-plugins.md#99-exporting-pipelines).

## ADR-014: Crash, don't guess

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** After a daemon restart, a step that was live may have had side effects.
- **Decision:** On startup `milld` replays unfinished runs. Queued nodes re-queue; `resumable` parts get `Resume`; `idempotent` parts re-run; anything else becomes `crashed` and the run fails with `M4301`, keeping finished outputs. Terminal agents always become `crashed`.
- **Consequences:** mill never silently repeats a push, a post or a payment. The user continues with `mill rerun <run> --from <node>`.
- **Source:** [spec §15.7](../spec/15-execution.md#157-daemon-restarts-and-crash-recovery), [§5.3](../spec/05-architecture.md#53-design-decisions).

## ADR-015: Steps are filters; edges carry item streams

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Unix tools should work as steps with zero integration.
- **Decision:** Edges carry ordered streams of items. `io: plain` and `io: text` nodes see only data or lines; mill wraps their output and carries files, workspace and lineage forward. mill itself reads items on stdin and writes outputs on stdout; logs go to stderr.
- **Consequences:** `jq`, `grep` and scripts are steps unchanged. Pipelines pipe into each other. Open question 1 (streams vs single values by default) may adjust ergonomics, not the model.
- **Source:** [spec §6.1](../spec/06-data-model.md#61-items), [§14](../spec/14-piping.md#14-piping-and-unix-composability), principle 6.

## ADR-016: Errors are interface

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Scripts, agents and people all need to act on errors.
- **Decision:** Every error has a stable `M` code (never reused or renumbered), a class, a plain headline, where it happened and the next command. The same object appears in CLI JSON, protocol `Error` events, MCP and the gateway. Exit codes derive from the class.
- **Consequences:** The error system ships with the first command (v0). Every code needs an explain page and golden tests; CI fails otherwise.
- **Source:** [spec §29](../spec/29-errors.md#29-errors-and-diagnostics), principle 13.

## ADR-017: Additive evolution

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Clients, parts and pipeline files from different versions must keep working together.
- **Decision:** Wire messages, the parts protocol and the file format are versioned. New optional fields and new message types do not bump the version; readers ignore unknown fields. Changing a meaning requires a version bump.
- **Consequences:** Never repurpose a field. CLI tables only add columns on the right.
- **Source:** [spec §2](../spec/02-principles.md#2-principles) principle 12, [§17.6](../spec/17-cli.md#176-output-modes).

## ADR-018: Safe by default

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** mill runs agents and commands unattended, sometimes from code it did not write.
- **Decision:** Code-changing nodes run in isolated workspaces. Destructive capabilities need `policy.allow_destructive`. Pipelines, plugins and project config from elsewhere need explicit trust, recorded by hash. Dangerous paths are refused. Listeners bind to localhost.
- **Consequences:** `mill lint` fails pipelines that use unlisted destructive parts. `--dry-run` skips destructive parts.
- **Source:** [spec §22](../spec/22-security.md#22-security-and-trust), principle 14.

## ADR-019: Secrets by reference only

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Secrets must never land in logs or process lists.
- **Decision:** Pipelines declare secret names; values come from backend parts and reach only the parts that name them, through env or the protocol, never argv. Every value is registered with the redactor.
- **Consequences:** In expressions a secret is an opaque handle. Exports carry secret names, never values.
- **Source:** [spec §21](../spec/21-secrets.md#21-secrets), [§5.4](../spec/05-architecture.md#54-invariants) invariant 9.

## ADR-020: Headless first

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** CI, cron, SSH and agents need the same power as the TUI.
- **Decision:** Every TUI action has a CLI command. Every listing takes `--json`. Destructive commands need `--yes` outside a terminal. `--dry-run` exists on every command that changes state.
- **Consequences:** The CLI ships before the TUI (v0.1 before v0.2).
- **Source:** [spec §17.5](../spec/17-cli.md#175-working-without-the-tui), principle 9.

## ADR-021: One board, simple surface

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** v1 of the spec had five tabs and seven-column tables.
- **Decision:** One board with FAILED, RUNNING and READY sections and one line per pipeline. Everything else opens over it; `esc` returns. Five keys in the help bar. Color means state only. The rules follow rex's design rules.
- **Consequences:** Depth lives in `milld` and reaches the screen as a state, a few words, or a square in the run history.
- **Source:** [spec §18.1](../spec/18-tui.md#181-design-rules), principles 2–3.

## ADR-022: View-only attach

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Watching a terminal agent is useful; typing into it would break hands-off.
- **Decision:** `mill attach` and `enter` on a node show the agent's screen without sending keystrokes. The protocol has no input intent.
- **Consequences:** A stuck terminal agent is fixed by changing the pipeline (prompt, flag or `auto_answer` rule), never by hand.
- **Source:** [spec §11.4](../spec/11-agents.md#114-agents-that-need-a-terminal), [§16.3](../spec/16-wire-protocol.md#163-intents).

## ADR-023: Local-first, hosted optional

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** mill must work offline on one machine.
- **Decision:** Everything works without the hosted layer. Relay, remote runners, run viewer, registry and shared history only add reach. Optional pieces (sound, notifications, summaries, telemetry) fail silently without affecting a run. Telemetry is never on by default.
- **Consequences:** No core feature may need a network service.
- **Source:** [spec §25](../spec/25-hosted.md#25-optional-hosted-layer), principles 10 and 15.
