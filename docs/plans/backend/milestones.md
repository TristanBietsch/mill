# Milestones

Re-sequences spec §28 into milestones that each end in something runnable, starting with the weekend prototype. Each milestone builds the real packages (no throwaway prototype code): M0 writes small versions of `pipeline`, `expr`, `engine`, `store` that later milestones grow.

Sizes are focused working days for one developer working with coding agents; "S" ≤ 3 d, "M" 4–8 d, "L" 9–15 d.

The three dogfood targets from spec §26, in the order decided on 2026-11-11 (spec §28): `nightly-vuln-sweep` first, then `morning-brief`, then `issue-to-pr`. Until mill has its own cron (M4), the OS crontab calls `mill run --local`.

| Target | First real run | How it's triggered then | Full form |
|---|---|---|---|
| `nightly-vuln-sweep` | **end of M0** (lite: `sh` runs `govulncheck -json ./...` and `npm audit --json` → headless agent triages the findings → `sh` writes a markdown report) | OS cron calling `mill run` (M0 has no daemon, so no `--local` flag is needed) | M3: `if` found → `agent.loop` fix in a worktree until `go test ./...` → draft `gh.pr`; M4: mill cron with `catch_up: last` |
| `morning-brief` | end of M1 (lite: `sh` + `curl` + headless agent → markdown file) | OS cron calling `mill run --local` | M4: mill cron, RSS via `http`+`parse`, calendar via MCP, Slack via MCP |
| `issue-to-pr` | ~M4 | `poll` trigger on `gh issue list --label agent` (manual `--input issue=@issue.json` works from M3) | M7: GitHub webhook via gateway |

## Overview

| # | Name | Spec phase | Size | Ends with |
|---|---|---|---|---|
| M0 | Weekend prototype | §28 prototype | S (2–3 d) | `mill run file.mill.yaml` runs sh + headless agent in topological order, JSONL between steps; `nightly-vuln-sweep-lite` runs from OS cron |
| M1 | Engine core, local mode | v0 (part) | L | `mill run --local` with the real engine: batch/each, retries, budgets, cancel, event log, items, errors, config |
| M2 | Daemon | v0 (rest) | L | `milld` + socket + autostart + recovery + index; CLI `run/runs/logs/wait/items/rerun/cancel/status/watch` |
| M3 | Code pipelines | v0 parts + v0.1 bits | L | `nightly-vuln-sweep` full form runs nightly via OS cron; `issue-to-pr` passes on a sandbox repo |
| M4 | Triggers and MCP client | v0.4 (part) + v0.5 (MCP client) | M–L | `morning-brief` and `nightly-vuln-sweep` run on mill's own cron for a week; `issue-to-pr` dogfoods on `poll` |
| M5 | CLI completeness and pipe | v0.1 | M | every §17 command except plugin/import/export; pipe expressions; YAML editing; `render`/`show` |
| M6 | Backend for the TUI | v0.2–0.3 support | M | PTY runner + attach repaint, stats, board snapshot fields, EditPipeline, lint-on-save |
| M7 | Gateway, webhooks, fswatch, git hooks, stream mode | v0.4 (rest) | M | GitHub webhook starts `issue-to-pr` |
| M8 | Generic loop, fanout/judge, cache polish, retention | v0 leftovers | M | `best-of-3` runs |
| M9 | Plugins | v0.5 | L | external Linear plugin installs pinned; export/import between machines; container + ssh runners |
| M10 | Interop and remote | v0.6 + v1 | L | n8n import; remote runner |
| M11 | Hardening | ongoing | — | Windows parity for PTY, cgroups option, signatures |

TUI work (spec v0.2–v0.3) runs in parallel from M5 on, against the M2 protocol; it's outside this plan except for the backend it needs (M6).

---

## M0 — Weekend prototype

