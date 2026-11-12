# Milestones

Re-sequences spec §28 into milestones that each end in something runnable, starting with the weekend prototype. Each milestone builds the real packages (no throwaway prototype code): M0 writes small versions of `pipeline`, `expr`, `engine`, `store` that later milestones grow. Since the 2026-11-12 refinement each milestone also says **which layers it builds or extends** ([layers.md](layers.md)) and **which fitness functions it adds**; a milestone isn't done until its fitness functions are green in `make check` [BEA ch. 2] [BEA ch. 4].

Sizes are focused working days for one developer working with coding agents; "S" ≤ 3 d, "M" 4–8 d, "L" 9–15 d.

The three dogfood targets from spec §26, in the order decided on 2026-11-11 (spec §28, ADR-040): `nightly-vuln-sweep` first, then `morning-brief`, then `issue-to-pr`. Until mill has its own cron (M4), the OS crontab calls `mill run --local`.

| Target | First real run | How it's triggered then | Full form |
|---|---|---|---|
| `nightly-vuln-sweep` | **end of M0** (lite: `sh` runs `govulncheck -json ./...` and `npm audit --json` → headless agent triages the findings → `sh` writes a markdown report) | OS cron calling `mill run` (M0 has no daemon, so no `--local` flag is needed) | M3: `if` found → `agent.loop` fix in a worktree until `go test ./...` → draft `gh.pr`; M4: mill cron with `catch_up: last` |
| `morning-brief` | end of M1 (lite: `sh` + `curl` + headless agent → markdown file) | OS cron calling `mill run --local` | M4: mill cron, RSS via `http`+`parse`, calendar via MCP, Slack via MCP |
| `issue-to-pr` | ~M4 | `poll` trigger on `gh issue list --label agent` (manual `--input issue=@issue.json` works from M3) | M7: GitHub webhook via gateway |

## Overview

| # | Name | Spec phase | Size | Layers built (B) / extended (E) | Ends with |
|---|---|---|---|---|---|
| M0 | Weekend prototype | §28 prototype | S (2–3 d) | B: L0, L1, L2, L3, L4, L5, L7 (thin slice of each) | `mill run file.mill.yaml` runs sh + headless agent in topological order, JSONL between steps; `nightly-vuln-sweep-lite` runs from OS cron |
| M1 | Engine core, local mode | v0 (part) | L | E: L0–L5, L7 (L0, L1 complete) | `mill run --local` with the real engine: batch/each, retries, budgets, cancel, event log, items, errors, config |
| M2 | Daemon | v0 (rest) | L | B: L6; E: L1 (`wire/protocol`), L3 (`store/index`), L5 (`wire/client`, `daemonctl`), L7 | `milld` + socket + autostart + recovery + index; CLI `run/runs/logs/wait/items/rerun/cancel/status/watch` |
| M3 | Code pipelines | v0 parts + v0.1 bits | L | E: L4 (`workspace`, `secrets`), L5 (git, gh, agent parts, trust) | `nightly-vuln-sweep` full form runs nightly via OS cron; `issue-to-pr` passes on a sandbox repo |
| M4 | Triggers and MCP client | v0.4 (part) + v0.5 (MCP client) | M–L | E: L4 (breakers in `engine/sched`), L5 (`trigger/**`, MCP in `parthost`, `http`) | `morning-brief` and `nightly-vuln-sweep` run on mill's own cron for a week; `issue-to-pr` dogfoods on `poll` |
| M5 | CLI completeness and pipe | v0.1 | M | E: L2 (`pipe`, `edit`, `yamledit`), L4 (`engine/cache`), L7 (every §17 command) | every §17 command except plugin/import/export; pipe expressions; YAML editing; `render`/`show`; `doctor`, `stats`, `digest` |
| M6 | Backend for the TUI | v0.2–0.3 support | M | E: L4 (`runner/pty`), L5 (`agents/detect`), L6 (server intents) | PTY runner + attach repaint, stats, board snapshot fields, EditPipeline, lint-on-save |
| M7 | Gateway, webhooks, fswatch, git hooks, stream mode | v0.4 (rest) | M | B: `gateway` (L6); E: L4 (stream mode), L5 (webhook, fswatch, githook triggers; `otel`/`prometheus` sinks) | GitHub webhook starts `issue-to-pr` |
| M8 | Loops, fanout, cache, retention | v0 leftovers | M | E: L3 (GC, zstd), L4 (`loop`, `pipeline`, fanout) | `best-of-3` runs; disk stays bounded |
| M9 | Plugins | v0.5 | L | E: L4 (`runner/container`, `runner/ssh`), L5 (`plugins`, `parthost/luapart`); B: `sdk/go` | external Linear plugin installs pinned; export/import between machines; container + ssh runners |
| M10 | Interop and remote | v0.6 + v1 | L | E: L5 (`runner/remote`, `runner/external`, importers) | n8n import; remote runner |
| M11 | Hardening | ongoing | — | E: L0 (`proc` on Windows), L4 (cgroups), L6 (`mill daemon install`) | Windows parity for PTY, cgroups option, signatures, soak |

