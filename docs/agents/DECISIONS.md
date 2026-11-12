# Decisions

Choices already made in the spec, as a log an agent can check before undoing something. Each entry: date, status, context, decision, consequences, source.

ADR-001 to ADR-023 were recorded on 2026-11-11 from spec v2. ADR-024 to ADR-044 record the decisions the user (and, for ADR-044, the main session acting for the user) made the same day on the backend and editor plans' open decisions; they are applied in spec v2.1. ADR-045 to ADR-053 record the wizard and command-line decisions of 2026-11-12, applied in spec v2.2. ADR-055 records the full-screen pages of spec v2.3. Status `accepted` means the spec states it as decided. Questions still open live in [spec §30](../spec/30-open-questions.md#30-open-questions). To add a decision: append the next number, never renumber; to reverse one, add a new entry that supersedes it and mark the old one `superseded by ADR-NNN`.

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
- **Consequences:** The canvas needs its own cell-grid component inside Bubble Tea. Exact library versions are a backend plan matter (decisions D2–D4, D28; see [ADR-027](#adr-027-go-127) and [ADR-044](#adr-044-accepted-plan-defaults)).
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

- **Date:** 2026-11-11 · **Status:** accepted · amended by [ADR-041](#adr-041-built-in-part-ids-win)
- **Context:** A protocol that built-ins bypass goes incomplete.
- **Decision:** Built-in parts get no private internals. Any built-in can be replaced by a user part with the same id.
- **Consequences:** If a built-in needs a capability, the protocol grows it. The backend plan runs built-ins in-process behind the same interface, proven equal by a conformance suite (open decision D12).
- **Source:** [spec §5.3](../spec/05-architecture.md#53-design-decisions), [§10](../spec/10-builtin-parts.md#10-built-in-parts-catalog).

## ADR-008: Event-sourced run log; files are the truth

- **Date:** 2026-11-11 · **Status:** accepted · amended by [ADR-044](#adr-044-accepted-plan-defaults) (redaction before persistence, S1)
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

- **Date:** 2026-11-11 · **Status:** accepted · amended by [ADR-042](#adr-042-daemon-stop-is-a-crash)
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

- **Date:** 2026-11-11 · **Status:** accepted · amended by [ADR-029](#adr-029-fresh-clones-are-disarmed) and [ADR-036](#adr-036-dry-run-semantics)
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

## ADR-024: Name `mill`, own Homebrew tap

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Spec §30 Q12 asked for a name check. Core Homebrew already has a `mill` formula (a Scala build tool).
- **Decision:** The name stays `mill` (binaries `mill` and `milld`). It's distributed through its own tap: `brew install tristanbietsch/tap/mill`. Module path `github.com/TristanBietsch/mill`.
- **Consequences:** GoReleaser publishes to the `tristanbietsch/tap` tap. §30 Q12 is resolved.
- **Source:** User decision 1. [spec §27.1](../spec/27-tech-stack.md#271-language-go), [§30](../spec/30-open-questions.md#30-open-questions).

## ADR-025: Public repo on GitHub

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** The repo, default branch and remote were unconfirmed.
- **Decision:** mill lives at `github.com/TristanBietsch/mill`, public. All mill work is committed there.
- **Consequences:** Module path and repo name match. Links and docs use that URL.
- **Source:** User decision 2. [spec §27.1](../spec/27-tech-stack.md#271-language-go).

## ADR-026: Copy and diverge from rex

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** rex already has the codec, socket server, daemon autostart, PTY console, Lua dispatcher, CLI kit, test helpers and CI that mill needs (backend D1).
- **Decision:** Copy those files into mill and let them diverge. Each copied file starts with a header naming the rex source commit and path. A shared module is reconsidered after M5 (v0.1).
- **Consequences:** Fast start, no coupled release trains; fixes are ported by hand, tracked through the headers.
- **Source:** User decision 3, backend D1. [tech-stack §6](../plans/backend/tech-stack.md#6-what-mill-reuses-from-rex), [spec §27.1](../spec/27-tech-stack.md#271-language-go).

## ADR-027: Go 1.27

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Go version floor (backend D28).
- **Decision:** `go 1.27` with `toolchain go1.27.4`; `CGO_ENABLED=0`.
- **Consequences:** mill and rex use different toolchains (rex is on 1.26.x). The patch floor is raised for security fixes.
- **Source:** User decision 4, backend D28. [spec §27.1](../spec/27-tech-stack.md#271-language-go).

## ADR-028: Primary upstream edge rule

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Spec §7.4 made every reference a data edge, so `pr` in §7.5 would receive both `plan` and `review` items (backend S7, D9; editor decision 1).
- **Decision:** Every reference orders. A node's input is `in:`, else explicit `edges:`, else the primary upstream: the referenced node every other referenced node is upstream of. No primary upstream is lint error `M3021` (ambiguous input), with `in:` and `merge` fixes. "Unreachable" moves from `M3021` to `M3024`.
- **Consequences:** The editor draws inferred edges by the same rule the compiler uses.
- **Source:** User decision 5. [spec §7.4](../spec/07-pipeline-format.md#74-edge-inference), [§18.7.3](../spec/18-tui.md#1873-how-the-graph-is-drawn), [§18.7.9](../spec/18-tui.md#1879-lint).

## ADR-029: Fresh clones are disarmed

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** mill can't tell your repo from a cloned one; a clone with a cron or fswatch trigger would start running code as soon as `milld` saw it (backend S4, D18).
- **Decision:** A project's pipelines are untrusted until `mill trust .`: listed, triggers never armed, manual runs refused with `M9101`. The board shows `◌ not trusted · triggers off · mill trust .` in READY.
- **Consequences:** One `mill trust .` per project. User-dir pipelines stay trusted.
- **Source:** User decision 6. [spec §22](../spec/22-security.md#22-security-and-trust), [§13](../spec/13-triggers.md#13-triggers), [§4](../spec/04-concepts.md#pipeline-status-on-the-board).

## ADR-030: Windows is tier 2 in v0

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Windows support level for v0 (backend D7).
- **Decision:** Windows builds and runs unit, integration and `--local` smoke tests in CI from M1. PTY (terminal agents) and the container runner are best-effort until M11.
- **Consequences:** Windows doesn't rot, and ConPTY quirks don't gate every milestone.
- **Source:** User decision 7. [spec §27.1](../spec/27-tech-stack.md#271-language-go), [§28](../spec/28-roadmap.md#28-roadmap).

## ADR-031: Step environment by allowlist

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** `milld` may be started from a shell full of unrelated tokens (backend D30).
- **Decision:** Steps get `milld`'s environment filtered by `env.passthrough`, then pipeline `env`, then node `env`, then mill's own `MILL_*` variables. Node `env` is the only way a secret reaches an `sh`/`exec` process.
- **Consequences:** "Works like my shell" needs the variable added to `env.passthrough`.
- **Source:** User decision 8. [spec §12.1](../spec/12-runners.md#121-environment), [§19.6](../spec/19-config-lua.md#196-settings-registry).

## ADR-032: Loop YAML shape

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** The generic loop needed a syntax for its body (editor decision 4, backend S30).
- **Decision:** `uses: loop` with `with.body: [ids]`, `until` and required `max_iters`; the first body node reads `<loop>.body`; ports `out` (until held) and `exhausted` (ran out). The body is nodes in the same pipeline, not a sub-pipeline.
- **Consequences:** Overrides S30's sub-pipeline-body proposal; backend engine §11 updated. `agent.loop` covers the single-agent case before M8.
- **Source:** User decision 9. [spec §10.3](../spec/10-builtin-parts.md#103-control-flow).

## ADR-033: Agent pipelines without a budget

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Editor decision 7: should `M3201` (agents, no `budget`) always be an error?
- **Decision:** Error when the pipeline has any automatic trigger (anything but `manual`); warning when it only runs by hand.
- **Consequences:** Differs from the editor plan's default (always error). Manual pipelines like best-of-3 save without a budget, with a warning.
- **Source:** User decision 10. [spec §18.7.9](../spec/18-tui.md#1879-lint).

## ADR-034: Saving with lint errors

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Editor decision 8.
- **Decision:** `:w` refuses while there are errors (`M2203`); `:w!` saves, and a pipeline saved with errors keeps its triggers disarmed and can't run until lint passes. CLI edits need `--force`.
- **Consequences:** A broken pipeline never runs unattended. The board shows `lint errors · e to fix`.
- **Source:** User decision 11. [spec §18.7.9](../spec/18-tui.md#1879-lint).

## ADR-035: Pinned samples are committed files

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Editor decision 12: where pinned samples live.
- **Decision:** `.mill/samples/<pipeline>/<node>.jsonl`, committed with the pipeline. Pinning goes through the redactor, and lint warns (`M9208`) when a sample looks like it holds a secret.
- **Consequences:** Samples are repeatable fixtures that travel with the repo.
- **Source:** User decision 12. [spec §18.7.12](../spec/18-tui.md#18712-testing-while-you-build), [§18.7.9](../spec/18-tui.md#1879-lint).

## ADR-036: Dry run semantics

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** `sh`/`exec` can do anything but aren't flagged destructive (backend S22, D17).
- **Decision:** In `--dry-run`, `sh` and `exec` run for real unless the node sets `destructive: true`. Agent nodes are stubbed (a stub item matching `output`, nothing spent). `git.push`, `gh.*` and other destructive parts report what they would do.
- **Consequences:** Dry runs still build and test; nothing pushes, posts or spends tokens. Goes further than D17's default by also stubbing agents.
- **Source:** User decision 13. [spec §17.10](../spec/17-cli.md#1710-no-daemon---local), [§7.3](../spec/07-pipeline-format.md#73-node-keys), [§22](../spec/22-security.md#22-security-and-trust).

## ADR-037: Default retry policy

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Unattended runs hit rate limits (backend D23).
- **Decision:** Without a `retry`, a node gets 2 retries on `transient` and `rate_limited`, with exponential backoff, only for idempotent parts or failures marked retryable. Each retry is a new attempt; the failed one stays `failed`.
- **Consequences:** Rate limits don't fail nightly runs; side-effecting steps aren't re-run on a guess.
- **Source:** User decision 14. [spec §15.3](../spec/15-execution.md#153-retries), [§4](../spec/04-concepts.md#states).

## ADR-038: MIT license

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Backend D29 proposed Apache-2.0.
- **Decision:** mill is MIT licensed, `Copyright (c) 2026 Tristan Bietsch`. `LICENSE` at the repo root.
- **Consequences:** README and CONTRIBUTING state it; contributions are under the same terms.
- **Source:** User decision 15 (differs from the default). [LICENSE](../../LICENSE), [spec §25](../spec/25-hosted.md#25-optional-hosted-layer).

## ADR-039: One-page design and tech overviews

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** `docs/design.md` and `docs/tech.md` existed and were empty.
- **Decision:** `design.md` is a one-page product and UX overview; `tech.md` a one-page stack overview. Both link into the spec and plans instead of copying them.
- **Consequences:** Listed in `docs/index.md` and `docs/STRUCTURE.md`.
- **Source:** User decision 16. [../design.md](../design.md), [../tech.md](../tech.md).

## ADR-040: Dogfood order

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Which pipeline mill runs for real first (backend S28).
- **Decision:** `nightly-vuln-sweep` at the end of M0, from OS cron until mill has its own cron; then `morning-brief` (M1); then `issue-to-pr` (~M4, poll trigger).
- **Consequences:** M0's exit criteria and the milestone dogfood table changed.
- **Source:** User decision 17. [milestones](../plans/backend/milestones.md), [spec §28](../spec/28-roadmap.md#28-roadmap).

## ADR-041: Built-in part ids win

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Spec §9.4 let any later source override an id, so a stray `mill-part-sh` on `PATH` could replace `sh` (backend S5, D19).
- **Decision:** Built-ins win. `PATH` executables and MCP tools only fill ids nobody else provides. Overriding an existing id needs an explicit plugin or `mill.part` with `override = true`.
- **Consequences:** Amends ADR-007: a built-in is still replaceable, but only on purpose.
- **Source:** User decision 18. [spec §9.4](../spec/09-parts-and-plugins.md#94-resolution-order).

## ADR-042: Daemon stop is a crash

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Child processes can't outlive the daemon's pipes (backend S24, D26).
- **Decision:** `mill daemon stop` and `restart` with live runs use crash semantics: no drain, recovery rules apply on the next start. `--drain` waits for live node-runs up to 10 minutes (`shutdown.drain_timeout`).
- **Consequences:** One path for every way the daemon goes away. Amends ADR-014 by applying it to stops.
- **Source:** User decision 19. [spec §15.7](../spec/15-execution.md#157-daemon-restarts-and-crash-recovery).

## ADR-043: Registry and signing in v0.5

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** Spec §30 Q9 and Q10 (backend D27).
- **Decision:** v0.5's registry is a curated GitHub index repo plus `gh:` sources; the lockfile records SHA-256 hashes. Signatures (Sigstore, keyless) come with the hosted registry.
- **Consequences:** No service to run before v0.5; integrity from hashes until then.
- **Source:** User decision 20. [spec §9.6](../spec/09-parts-and-plugins.md#96-installing-plugins), [§9.11](../spec/09-parts-and-plugins.md#911-lockfile), [§22](../spec/22-security.md#22-security-and-trust).

## ADR-044: Accepted plan defaults

- **Date:** 2026-11-11 · **Status:** accepted
- **Context:** The backend plan's 30 decisions and 31 spec issues, the editor plan's 30 decisions, and the mockup↔spec mismatches needed answers. The user decided ADR-024 to ADR-043; the main session, acting for the user, accepted everything else.
- **Decision:** Accepted as the plans stated them:
  - **Backend decisions (defaults):** D2 Bubble Tea v2 · D3 `go.yaml.in/yaml/v4` + text-splice edits · D4 `modernc.org/sqlite` · D5 stdlib `flag` + rex output kit · D6 128-bit random hex run ids · D8 Git for Windows `sh.exe` · D10 `stream` mode deferred to M7 · D11 reset the workspace before a retry · D12 built-ins in-process with a conformance suite · D13 redact text artifacts, fail binary ones (`M9205`) · D14 global artifact store · D15 fsync on state changes, 100 ms group commit · D16 `max_wall` counts active time · D20 socket `$XDG_RUNTIME_DIR/mill/milld.sock`, else `~/.cache/mill/milld.sock` · D21 honor `XDG_*` on macOS, add `MILL_STATE_DIR` · D22 AF_UNIX on Windows too · D24 `fail_fast: true` · D25 extraction on the same agent's cheapest priced model.
  - **Spec issues S1–S31:** every recommendation, applied in spec v2.1 (the backend plan's table lists where). S30's sub-pipeline loop body is replaced by ADR-032.
  - **Editor decisions:** the defaults for 2, 3, 5, 6, 9, 10, 11 and 13–30.
  - **Package layout:** the backend plan's (architecture §2), now spec §27.4.
  - **Mockup↔spec:** crashed (`↯`) and budget-cancelled (`⊗`) in FAILED and counted as failed; user-cancelled (`⊗` dim) and untrusted (`◌`) in READY; branch rows indented under the node that split; retries as new attempts with the failed one as its own sub-row; `M3021` ambiguous input, `M3024` unreachable; mode name at the left of the board's help bar; conditional columns between text and runs; socket fallback `~/.cache/mill/milld.sock`.
- **Consequences:** Spec v2.1 states all of these. The plans mark each item decided. Code clash fixed on the way: the backend plan's argv-secret check moves from `M9203` (taken by "secret declared but never used") to `M9207`.
- **Source:** [open-decisions.md](../plans/backend/open-decisions.md), [00-overview.md](../plans/backend/00-overview.md#spec-issues), [editor.md](../plans/editor.md) §5, [spec index](../spec/index.md).

## ADR-045: Four-step wizard, name last

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** The v2.1 wizard had five steps, asked for the name first and used keys unlike the editor's ([wizard plan](../plans/wizard.md) §1).
- **Decision:** Four steps, `1 start › 2 shape › 3 run › 4 review`, with the editor's modal keys (`hjkl`, `i`, `esc`). The name is chosen last, on review, suggested from the source. Every field has a default, so `enter` goes all the way through.
- **Consequences:** Nothing is written before review. `mill new` flags follow the same four steps.
- **Source:** User decisions (wizard 1, 2). [spec §18.8](../spec/18-tui.md#188-wizards).

## ADR-046: Dry run, then edit after create

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** What the wizard does once the pipeline exists (wizard decision 6).
- **Decision:** Create runs a dry run (once lint is clean; destructive steps never run in a dry run) and opens the editor with the states painted on the graph. "Open the editor" stays as the alternative on review.
- **Consequences:** A new pipeline is seen working before it's edited.
- **Source:** User decision. [spec §18.8](../spec/18-tui.md#188-wizards), [§18.7.12](../spec/18-tui.md#18712-testing-while-you-build).

## ADR-047: Local-LLM drafter

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** *Describe it* turns a sentence into a proposed pipeline. The plan's default was a headless agent run (`wizard.agent`).
- **Decision:** A local model through Ollama's HTTP API drafts it, the way rex's summarizer works. Settings in `:settings` (§19.6): `drafter.backend` (`ollama` default · `agent` · `off`), `drafter.base_url`, `drafter.model` (resolved against pulled models), `drafter.agent`, `drafter.timeout`. When the backend is unavailable the keyword drafter is used with `M2212`; it never blocks. Drafts are validated against the schema and lint before they're shown; the sentence is untrusted input and is delimited.
- **Consequences:** Describing a pipeline is free and offline by default. `wizard.agent` is gone. Boot and `mill doctor` report drafter availability as optional (principle 10).
- **Source:** User decision (wizard 3). [spec §19.6](../spec/19-config-lua.md#196-settings-registry), [§18.8](../spec/18-tui.md#188-wizards), [modules/drafter.md](modules/drafter.md).

## ADR-048: n8n steps that wait for a person become handoffs

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** n8n workflows can contain approval or form-wait steps; mill is hands-off (principle 1).
- **Decision:** On import they become a `handoff` node that ends the branch, with warning `M2307` in the preview.
- **Consequences:** Imports still succeed and the result never waits on a person.
- **Source:** User decision (wizard 8). [spec §18.8](../spec/18-tui.md#188-wizards), [§23.3](../spec/23-integrations.md#233-n8n-and-zapier).

## ADR-049: Duplicates start manual

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** A copied pipeline with the same triggers fires twice on one event.
- **Decision:** A duplicate's triggers start as `manual`; step 3 offers `as in <original>`.
- **Consequences:** No accidental double runs.
- **Source:** User decision (wizard 9). [spec §18.8](../spec/18-tui.md#188-wizards).

## ADR-050: Accepted wizard defaults

- **Date:** 2026-11-12 · **Status:** accepted
- **Decision:** The wizard plan's defaults for decisions 4 (budget `medium`), 5 (GitHub issue on failure when `mill/github` is installed), 7 (`n8n.call` stubs, `M2304`), 10 (milld reads imports), 11 (5 MB cap), 12 (keep `mill import` and `mill new --from`), 13 (path, picker and pasted paths; terminal file transfer LATER).
- **Source:** User accepted the defaults. [wizard plan §12](../plans/wizard.md#12-decisions-2026-11-12).

## ADR-051: `i` on the board is a hint

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** The `λ` prompt that `i` opened is gone (ADR-053).
- **Decision:** The board has no text field, so `i` only echoes the `:run`/`:pipe` hint. `i` means insert everywhere else.
- **Source:** User decision (command line 2). [spec §18.2](../spec/18-tui.md#182-the-board).

## ADR-052: A pending confirm swallows `:`

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** `:` opens the command line from every screen (bottom-left, nvim-style). The plan's default let it open over a confirm.
- **Decision:** A pending y/n confirm takes only `y`, `n` and `esc`, as nvim's `confirm()`; `:` and a click on the last row do nothing until it's answered. In INSERT and text fields `:` types a colon.
- **Source:** User decision (command line 7). [spec §18.4](../spec/18-tui.md#184-focus-modes), [§18.5](../spec/18-tui.md#185-keys-and-mouse).

## ADR-053: Global `:` command line replaces the `λ` prompt

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** User requirement: commands are vim-like, typed after `:` on the bottom row, never after `/`; `:` runs from anywhere and sits in the bottom-left corner.
- **Decision:** One command line on the terminal's last row at column 0, from every screen (ADR-052 exception). `/` is search only. The `λ` prompt is replaced by `:run <pipeline> k=v` and `:pipe <expr>`. The plan's other defaults are accepted: one-row wildmenu, this screen's commands first, palette `:` types a colon, 200-entry per-user history, `gi` for items with `i` as an alias for one release, echo until the next key (12 s at most).
- **Source:** User requirement and accepted defaults. [command-line plan §7](../plans/command-line.md#7-decisions-2026-11-12), [spec §18.5](../spec/18-tui.md#185-keys-and-mouse).

## ADR-054: Layered backend refinement defaults (D31–D41)

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** The layered backend plan ([layers.md](../plans/backend/layers.md), citations in [references.md](../plans/backend/references.md)) needed eleven new decisions.
- **Decision:** The plan's defaults, accepted by the user: D31 event catalog and reducer in L1 (`core/event`, `core/fold`) · D32 `assemble` (L6) wires `milld` and `--local` · D33 directory decides layer (`lock`, `proc`, `redact` become L0 leaves; runner interface in `partproto`) · D34 `internal/archtest` + depguard + forbidigo enforce layering · D35 in-process metrics registry, OpenTelemetry/Prometheus only as optional sinks · D36 circuit breakers per agent (hold work queued, no retries spent) and per HTTP host (fail fast) · D37 no `mill backup` in v0 · D38 readers keep every shipped `log_v` decoder; newer → M7007 · D39 per-layer coverage floors · D40 CLI additions for stats/doctor/daemon logs/prune/daemon install · D41 stop granting below 1 GiB free (M2002), cap queued runs (M4108).
- **Consequences:** Spec §27.4 paths follow the layered layout. The M0 plan already uses it.
- **Source:** User decision. [open-decisions.md](../plans/backend/open-decisions.md).

## ADR-055: Fullscreen settings, help and new-pipeline screens, following rex v2

- **Date:** 2026-11-12 · **Status:** accepted
- **Context:** Settings, help and the new-pipeline wizard were square overlays (`clamp(w-8, 60, 110)`) over the dimmed board. Setting help and help rows were cut, help dropped whole groups on small terminals, the wizard lost its preview below 90 columns, and the board behind them was noise. rex v2 hit the same limits and turned its settings, help and new-agent screens into full-screen pages (R04, R16; shipped in rex v2.0.0). mill shares rex's conventions.
- **Decision:** The three screens become pages (§18.1.1): `≋ mill › <page>` in the board header's place, two panes (a list of at most 24 columns, then the content), an inspector on settings, mill's status line and five-key help bar, `tab` for panes, `1`–`9` and `[`/`]` to jump, a one-line strip under 60 columns, every state in a word or glyph, errors that never close the page. Rule 1 and rule 10 are rewritten with their reason (§18.1). Settings (§18.10) gains section badges, an inspector (`K` for the full help) and search across sections; help (§18.5.1) shows eight groups at full height with a filter across groups and a live legend, and `?` opens the current screen's group; new pipeline (§18.8) gets a step rail with marks and words and a graph preview at 110+ columns. mill keeps its own keys where rex's differ: `u` resets (rex `r`), `?` opens help (rex `?` expands a setting's help, here `K`), five keys in the help bar (rex shows more), and clicks on pages (rex takes the wheel only).
- **Consequences:** Decided semantics are unchanged: the four steps, name last, dry run then edit, the drafter, import, n8n handoffs, duplicates starting manual and the modal keys (ADR-045 to ADR-050), and the settings registry with its Drafter section. rex choices that would change them, plus accent color, an overlay flag, plugins as a page and extra help groups, are open questions 13–21 (§30), not decided here. §17.5 names the pages and adds help's CLI (`mill <command> --help`, `mill explain`). The browser mockup still shows overlays until it is updated.
- **Source:** User request (2026-11-12). rex: `docs/plans/r2/R04-00-plan.md`, `docs/plans/r2/R04-fullscreen-settings-help.md`, `docs/plans/ANTI-fullscreen-settings-help-spec.md`, `docs/plans/r2/R16-00-plan.md`, `docs/plans/r2/R16-new-agent-flow.md`, `docs/releases/v2.0.0.md`, `docs/tui.md`, `docs/settings.md`, `internal/surface/tui/page.go`. [spec §18.1](../spec/18-tui.md#181-design-rules), [§18.10](../spec/18-tui.md#1810-settings), [§18.8](../spec/18-tui.md#188-wizards), [open questions](../spec/30-open-questions.md#30-open-questions).