**Scope** (spec §28 last paragraph, nothing more):
- `mill run <file.mill.yaml> [--input k=v]... [--stdin]`: parse YAML (subset: `mill`, `name`, `inputs`, `nodes` with `uses`, `with`, `in`, `outputs`), infer edges from `${{ nodes.x }}` and `in`, topological sort, run nodes **sequentially**.
- Parts: `sh` (`io: plain|text`), `agent` headless only (registry entries for `claude`, `codex`, `echo` hard-coded in Go), `stdout`.
- CEL `${{ }}` for `inputs`, `item`, `items`, `nodes.x.item(s)`.
- Items as JSONL between steps (each step's stdout → items file → next step's stdin).
- Run dir with `events.jsonl` (only `RunCreated`, `NodeRunStarted`, `NodeRunFinished`, `RunFinished`) and `items/`, `logs/`.
- Outputs to stdout as JSONL; progress to stderr; exit 0/1.

**Packages:** `cmd/mill`, `pipeline` (parse + infer), `expr` (scanner + CEL env), `engine` (sequential runner, no scheduler yet), `store` (run dir writer), `parts/builtin/sh`, `parts/builtin/agent`, `core`.

**Not in M0:** daemon, Lua, retries, budgets, each mode, errors system beyond plain messages, Windows.

**Exit criteria:**
1. `examples/hello.mill.yaml` (two `sh` nodes) runs and `mill run hello.mill.yaml --json | jq .greeting` works.
2. `nightly-vuln-sweep-lite`: `sh` (`govulncheck -json ./...`, `npm audit --json` on a sandbox repo) → `agent claude/haiku` "triage these findings: what's real, what to fix first" → `sh` writes `~/notes/vuln-<date>.md`. Runs nightly from the user's crontab. This is the first dogfood pipeline.
3. A three-node pipeline with `item` per line through `jq` (`io: plain`) shows the pipe ergonomics.
4. Notes file `docs/plans/backend/m0-findings.md` (format, item model, ergonomics, what the vuln sweep needed that M0 lacked), checked against decisions D9–D10.

## M1 — Engine core, local mode

**Scope:**
- Errors system first (spec §28 v0): `errs` package, M-codes, classes, human + JSON rendering via the CLI kit copied from rex, `mill explain`, `errors-check`.
- Full compile pipeline ([engine.md](engine.md) §1): positions, lint (unknown nodes/parts, cycles, undeclared secrets, policy), M3021 ambiguous input.
- Run actor + reducer + scheduler loop with slot pools; `batch` and `each` modes with `concurrency`; `when`; skipped propagation; `on_error`; `fail_fast`.
- Retries with backoff and the default policy; timeouts; cancel/teardown path (Ctrl-C on `--local` = cancel); budgets (tokens, cost, wall, nodes, agents).
- Store: event log with group commit and Sync events, item commit protocol, logs with redaction hook (redactor implemented, secrets broker with `env` backend only), CAS, `run.json`, `run.lock`.
- Runner `local` with process groups on macOS/Linux and Job Objects on Windows.
- Agents: Lua-free registry in Go (`builtin` entries), headless output parsing, usage, error normalization, structured output (MILL_OUTPUT + schema + retry-once; extraction later).
- Config: Lua `init.lua` with `mill.setup` and `mill.agent`, settings registry, layered precedence, `config get --why`.
- Parts: `sh`, `exec`, `check`, `map`, `filter`, `take`, `split`, `jq`, `if`, `assert`, `fail`, `noop`, `agent`, `stdin`, `stdout`.
- CLI: `mill run --local` (+ `--stdin`, `--each`, `--json`, `--dry-run` skeleton), `mill lint`, `mill explain`, `mill runs --store`.
- `clock` fake, `testenv`, echo agent, testscript harness, CI gate on three OSes.

**Packages:** `core errs clock paths mlog expr schema pipeline pipeline/ir engine engine/sched engine/budget engine/retry store store/cas store/lock runner runner/proc runner/local agents secrets secrets/redact config config/settings luaapi partproto parthost parts/builtin/* surface/cli surface/cli/core testutil`.