TUI work (spec v0.2–v0.3) runs in parallel from M5 on, against the M2 protocol; it's outside this plan except for the backend it needs (M6).

**Every milestone's gate** = `make check` (vet, race tests, lint with depguard and forbidigo, coverage floors, govulncheck, errors-check, `internal/archtest`) plus the fitness functions listed under it. Adding an integration point (a new runner, part kind, trigger or external service) also means reviewing it against the antipattern checklist in [resilience.md](resilience.md) §9 [RI ch. 4].

---

## M0 — Weekend prototype

**Scope** (spec §28 last paragraph, nothing more):
- `mill run <file.mill.yaml> [--input k=v]... [--stdin]`: parse YAML (subset: `mill`, `name`, `inputs`, `nodes` with `uses`, `with`, `in`, `outputs`), infer edges from `${{ nodes.x }}` and `in`, topological sort, run nodes **sequentially**.
- Parts: `sh` (`io: plain|text`), `agent` headless only (registry entries for `claude` and `echo` hard-coded in Go; `codex` moves to M1), `stdout`.
- CEL `${{ }}` for `inputs`, `item`, `items`, `nodes.x.item(s)`.
- Items as JSONL between steps (each step's stdout → items file → next step's stdin).
- Run dir with `events.jsonl` (only `RunCreated`, `NodeRunStarted`, `NodeRunFinished`, `RunFinished`) and `items/`, `logs/`.
- Outputs to stdout as JSONL; progress to stderr; exit 0/1/5.

**Packages by layer** (final names from day one, so M1 extends instead of renaming; detail in [M0-weekend-prototype.md](M0-weekend-prototype.md)):

| Layer | M0 packages |
|---|---|
| L0 | `errs`, `clock`, `paths`, `proc` (process groups, env allowlist) |
| L1 | `core`, `partproto` (M0 shape: `Part.Run(ctx, Call) (Result, error)`) |
| L2 | `expr`, `pipeline` |
| L3 | `store` (run dir writer) |
| L4 | `engine` (sequential, no scheduler) |
| L5 | `parthost` (registry), `parts/builtin/{sh,agent,stdout}`, `agents` |
| L7 | `cmd/mill`, `surface/cli`, `surface/cli/local` (composition root) |

**Not in M0:** daemon, Lua, retries, budgets, each mode, errors system beyond the §29 text shape, Windows.

**Fitness functions added:** `internal/archtest` with A1 (every package mapped), A2 (downward only), A3 (closed surface, with the `surface/cli/local` exceptions marked `until M2`), A6 (built-ins have no internals), A8 (pure core), A10 (no test helpers in production); depguard rules L0, L1, L2 (strict) and L7; forbidigo for `time.*`, `fmt.Print*`, `exec.Command`.

**Exit criteria:**
1. `examples/hello.mill.yaml` (two `sh` nodes) runs: `mill run hello.mill.yaml | jq .greeting` and `mill run hello.mill.yaml --json | jq .outputs.greeting` both print the greeting.
2. `nightly-vuln-sweep-lite`: `sh` (`govulncheck -json ./...`, `npm audit --json` on a sandbox repo) → `agent claude/haiku` "triage these findings: what's real, what to fix first" → `sh` writes `~/notes/vuln-<date>.md`. Runs nightly from the user's crontab. This is the first dogfood pipeline.
3. A three-node pipeline with `item` per line through `jq` (`io: plain`) shows the pipe ergonomics.
4. Notes file `docs/plans/backend/m0-findings.md` (format, item model, ergonomics, what the vuln sweep needed that M0 lacked), checked against decisions D9–D10.
5. `make check` green including the M0 fitness functions above.

