# Operations

Status: review (2026-11-12 refinement, awaiting the user). Spec: [§15.7](../../spec/15-execution.md#157-daemon-restarts-and-crash-recovery), [§29.9](../../spec/29-errors.md#299-common-failures-designed-in-advance), [§29.10](../../spec/29-errors.md#2910-mill-doctor), [§18.13](../../spec/18-tui.md#1813-boot-splash).

Running `milld` day to day: start and stop, upgrades, what happens after a crash, `mill doctor`, the support bundle, and a runbook for the situations designed in advance. The user is the operator; every procedure is one or two commands. Layers per [layers.md](layers.md).

## 1. Daemon lifecycle

| Action | Command | What happens |
|---|---|---|
| Start | any `mill` command, or `mill daemon start` | `daemonctl` (L5) finds `milld` next to `mill` or on PATH, spawns it detached (setsid / new process group, hidden console on Windows), waits up to 5 s for `Snapshot` (M7001 / M7004 on failure) |
| Status | `mill daemon status` | pid, version, uptime, data dir, live runs, armed triggers, config status |
| Stop | `mill daemon stop` | stop granting, exit within 5 s; live node-runs follow the crash rules on next start (ADR-042) [RI ch. 5, Let It Crash] |
| Restart | `mill daemon restart [--drain]` | stop then start; `--drain` waits for working node-runs up to `shutdown.drain_timeout` (10 m) [RI ch. 13] |
| Reload | `mill reload`, or edit a config/pipeline file | config and pipelines reload; failures keep the last good config ([configuration.md](configuration.md) §5) |
| Logs | `mill daemon logs [-f] [--run <sel>] [--level warn]` | `milld.log` filtered ([observability.md](observability.md) §3) |
| Run at login (later, M11) | `mill daemon install` | writes a launchd agent (macOS) or systemd user unit (Linux) that runs `milld`; until then autostart covers it |

Boot order is in [concurrency.md](concurrency.md) §4 (shutdown is its reverse); the boot splash shows the same steps (spec §18.13).

## 2. Upgrades

1. Install the new `mill` and `milld` together (Homebrew, Scoop, packages; both binaries ship in one archive).
2. `mill daemon restart --drain` (the CLI suggests it when it sees M7003).
3. On start, the new daemon reads every older event log, rebuilds the index if its schema changed, and re-describes parts whose binaries changed ([versioning.md](versioning.md) §4).

Downgrades work for runs the older version wrote; newer runs are refused one by one with M7007 and left intact.

## 3. Crash recovery

What happens when `milld` dies (crash, `kill -9`, power loss, `mill daemon stop`) [DBI ch. 5] [BSRS ch. 9]:

1. The next start takes `daemon.lock` (a second daemon exits with M7005 naming the holder).
2. For each run in `live/` whose `run.lock` is free (a `--local` run in progress keeps its lock and is skipped): replay `events.jsonl` with `core/fold`; truncate a torn last line after copying it aside; mark the run `crashed` with M4302 if anything before the last line is invalid.
3. Kill orphaned process groups whose recorded identity (pid, pgid, start time) still matches; remove containers by label.
4. Decide per node-run (spec §15.7): re-queue `queued`/`pending`/`retrying`; `Resume` resumable work; re-run idempotent work as a new attempt; mark everything else `crashed` (M4301), keeping transcripts and partial items.
5. Truncate uncommitted item tails; append `RunRecovered{decisions}`.
6. Start the server, arm triggers; missed cron fires follow each trigger's `catch_up` [SRE ch. 24].

What you see: the board shows recovered runs with their new state; `mill events <run>` shows `RunRecovered` with every decision; `mill rerun <run> --from <node>` continues a `crashed` run. Nothing with side effects is re-run without you asking (ADR-014).

Recovery is exercised continuously: crash-point exploration kills the engine at every append in every example pipeline (nightly), and the M2 exit test kills `milld` with SIGKILL mid-run ([testing.md](testing.md) §8).

## 4. `mill doctor`

One line per check, grouped by layer so a failure points at the right part of the system [RI ch. 10] [SRE ch. 12]. Each failed check has a fix command (spec §29.10). Exit 1 if any check fails.

| Layer | Check | Fail / warn example and fix |
|---|---|---|
| L0 | paths writable, socket path length, data dir on a local filesystem | `M2001 can't write ~/.local/state/mill` · fix permissions; `M7002` · set `MILL_SOCKET` |
| L0 | disk free in data, state and workspace dirs; 7-day growth projection | `M2002 not enough disk space` · `mill prune --older 14d` |
| L0 | file-descriptor limit ≥ 1024 | raise `ulimit -n` |
| L1 | event log versions present vs supported | `M7007` runs listed · upgrade mill |
| L2 | every pipeline lints; trust state per project | `M3xxx` with file:line · `mill lint <file>` |
| L3 | index schema, staleness vs logs, `PRAGMA integrity_check` (quick) | stale or corrupt → `mill doctor --reindex` |
| L3 | live markers without a lock or a daemon (stranded runs) | `mill daemon start` recovers them |
| L4 | agent CLIs installed and signed in (`agents.probe`), runners reachable (`docker info`, `ssh -O check`) | `M6101`, `M6202` with the registry's help line |
| L4 | secrets referenced by pipelines resolvable (names only, never values) | `M9202` · `mill secrets set NAME` |
| L4 | orphaned workspaces and stale worktrees (`git worktree prune --dry-run`) | `mill prune --workspaces` |
| L5 | plugins present, pinned, hash-matching | `M5301` · `mill plugin update <id>` |
| L5 | MCP servers start and list tools; part health | part restart loop · `mill part info <id>` |
| L5 | config loads; settings valid; restart-needed keys | `M8004` with file:line |
| L5 | trigger state files parse; next fire times | broken state → recreated with a warning |
| L6 | daemon reachable, same version, uptime, stalls since start, M0001 count | `M7003` · `mill daemon restart --drain` |
| L6 | gateway bind (when enabled): loopback unless `allow_public` | warning |
| — | `--health`: targets O1–O8 ([observability.md](observability.md) §7) | warn with the measured value |

Flags: `--reindex` (rebuild the index), `--bundle [--run <sel>] [--profile 30s] [--include-logs]` (§5), `--health`, `--json`. `--health`, `--profile`, `--include-logs`, `mill prune --workspaces` and `mill daemon install` are additions to spec §17.1, proposed as D40.

## 5. Support bundle

`mill doctor --bundle` writes one `.tar.gz` for bug reports (spec §29.10) [BSRS ch. 15]:

| Included | Notes |
|---|---|
| `doctor.json` | every check above |
| versions, OS, arch, Go version, build info | |
| effective settings | secrets stripped; references only |
| `milld.log` tail (last 5,000 lines), `cli-errors.log` tail | redacted by the daemon redactor |
| `Stats` health snapshot, goroutine dump | no payloads |
| with `--run <sel>`: that run's `events.jsonl`, `run.json`, lint of its frozen pipeline | item payloads and raw logs only with `--include-logs` |
| with `--profile`: CPU and heap profiles | |

Before writing, it prints the file list and sizes. Everything passes through the redactor; paths under `$HOME` are shown as `~`. Nothing is uploaded: the user attaches the file where they choose.

## 6. Runbook

Each row is a situation designed in advance (spec §29.9) and the one or two commands that resolve it.

| Situation | You see | Do |
|---|---|---|
| Daemon won't start | `M7001` with the reason; `M7005` lock held by pid N | `mill daemon logs`; if pid N is a stale `milld`, `mill daemon stop` |
| Version mismatch | `M7003` | `mill daemon restart --drain` |
| Run stuck `working` | board text `stalled` or no progress | `mill doctor` (stall report), `mill logs <run>/<node> -f`; `mill cancel <run>` ends it through teardown |
| Run crashed after a restart | `M4301` on the board | `mill rerun <run> --from <node>` |
| Agent rate limited for a long time | `queued · claude rate limited · resumes 12:41` | wait (breaker, [resilience.md](resilience.md) §6), or lower `max_concurrent_agents` |
| Agent signed out | `M6202` with the registry help | run the agent once to sign in; `mill agent test claude` |
| Part crash-looping | `PartStatus` unhealthy; `part_restarts_total` rising | `mill part info <id>`; `mill daemon logs --level warn` |
| Index wrong or corrupt | odd `mill runs` output; doctor flags staleness | `mill doctor --reindex` |
| Disk low | `disk low` on the board; `M2002` | `mill prune --older 14d --dry-run`, then without `--dry-run` |
| Config typo | one red line under the board header with file:line | fix the file; reload is automatic |
| Plugin changed on disk | `M5301` | `mill plugin update <id>` to accept, or reinstall |
| Bug in mill | `M0001` | `mill doctor --bundle`, attach to an issue |

## 7. Sources

| Decision | Follows |
|---|---|
| Stop is a crash; drain on request; recovery from the log | [RI ch. 5, Let It Crash] [RI ch. 13] [DBI ch. 5] [BSRS ch. 9] |
| Doctor as the control plane, grouped by layer | [RI ch. 10] |
| Troubleshooting from symptoms to cause | [SRE ch. 12] |
| Missed cron fires by policy | [SRE ch. 24] |
| Redacted, listed-before-written support bundle | [BSRS ch. 15] |