**Exit criteria:**
1. Example suite (`hello`, `each-fanout`, `retry-rate-limit`, `budget-stop`, `cancel`) passes on Linux, macOS, Windows (`--local`; Windows is tier 2, D7).
5. `nightly-vuln-sweep-lite` moves to `mill run --local` in the crontab; `morning-brief-lite` (`sh` curl two RSS feeds → `agent claude/haiku` "summarize" → `sh` append to `~/notes/brief.md`) joins it.
2. `mill run x --local --budget-tokens 1k` exits 9 with M4101; Ctrl-C leaves no child processes (checked by the test with `@echo:spawn-child`).
3. Invariants 4–9 tests green; engine coverage ≥ 85 %.
4. Benchmarks B1–B7, B11 recorded as baselines.

## M2 — Daemon

**Scope:**
- `milld`: boot wiring, `daemon.lock`, socket server (rex copy: conns, peercred, bounded queues), handshake, intents `Hello StartRun RunInput RunInputEnd CancelRun DeleteRun RerunFrom RetryNode Subscribe Unsubscribe ListRuns GetRun GetItems GetLogs ListPipelines GetPipeline Lint Reload Shutdown`.
- `daemonctl` autostart from `mill` (rex copy), `mill daemon start|stop|status|restart|logs`.
- Pipeline catalog with fsnotify reload (invariant 10).
- Crash recovery: live markers, replay, orphan kill by identity, §15.7 table, `RunRecovered`.
- Index (SQLite): schema, write-behind indexer, rebuild, selectors (`#n`, `#last`, prefix).
- Status file + `mill prompt`, `mill status`.
- CLI: `run` (wait/detach via daemon), `runs`, `logs [-f]`, `items`, `wait`, `cancel`, `rm`, `rerun [--from]`, `watch`, `events`.
- Lua hooks in milld (`mill.on`) through the rex dispatcher pattern.

**Exit criteria:**
1. Kill -9 milld mid-run in a test: idempotent node re-runs, non-idempotent node `crashed` M4301, `mill rerun --from` finishes it.
2. Two clients see the same events; a stalled client is dropped without slowing the other (rex test pattern).
3. `mill run p --json | jq` works identically with and without `--local`.
4. B9, B10, B12 within budget.

## M3 — Code pipelines

**Scope:**
- Workspace manager: git worktrees, temp dirs, cleanup policies, retry snapshot/reset, diff artifacts, path safety with `os.Root`.
- Parts: `git.worktree`, `git.diff`, `git.commit`, `git.push`, `gh.pr`, `gh.issue`, `gh.comment` (via `gh` CLI), `handoff`, `agent.loop`, `agent.review`, `agent.extract`, `file.read|write|append|glob`, `template`, `merge`.
- Secrets: broker with `keychain` backend, `mill secrets set|ls|rm|test`, redaction everywhere, M9207 argv checks.
- Policy: `allow_destructive` lint, `--dry-run` for destructive parts.
- Project trust: `mill trust`, untrusted projects data-only.

**Exit criteria:**
1. `nightly-vuln-sweep` full form (`govulncheck` → `if` → `agent.loop` until `go test ./...` → `gh.pr`) runs nightly from the user's crontab for a week with no manual intervention.
2. `issue-to-pr` (spec §7.5 minus webhook and Slack) opens a draft PR on a sandbox repo from `--input issue=@issue.json` in an integration test; out-of-scope plans produce an issue comment. Its first real dogfood run waits for M4.
3. `TestSecrets_NotOnDisk` and crash-point exploration green for both examples.

## M4 — Triggers and MCP client