## M1 — Engine core, local mode

**Scope:**
- Errors system first (spec §28 v0): `errs` package, M-codes, classes, human + JSON rendering via the CLI kit copied from rex, `mill explain`, `errors-check` ([errors.md](errors.md)).
- Full compile pipeline ([engine.md](engine.md) §1): positions, lint (unknown nodes/parts, cycles, undeclared secrets, policy), M3021 ambiguous input; `pipeline.Resolver` port.
- Run actor + `core/fold` reducer + `core/event` catalog + scheduler loop with slot pools; `batch` and `each` modes with `concurrency`; `when`; skipped propagation; `on_error`; `fail_fast`.
- Retries with backoff and the default policy; timeouts; cancel/teardown path (Ctrl-C on `--local` = cancel); budgets (tokens, cost, wall, nodes, agents).
- Store: event log with group commit and Sync events, item commit protocol, logs with redaction (`redact` implemented, secrets broker with `env` backend only), CAS, `run.json`, `run.lock` (`lock`).
- Runner `local` with process groups on macOS/Linux and Job Objects on Windows (`proc`).
- Agents: Lua-free registry in Go (`builtin` entries), headless output parsing, usage, error normalization, structured output (MILL_OUTPUT + schema + retry-once; extraction later).
- Config: Lua `init.lua` with `mill.setup` and `mill.agent`, settings registry, layered precedence, `config get --why` ([configuration.md](configuration.md)).
- Parts: `sh`, `exec`, `check`, `map`, `filter`, `take`, `split`, `jq`, `if`, `assert`, `fail`, `noop`, `agent`, `stdin`, `stdout`; full `partproto` (Open/Session/Emitter) with loopback and conformance.
- Observability basics: `mlog` with fields, `metrics` registry with the L3/L4 instruments ([observability.md](observability.md) §5).
- CLI: `mill run --local` (+ `--stdin`, `--each`, `--json`, `--dry-run` skeleton), `mill lint`, `mill explain`, `mill runs --store`.
- `clock` fake, `testenv`, echo agent, testscript harness, CI gate on three OSes.

**Packages by layer:** L0 `errs clock paths mlog version metrics lock proc redact wire/jsonl` (complete) · L1 `core core/event core/fold partproto partproto/loopback partproto/conformance` (complete) · L2 `expr schema pipeline pipeline/ir` · L3 `store store/cas` · L4 `engine engine/sched engine/budget engine/retry runner runner/local secrets` · L5 `parthost parts/builtin/* agents config config/settings luaapi` · L7 `surface/cli surface/cli/core surface/cli/local` · T `testutil archtest`.

**Fitness functions added:** archtest A4, A5, A7, A9 (third-party allowlists), A11 (exception ratchet with `internal/version/milestone.txt`), A14; every depguard rule in [layers.md](layers.md) §8.2; every forbidigo rule; coverage floors per layer (D39); `errors-check` with code-range-by-layer; goleak in every `TestMain`; `sched.AssertDrained` in every engine test; benchmarks B1–B8, B11, B16 baselined.

**Exit criteria:**
1. Example suite (`hello`, `each-fanout`, `retry-rate-limit`, `budget-stop`, `cancel`) passes on Linux, macOS, Windows (`--local`; Windows is tier 2, ADR-030).
2. `nightly-vuln-sweep-lite` moves to `mill run --local` in the crontab; `morning-brief-lite` (`sh` curl two RSS feeds → `agent claude/haiku` "summarize" → `sh` append to `~/notes/brief.md`) joins it.
3. `mill run x --local --budget-tokens 1k` exits 9 with M4101; Ctrl-C leaves no child processes (checked by the test with `@echo:spawn-child`).
4. Invariants 4–9 tests green; engine coverage ≥ 85 %, `core/fold` ≥ 95 %.
5. Benchmarks recorded as baselines; `make check` green with the M1 fitness functions.

## M2 — Daemon

