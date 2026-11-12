# Observability

Status: review (2026-11-12 refinement, awaiting the user); D35 (metrics registry) in [open-decisions.md](open-decisions.md). Spec: [§24](../../spec/24-observability.md#24-observability-and-cost), [§17](../../spec/17-cli.md#17-cli-reference), [§18.12](../../spec/18-tui.md#1812-screensaver-long-runs-detach) (`:stats`).

How you find out what mill did and how `milld` itself is doing: logs, events, traces, metrics, the golden signals, local targets, and what `mill stats`, `mill digest` and `:stats` show. Layers per [layers.md](layers.md).

## 1. Principles

1. **The event log is the telemetry.** Every run already writes wide, structured events with timestamps, ids, states, usage and errors [OE ch. 5]. Traces, stats and digests are derived from it, never instrumented a second time.
2. **Local first** (principle 15). Everything is on disk and in the daemon. Export (OpenTelemetry, Prometheus) is an optional sink part that can fail without touching a run (principle 10).
3. **Two audiences, two stores.** Run telemetry (what your pipeline did) lives with the run. Daemon telemetry (how `milld` is doing) lives in `milld.log` and the metrics registry.
4. **Bounded cardinality for metrics.** Labels come from small, config-bounded sets (pool names, agent ids, states, classes). Run ids and node ids are never labels; they are in events and logs.

## 2. Signals at a glance

| Signal | Store | Written by (layer) | Read with |
|---|---|---|---|
| Run events | `runs/<id>/events.jsonl` | run actor (L4) via `store` (L3) | `mill events --json`, `mill watch`, run popup |
| Step output | `logs/<nr>.out.log`, `.err.log`, `.part.log` | worker (L4), redacted | `mill logs [-f] [--raw]` |
| Daemon log | `~/.local/state/mill/milld.log` (JSON lines) | every component via `mlog` (L0) | `mill daemon logs [--run <id>] [--level warn]` |
| Client logs | `logs/tui.log`, `logs/cli-errors.log` | L7 | support bundle |
| Metrics | in-process registry (`metrics`, L0) | each layer registers its own | `Stats` intent: `mill stats --health`, `:stats`, Prometheus sink |
| Traces | derived from run events | `otel` sink part (L5) | any OTLP backend |

## 3. Logs

`mlog` writes one JSON object per line [OE ch. 5]:

```json
{"ts":"2026-11-12T03:30:02.114Z","level":"WARN","msg":"part restarted","component":"parthost","part":"mill/linear","attempt":2,"backoff_ms":4000,"err_code":"M5201"}
```

- **Fields:** `ts`, `level`, `msg`, `component` (package name), and when present `run`, `nr`, `req` (intent id), `err_code`, `class`. Messages are short and constant (`"part restarted"`), variables go in fields, so `jq` and grep work.
- **Levels:** `ERROR` needs a look (`mill doctor` counts them since start); `WARN` degraded but handled; `INFO` lifecycle (start, stop, reload, recovery decisions); `DEBUG` only with `--debug` or `MILL_DEBUG=1`.
- **What never goes in the daemon log:** step output (it's in run logs), item payloads, secret values (the daemon redactor wraps the handler), stack traces except for M0001 and recovered panics.
- **Rotation:** 10 MiB × 5 files, 0600 ([data-lifecycle.md](data-lifecycle.md)).
- **Correlation:** every log line about a run carries `run` and `nr`; `mill daemon logs --run issue-to-pr#42` filters by the resolved id [BSRS ch. 15].

## 4. Traces derived from events

The `otel` sink (spec §24; built-in part from M7, D35) turns a run's events into OpenTelemetry spans [OE ch. 6] [OE ch. 7]:

| Span | Starts at | Ends at | Key attributes |
|---|---|---|---|
| run (root) | `RunCreated` | `RunFinished` | `mill.pipeline`, `mill.run.number`, `mill.trigger.kind`, `mill.state`, `mill.tokens.in/out`, `mill.cost_micro_usd`, `mill.error.code` |
| node-run (child) | `NodeRunStateChanged → queued` | `NodeRunFinished` | `mill.node`, `mill.part`, `mill.runner`, `mill.attempt`, `mill.items.in/out`, `mill.exit_code`, `mill.error.class` |
| span events | `queued → working` (queue wait), `RetryScheduled`, `BudgetWarning`, `Checkpoint`, `LoopIteration` | | |

- The run id is 128 bits, the size of a W3C trace id, so `trace_id = run id` and a trace is findable from `mill runs`. Span ids are the first 64 bits of `sha256(run id + node-run id)`, so re-exporting is idempotent.
- Child runs (`pipeline` part, loops) link to the parent span.
- Spans are emitted from durable events only, so an exported trace never shows something a crash could undo.

## 5. Metrics

`metrics` (L0, stdlib only, D35) holds counters, gauges and fixed-bucket histograms. Each layer registers its instruments at construction; the registry snapshot travels in the `Stats` reply (`group_by: "health"`) and a Prometheus sink can serve it on localhost.

| Metric | Type | Labels | Layer |
|---|---|---|---|
| `milld_build_info` | gauge | `version`, `protocol`, `log_v` | L6 |
| `runs_started_total` | counter | `trigger_kind`, `pipeline` | L4 |
| `runs_finished_total` | counter | `state`, `pipeline` | L4 |
| `node_runs_finished_total` | counter | `state`, `class`, `part_kind` (`agent`, `exec`, `data`, `control`, `external`) | L4 |
| `node_run_duration_seconds` | histogram | `part_kind` | L4 |
| `sched_queue_depth` | gauge | | L4 |
| `sched_wait_seconds` (ready → grant) | histogram | `pool_bound` (the pool that held it last) | L4 |
| `slots_in_use`, `slots_max` | gauge | `pool` (`runs`, `nodes`, `agents`, `agent:<id>`, `runner:<id>`) | L4 |
| `breaker_state` (0 closed, 1 half-open, 2 open) | gauge | `key` | L4, L5 |
| `tokens_total` | counter | `agent`, `model`, `dir` (`in`, `out`, `cache`) | L4 |
| `cost_micro_usd_total` | counter | `agent`, `model` | L4 |
| `store_append_seconds` | histogram | `durability` (`sync`, `group`) | L3 |
| `store_fsync_slow_total` | counter | | L3 |
| `index_lag_ops`, `index_dirty_runs` | gauge | | L3 |
| `disk_free_bytes` | gauge | `dir` (`data`, `state`, `workspaces`) | L3 |
| `intent_seconds` | histogram | `type` | L6 |
| `event_delivery_seconds` (append → written to socket) | histogram | | L6 |
| `conns_active` | gauge | `client` (`cli`, `tui`, `mcp`, `gateway`) | L6 |
| `conns_dropped_total` | counter | `reason` (`slow`, `line_too_long`, `version`) | L6 |
| `protocol_errors_total` | counter | `code` | L6 |
| `parts_live`, `part_restarts_total` | gauge, counter | `part` | L5 |
| `triggers_fired_total` | counter | `kind`, `outcome` (`started`, `skipped`, `queued`, `deduped`) | L5 |
| `sink_dropped_total` | counter | `sink` | L4 |
| `stalls_total` | counter | `component` | L4, L3 |
| `goroutines`, `heap_bytes`, `open_fds` | gauge | | L6 (sampled every 10 s) |

## 6. Golden signals for `milld`

The four signals [SRE ch. 6], applied to a local daemon, with the indicators that measure each:

| Signal | Indicator (SLI) | From |
|---|---|---|
| **Latency** | intent reply time; event delivery time (append → socket); scheduling wait when slots are free | `intent_seconds`, `event_delivery_seconds`, `sched_wait_seconds` |
| **Traffic** | runs and node-runs started per minute; intents per second; events appended per second | counters above |
| **Errors** | node-runs failed by class (`transient`, `rate_limited` and `internal` matter for mill itself; `user` and `agent` are the pipeline's); protocol errors; M0001 count | `node_runs_finished_total`, `protocol_errors_total`, log `ERROR` count |
| **Saturation** | slot use per pool; queue depth; indexer lag; connection queue fill; disk free; goroutines; fds | gauges above, plus USE checks in [performance.md](performance.md) [SP, USE method] |

## 7. Local targets

Targets in the style of SLOs [SRE ch. 4] [SRW ch. 2] [OE ch. 12], for mill's own machinery, not for users' pipelines (a pipeline that fails because a test fails is mill working). They are engineering budgets: benchmarks check them in CI ([performance.md](performance.md)) and `mill doctor --health` checks them on a live install. There is no paging and no error-budget policy [SRE ch. 3] (departure noted in [references.md](references.md)).

| # | Target | Measured by | Checked by |
|---|---|---|---|
| O1 | Non-streaming intent reply p99 ≤ 50 ms (excluding `StartRun` cold compile, `Prune`, `Reload`) | `intent_seconds` | B13, doctor |
| O2 | Event visible to subscribers p99 ≤ 100 ms after its append returns | `event_delivery_seconds` | B14, doctor |
| O3 | Ready node-run granted p99 ≤ 10 ms when its pools have capacity | `sched_wait_seconds` | B7 |
| O4 | Daemon start to first `Snapshot` ≤ 300 ms with 1k finished runs; recovery of 100 live runs ≤ 2 s | boot timing in `milld.log` | B10, B15 |
| O5 | Index at most 1 s behind the logs at p99 | `index_lag_ops`, rebuild-equivalence | doctor |
| O6 | Zero lost terminal events and zero duplicate non-idempotent executions across crash-point exploration | test harness | nightly stress |
| O7 | Idle daemon: < 1 % CPU, RSS ≤ 60 MiB | `heap_bytes`, OS | doctor |
| O8 | Internal errors: zero M0001 per week of dogfooding | log `ERROR` with `err_code=M0001` | digest |

## 8. What the commands show

### 8.1 `mill stats` and `:stats`

Spec §18.12 (`:stats`) and §17.1: one table per pipeline with runs, success rate, p50 and p90 duration and cost for today, 7 d and 30 d, totals on the last line. Backed by the `Stats` intent over the index (`daily` rollup table and `runs`).

| Flag | Adds |
|---|---|
| `--since 30d`, `--by pipeline|agent|model|node` | grouping (default `pipeline`) |
| `--retries` | retries and their classes per pipeline; top retried nodes |
| `--slow` | slowest nodes by p90 with their share of run time |
| `--health` | the daemon block: golden signals (§6), targets O1–O8 with pass/warn, slot use per pool, breakers, indexer lag, disk free, goroutines, uptime |
| `--json` | everything as one JSON object (stable field names, [versioning.md](versioning.md)) |

`--by`, `--retries`, `--slow` and `--health` (and `mill daemon logs --run`) are additions to spec §17.1, proposed as D40.

### 8.2 `mill digest`

A day in one screen (spec §24), text by default, `--json` for scripts, `--date d` for another day. Sections in order: headline (`14 runs · 1 failed · $4.10 · 2 retries`); failures with cause, code and the `help:` line; budget stops; crashed runs and recovery decisions; slowest nodes; spend by agent and model (unknown prices called out); breaker openings and part restarts; disk use change and what `retention` will prune next; tomorrow's scheduled fires. `morning-brief` can include it by running `mill digest --json` in an `sh` node.

### 8.3 `mill doctor`

The health checks are listed in [operations.md](operations.md) §4. Observability adds `--health`, which evaluates O1–O8 from the live registry.

## 9. Debugging path

The intended way to answer "why did this run do that" [OE ch. 5] [BSRS ch. 15]:

1. Board row text (cause in a few words) → run popup (failed node red, cause chain).
2. `mill logs <run>/<node>` around the failure line; `--raw` for exact bytes.
3. `mill events <run> --json | jq` for the exact sequence: states, retries, grants, recovery decisions.
4. `mill daemon logs --run <run>` for what `milld` did around it (part restarts, breaker, slow fsync).
5. `mill replay <run> --node <node>` to run that node again in isolation (spec §15.9).

## 10. Tests

| Test | Guards |
|---|---|
| Golden `milld.log` lines for lifecycle events (fake clock) | §3 field names |
| `TestOtelSink_SpansFromEvents`: golden spans for a deterministic example run | §4 |
| `TestMetrics_Cardinality`: run 100 runs, label sets stay bounded | §1 rule 4 |
| `TestStats_MatchesRebuild`: stats from the live index equal stats after rebuild | §8.1 |
| Golden `mill digest` for a fixture day | §8.2 |
| Benchmarks B13–B15 | §7 |

## 11. Sources

| Decision | Follows |
|---|---|
| Wide structured events as the base signal | [OE ch. 5] |
| Traces derived from events; OTLP export | [OE ch. 6] [OE ch. 7] |
| Four golden signals for the daemon | [SRE ch. 6] [SRW ch. 4] |
| SLO-style targets without SLAs or paging | [SRE ch. 3] [SRE ch. 4] [SRW ch. 2] [OE ch. 12] |
| USE method for resources | [SP, USE method] |
| Correlated logs and investigation path | [BSRS ch. 15] |
| Optional exporters never block core | principle 10; [RI ch. 5, Bulkheads] |