**Scope:**
- Trigger host: arm/disarm on pipeline reload, persistent state (`triggers/*.json`), concurrency policies (`queue`, `skip`, `replace`, `parallel N`), dedupe windows.
- Triggers: `cron` (tz, jitter, `catch_up`), `schedule.once`, `poll` (any node part, dedupe key), `run` (another run's done/failed), `manual`.
- MCP client: `mill.mcp` config, server lifecycle in parthost, `mcp:<server>/<tool>` nodes with schemas from `tools/list`, `mill mcp ls`.
- Parts: `http` (pagination helpers), `parse` (json/yaml/csv/xml/html/front-matter), `notify` (desktop: `osascript`/`notify-send`/toast), `wait`, `sort`, `uniq`, `skip`, `batch`, `reduce`.
- `mill trigger ls|enable|disable|fire`.

**Exit criteria:**
1. `morning-brief` full form (cron 07:30 weekdays, RSS + calendar MCP + Slack MCP) runs a week; missed runs during sleep follow `catch_up: last`.
2. `nightly-vuln-sweep` moved from crontab to mill cron.
3. `issue-to-pr` starts dogfooding on a `poll` trigger (`gh issue list --label agent --json number,title,body` every 10 min, dedupe by number).

## M5 — CLI completeness and pipe

**Scope:** pipe expressions (`mill pipe`, `--save`, `--dry-run`), `yamledit` + `mill node|edge` commands + `EditPipeline` intent, `mill new` (flags + terminal prompts, `describe:` via headless agent), `mill show --outline|--yaml|--graph`, `mill render` (text, mermaid), `mill ls`, completion (bash/zsh/fish), `mill doctor` (+ `--bundle`), `mill setup` (CLI flow), `mill stats`, `mill digest`, `mill cache`, `mill prune`, `mill config list|get|set|reset|edit|check`, `mill lua-types`, `mill agent ls|test`, `mill part ls|info`, `mill mcp serve`.

**Exit criteria:** spec §26 pipe one-liners work; §17.11 one-liners work; every command has golden tests in all output modes.

## M6 — Backend for the TUI

**Scope:** PTY runner (rex console copy, supervisor, vt10x mirror, auto-answer, M6304 detection, transcripts), `Subscribe{repaint}` + `Resize`, `NodeOutput` coalescing, `Snapshot` board fields (`text` column, next fire, long-run p50 durations), `Stats`/`ListTriggers`/`ListParts`/`PartCall` intents, `ConfigStatus` events, lint-on-save.

**Exit criteria:** `mill attach run/node` shows a terminal-only agent's screen at once (repaint) on macOS and Linux; an agent that asks a question fails M6304 quoting it within `idle_ms + 1 s`.

## M7 — Gateway, webhooks, fswatch, git hooks, stream mode

**Scope:** `gateway` (HTTP + WS, tokens, limits), `webhook` trigger with all verifiers and `ack|wait` response modes, `fswatch`, `git` hooks (`mill hooks install`), `stream` mode with windows and credits, `n8n.call`/`zapier.call` with callbacks.

**Exit criteria:** a GitHub webhook (via a tunnel) starts `issue-to-pr`; `notes-to-post` (fswatch) runs.

## M8 — Loops, fanout, cache, retention

**Scope:** generic `loop` (sub-pipeline body), `pipeline` part, `agent.fanout`, `agent.judge`, cache keys + `mill cache`, retention/GC job, zstd log archiving.

**Exit criteria:** `best-of-3` and `multi-repo-refactor` run; GC keeps disk bounded over a month of dogfooding.

## M9 — Plugins (v0.5)

**Scope:** public parts protocol v1 + conformance suite as a published test kit, Go SDK, Lua parts with permission enforcement, plugin manifest/resolver/installer/lockfile, `mill plugin *`, `mill import|export` bundles with secret scanning, OpenAPI plugin generator, container and ssh runners, `plugins.sandbox = "container"`, registry index on GitHub (D27).

**Exit criteria:** spec v0.5 "done when".

## M10 — Interop and remote (v0.6, v1)

n8n import/export, mermaid/gha exports, `external` runner adapters (`mill/rex` plugin), remote runners, relay client, run viewer (HTML + SVG).

## M11 — Hardening (continuous after M4)

Windows PTY parity, `CREATE_SUSPENDED` job assignment, Linux cgroup scopes, plugin signatures, WASM parts (wazero), long-duration soak tests.