**Scope:**
- `milld`: boot wiring, `daemon.lock`, socket server (rex copy: conns, peercred, bounded queues), handshake, intents `Hello StartRun RunInput RunInputEnd CancelRun DeleteRun RerunFrom RetryNode Subscribe Unsubscribe ListRuns GetRun GetItems GetLogs ListPipelines GetPipeline Lint Reload Shutdown`.
- `assemble` (L6, D32): the wiring shared by `daemon/boot` and `surface/cli/local`.
- `daemonctl` autostart from `mill` (rex copy), `mill daemon start|stop|status|restart|logs` ([operations.md](operations.md) §1).
- Pipeline catalog with fsnotify reload (invariant 10); config reload with last-good ([configuration.md](configuration.md) §5).
- Crash recovery: live markers, replay, orphan kill by identity, §15.7 table, `RunRecovered` ([operations.md](operations.md) §3).
- Index (SQLite): schema, write-behind indexer, rebuild via `core/fold`, selectors (`#n`, `#last`, prefix).
- Status file + `mill prompt`, `mill status`.
- Stall detection for the scheduler, run actors and indexer ([concurrency.md](concurrency.md) §5); ordered shutdown (§4 there).
- CLI: `run` (wait/detach via daemon), `runs`, `logs [-f]`, `items`, `wait`, `cancel`, `rm`, `rerun [--from]`, `watch`, `events`.
- Lua hooks in milld (`mill.on`) through the rex dispatcher pattern, via the `luaapi.Host` port.

**Packages by layer:** L1 `wire/protocol` · L3 `store/index` · L5 `wire/client daemonctl` · L6 `daemon/server daemon/boot daemon/status assemble` (new layer) · L7 `surface/cli` (daemon commands).

**Fitness functions added:** exception ratchet: the `surface/cli/local → engine, store, parthost, parts, agents, runner` exceptions are removed and `L7-cli-local-root` becomes strict with `assemble` only (A11 fails if they linger); A12 (`milld` doesn't link TUI or audio); rebuild equivalence (live index == rebuilt index); golden wire messages; benchmarks B9, B10, B12–B15.

**Exit criteria:**
1. Kill -9 milld mid-run in a test: idempotent node re-runs, non-idempotent node `crashed` M4301, `mill rerun --from` finishes it.
2. Two clients see the same events; a stalled client is dropped without slowing the other (rex test pattern).
3. `mill run p --json | jq` works identically with and without `--local` (both through `assemble`).
4. B9, B10, B12–B15 within budget; targets O1–O4 measurable in `mill stats --health` output from the registry.
5. Shutdown with runs in every state is goleak-clean within 5 s.

## M3 — Code pipelines

**Scope:**
- Workspace manager: git worktrees, temp dirs, cleanup policies, retry snapshot/reset, diff artifacts, path safety with `os.Root`.
- Parts: `git.worktree`, `git.diff`, `git.commit`, `git.push`, `gh.pr`, `gh.issue`, `gh.comment` (via `gh` CLI), `handoff`, `agent.loop`, `agent.review`, `agent.extract`, `file.read|write|append|glob`, `template`, `merge`.
- Secrets: broker with `keychain` backend, `mill secrets set|ls|rm|test`, redaction everywhere, M9207 argv checks.
- Policy: `allow_destructive` lint, `--dry-run` for destructive parts.
- Project trust: `mill trust`, untrusted projects data-only, trust-widening keys ignored from project config.

**Packages by layer:** L4 `workspace secrets` (backends) · L5 `parts/builtin/{git,gh,file,template,merge,handoff}`, agent parts, trust in `parthost`.

**Fitness functions added:** security fitness functions ([security.md](security.md) §8.1): `secrets.Value` reflection test, `TestSecrets_NotOnDisk`, `TestErrors_Redacted`, `TestConfig_ProjectCannotWidenTrust`; loopback conformance for every new built-in; A6 for every new built-in package.

**Exit criteria:**
1. `nightly-vuln-sweep` full form (`govulncheck` → `if` → `agent.loop` until `go test ./...` → `gh.pr`) runs nightly from the user's crontab for a week with no manual intervention.
2. `issue-to-pr` (spec §7.5 minus webhook and Slack) opens a draft PR on a sandbox repo from `--input issue=@issue.json` in an integration test; out-of-scope plans produce an issue comment. Its first real dogfood run waits for M4.
3. `TestSecrets_NotOnDisk` and crash-point exploration green for both examples.

## M4 — Triggers and MCP client

**Scope:**
- Trigger host: arm/disarm on pipeline reload, persistent state (`triggers/*.json`), concurrency policies (`queue`, `skip`, `replace`, `parallel N`), dedupe windows, bounded queued fires.
- Triggers: `cron` (tz, jitter, `catch_up`), `schedule.once`, `poll` (any node part, dedupe key), `run` (another run's done/failed), `manual`.
- MCP client: `mill.mcp` config, server lifecycle in parthost, `mcp:<server>/<tool>` nodes with schemas from `tools/list`, `mill mcp ls`.
- Parts: `http` (pagination helpers), `parse` (json/yaml/csv/xml/html/front-matter), `notify` (desktop: `osascript`/`notify-send`/toast), `wait`, `sort`, `uniq`, `skip`, `batch`, `reduce`.
- Circuit breakers per agent id and HTTP host; `agent:<id>` slot pools (D36, [resilience.md](resilience.md) §6).
- `mill trigger ls|enable|disable|fire`.

**Packages by layer:** L4 `engine/sched` (breaker holds, `agent:<id>` pools) · L5 `trigger trigger/{cron,poll,runevent,once}`, MCP in `parthost`, `parts/builtin/{http,parse,notify,…}`.

**Fitness functions added:** timeout-table tests for MCP, HTTP and part rows ([resilience.md](resilience.md) §10); `TestBreaker_HoldsQueue_NoAttemptSpent`; `TestBulkhead_AgentPoolIsolation`; cron under `synctest` (DST, sleep, `catch_up`); antipattern checklist review for MCP and HTTP integration points.

**Exit criteria:**
1. `morning-brief` full form (cron 07:30 weekdays, RSS + calendar MCP + Slack MCP) runs a week; missed runs during sleep follow `catch_up: last`.
2. `nightly-vuln-sweep` moved from crontab to mill cron.
3. `issue-to-pr` starts dogfooding on a `poll` trigger (`gh issue list --label agent --json number,title,body` every 10 min, dedupe by number).

## M5 — CLI completeness and pipe

**Scope:** pipe expressions (`mill pipe`, `--save`, `--dry-run`), `pipeline/edit` + `yamledit` + `mill node|edge` commands + `EditPipeline` intent, `mill new` (flags + terminal prompts, `describe:` via headless agent), `mill show --outline|--yaml|--graph`, `mill render` (text, mermaid), `mill ls`, completion (bash/zsh/fish), `mill doctor` (+ `--bundle`, `--health`, `--reindex`; [operations.md](operations.md) §4–5), `mill setup` (CLI flow), `mill stats` (+ `--health`, `--by`, `--retries`, `--slow`), `mill digest` ([observability.md](observability.md) §8), `mill cache`, `mill prune`, `mill config list|get|set|reset|edit|check`, `mill lua-types`, `mill agent ls|test`, `mill part ls|info`, `mill mcp serve` (`surface/mcpserver`).

**Packages by layer:** L2 `pipeline/pipe pipeline/edit pipeline/yamledit pipeline/layout` · L4 `engine/cache` · L7 `surface/cli` (all commands), `surface/mcpserver`.

**Fitness functions added:** goldens for every command in every output mode; `--json` output goldens as contracts ([versioning.md](versioning.md) §6); doctor check goldens per layer; `FuzzYAMLEdit`, `FuzzPipeExpr`; `TestStats_MatchesRebuild`.

**Exit criteria:** spec §26 pipe one-liners work; §17.11 one-liners work; every command has golden tests in all output modes; `mill doctor` reports every check in [operations.md](operations.md) §4 that exists by M5.

## M6 — Backend for the TUI

**Scope:** PTY runner (rex console copy, supervisor, vt10x mirror, auto-answer, M6304 detection, transcripts), `Subscribe{repaint}` + `Resize`, `NodeOutput` coalescing, `Snapshot` board fields (`text` column, next fire, long-run p50 durations), `Stats`/`ListTriggers`/`ListParts`/`PartCall`/`RunNode`/`Estimate`/`DraftPipeline` intents, `ConfigStatus` events, lint-on-save.

**Packages by layer:** L4 `runner/pty` · L5 `agents/detect`, `drafter` · L6 `daemon/server` (intents).

**Fitness functions added:** A12 re-checked with the PTY stack (vt10x allowed in `milld`, Charm libraries still not); event coalescing tests; `NodeOutput` repaint golden.

**Exit criteria:** `mill attach run/node` shows a terminal-only agent's screen at once (repaint) on macOS and Linux; an agent that asks a question fails M6304 quoting it within `idle_ms + 1 s`.

## M7 — Gateway, webhooks, fswatch, git hooks, stream mode

**Scope:** `gateway` (HTTP + WS, tokens, limits), `webhook` trigger with all verifiers and `ack|wait` response modes, `fswatch`, `git` hooks (`mill hooks install`), `stream` mode with windows and credits, `n8n.call`/`zapier.call` with callbacks, `otel` and `prometheus` sink parts (D35).

**Packages by layer:** L6 `gateway` (new) · L5 `trigger/{webhook,fswatch,githook}`, sinks · L4 stream mode in `engine`.

**Fitness functions added:** gateway security tests (forged signatures, replay, oversized bodies, rate limit); `TestOtelSink_SpansFromEvents`; `TestMetrics_Cardinality`; sink-drop tests; stream backpressure tests.

**Exit criteria:** a GitHub webhook (via a tunnel) starts `issue-to-pr`; `notes-to-post` (fswatch) runs.

## M8 — Loops, fanout, cache, retention

**Scope:** generic `loop` (`with.body` node list, ADR-032), `pipeline` part, `agent.fanout`, `agent.judge`, cache keys + `mill cache`, retention/GC job with grace and ordering, disk-pressure behavior, zstd log archiving ([data-lifecycle.md](data-lifecycle.md)).

**Packages by layer:** L3 `store` (zstd), GC in `store/cas` · L4 `engine` (loop, sub-runs), `engine/cache` · L5 `parts/builtin/{loop,pipeline,agent.fanout,agent.judge}`.

**Fitness functions added:** `TestGC_GraceProtectsInFlight`, `TestPrune_SkipsLiveAndParents`, `TestDiskLow_StopsGrants`, `TestBackup_CopyMidRun_Restore`; disk soak.

**Exit criteria:** `best-of-3` and `multi-repo-refactor` run; GC keeps disk bounded over a month of dogfooding.

## M9 — Plugins (v0.5)

**Scope:** public parts protocol v1 + conformance suite as a published test kit, Go SDK, Lua parts with permission enforcement, plugin manifest/resolver/installer/lockfile, `mill plugin *`, `mill import|export` bundles with secret scanning, OpenAPI plugin generator, container and ssh runners, `plugins.sandbox = "container"`, registry index on GitHub (ADR-043).

**Packages by layer:** L4 `runner/container runner/ssh` · L5 `plugins parthost/luapart` · `sdk/go` (separate module).

**Fitness functions added:** A13 (SDK doesn't import mill); conformance suite run against the SDK; lockfile and permission-diff tests; Lua sandbox escape tests.

**Exit criteria:** spec v0.5 "done when".

## M10 — Interop and remote (v0.6, v1)

n8n import/export, mermaid/gha exports, `external` runner adapters (`mill/rex` plugin), remote runners, relay client, run viewer (HTML + SVG). Layers: L5 `runner/remote`, `runner/external`, importers. Fitness: wire and parts v−1 compatibility tests ([versioning.md](versioning.md) §6).

## M11 — Hardening (continuous after M4)

Windows PTY parity, `CREATE_SUSPENDED` job assignment (`proc`), Linux cgroup scopes, plugin signatures, WASM parts (wazero), `mill daemon install` (launchd/systemd), long-duration soak tests (disk, goroutines, memory flat over a week, [resilience.md](resilience.md) §8).

## Sources

| Decision | Follows |
|---|---|
| Every milestone ends runnable, integrated and gated by CI | [SEG ch. 23] |
| Fitness functions per milestone, ratcheting exceptions | [BEA ch. 2] [BEA ch. 4] |
| Antipattern review per new integration point | [RI ch. 4] |
| Thin vertical slice through every layer first (M0) | this plan's own rule: M0 proves the layering end to end before any layer widens |
