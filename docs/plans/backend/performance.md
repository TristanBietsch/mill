# Performance and capacity

Status: review (2026-11-12 refinement, awaiting the user). Owns the benchmark budgets that [testing.md](testing.md) §7 used to list (B1–B12, unchanged) and adds B13–B16.

What has to be fast, how fast, how big mill is expected to get on one machine, and how it's checked. Layers per [layers.md](layers.md).

## 1. Principles

1. **Budgets are fitness functions** [BEA ch. 2]: each hot path has a number and a benchmark, and `make bench-check` fails on a > 10 % regression against `bench/BASELINE.md` (rex's tooling).
2. **The daemon's cost is the overhead around the work.** Agent calls take minutes; mill must add milliseconds. Targets cover mill's own machinery: compile, append, fold, grant, deliver, render.
3. **Flat memory.** Items stream through files; nothing holds a node's whole output in RAM (engine.md §3). Memory grows with live node-runs and connections, not with history.
4. **Measure resources with USE** [SP, USE method]: utilization, saturation and errors for CPU, memory, disk, file descriptors and goroutines.

## 2. Hot paths and budgets

| Id | Path | Layer | Benchmark | Budget |
|---|---|---|---|---|
| B1 | Encode + decode one event envelope | L0 `wire/jsonl` | `BenchmarkCodec_EventRoundTrip` | ≤ 12 µs, ≤ 30 allocs (rex is 11 µs) |
| B2 | Event append, group commit | L3 `store` | `BenchmarkEventAppend_Group` (1k events) | ≤ 5 µs/event amortized |
| B3 | Replay a run | L1 `core/fold` + L3 | `BenchmarkReplay_10kEvents` | ≤ 30 ms |
| B4 | Compile a pipeline | L2 `pipeline` | `BenchmarkCompile_50Nodes` | ≤ 5 ms cold, ≤ 50 µs cached |
| B5 | Evaluate a template | L2 `expr` | `BenchmarkCELEval_Template` | ≤ 5 µs per `${{ }}` |
| B6 | Write/read items | L3 `store` | `BenchmarkItems_Write100k_Small` / `Read` | ≥ 200k items/s |
| B7 | Scheduler grant pass | L4 `engine/sched` | `BenchmarkScheduler_1kReady` | ≤ 2 ms per pass |
| B8 | Redact a stream | L0 `redact` | `BenchmarkRedactor_20Secrets` | ≥ 200 MB/s |
| B9 | Rebuild the index | L3 `store/index` | `BenchmarkIndexRebuild_10kRuns` (integration) | ≤ 20 s |
| B10 | Daemon start to `Snapshot` | L6 | `BenchmarkDaemonStart_1kRuns` (integration) | ≤ 300 ms |
| B11 | `mill run --local` end to end, two `sh` nodes | L7 → L0 | `BenchmarkLocalRun_Hello` (integration) | ≤ 150 ms wall (pipe ergonomics) |
| B12 | `mill prompt` | L7 + L6 status file | `BenchmarkStatusPrompt` | ≤ 2 ms (spec §17.11 says ~1 ms) |
| B13 (new) | `ListRuns` intent, 1k runs, limit 100 | L6 → L3 | `BenchmarkIntent_ListRuns` | ≤ 5 ms (O1 is 50 ms p99) |
| B14 (new) | Event append → written to 3 subscriber sockets | L4 → L6 | `BenchmarkEventDelivery_3Subs` | ≤ 1 ms median (O2 is 100 ms p99) |
| B15 (new) | Recover 100 live runs (replay + decisions, fake processes) | L4 | `BenchmarkRecover_100Live` (integration) | ≤ 2 s (O4) |
| B16 (new) | Cache key for 1 MiB of input items | L4 `engine/cache` | `BenchmarkCacheKey_JCS_1MiB` | ≤ 10 ms |

O1–O8 are the live targets in [observability.md](observability.md) §7. Budgets B1–B12 are initial targets from the M1 plan; M1 records baselines and the budgets tighten to measured + 20 % (ratchet).

## 3. Capacity model

Sized for one heavy user with a margin of 10×.

| Quantity | Expected (dogfood) | Designed for | What bounds it |
|---|---|---|---|
| Runs per day | ≤ 100 | 1,000 | `max_concurrent_runs`, `max_queued_runs` |
| Runs on disk | ≤ 3,000 (30 d) | 30,000 | `retention`; B9 rebuild time |
| Nodes per pipeline | ≤ 20 | 200 | `budget.max_nodes`; B4 compile |
| Events per run | ≤ 2,000 | 20,000 | B3 replay |
| Live runs | ≤ 8 | 64 | `max_concurrent_runs` |
| Live node-runs | ≤ 16 | 128 | `max_concurrent_nodes` |
| Connected clients | ≤ 5 | 50 | per-connection queues |
| Pipelines | ≤ 50 | 500 | catalog reload time |

**Disk per run** (median, excluding artifacts): events ≈ 300 B × events, items as produced, logs dominated by agent output (1–5 MiB per agent node-run). Budget: median run ≤ 2 MiB, so 30 days at 100 runs/day ≈ 6 GiB before M8's zstd log compression (expected 4–6× smaller for text). `mill doctor` projects disk use from the last 7 days.

**Memory** (RSS of `milld`, children excluded):

| Component | Budget |
|---|---|
| Idle daemon with 50 pipelines and 3k runs indexed | ≤ 60 MiB (O7) |
| Per live run (actor state, ledger, redactor) | ≤ 2 MiB |
| Per working node-run (worker, pumps, buffers) | ≤ 512 KiB plus pipe buffers |
| Per connection | ≤ 1 MiB at a full 1024-line queue of small events |
| Compile cache | ≤ 32 MiB (LRU by bundle hash) |

**Goroutines:** about 2 + 3 per working node-run (worker, stdout and stderr pumps, stdin writer when fed) + 2 per connection + 1 per armed trigger + fixed owners (scheduler, indexer, bus helpers, watchers). 128 live node-runs and 50 clients ≈ 600.

**File descriptors:** about 6 per working node-run (3 pipes, 3 log files) + 1 per connection + SQLite and locks. At boot `milld` raises its soft `RLIMIT_NOFILE` to min(hard, 8192) on Unix and warns in `mill doctor` if the result is under 1024.

## 4. USE checks

`mill doctor --health` and `mill stats --health` evaluate these from the metrics registry ([observability.md](observability.md) §5):

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| Slots (per pool) | `slots_in_use / slots_max` | `sched_queue_depth`, `sched_wait_seconds` p99 | breaker open |
| CPU (milld) | process CPU % | run-actor inbox depth, stall counter | — |
| Memory | `heap_bytes` vs budget | GC pause p99 | — |
| Disk | `disk_free_bytes` | `store_fsync_slow_total`, indexer lag | ENOSPC count |
| FDs | `open_fds / limit` | — | EMFILE count |
| Connections | `conns_active` | outbound queue fill | `conns_dropped_total` |

## 5. Profiling

- `milld --debug-addr 127.0.0.1:0` (off by default) serves `net/http/pprof` on loopback only, printing the chosen port to `milld.log`.
- `mill doctor --bundle --profile 30s` adds a CPU profile, a heap profile and a goroutine dump to the support bundle (redacted metadata only; profiles contain no payloads).
- Benchmarks run with `-benchmem`; `benchstat` compares against the baseline in the `perf` CI job.

## 6. Tests

| Test | Guards |
|---|---|
| `make bench bench-check` (CI label `perf`, nightly) | §2 |
| `TestMemory_FlatOverItems`: 1M small items through a three-node pipeline; RSS growth ≤ 20 MiB | rule 3 |
| `TestCapacity_128NodeRuns`: fake parts, all slots busy, no stalls, goleak clean | §3 |
| `TestRlimit_Raised` (Unix) | §3 |

## 7. Sources

| Decision | Follows |
|---|---|
| Benchmarks with budgets as CI fitness functions; ratchet | [BEA ch. 2] [FSA ch. 6] |
| USE method per resource | [SP, USE method] |
| Golden signals tie budgets to live targets | [SRE ch. 6] |
| Flat memory by streaming through files | [DDIA ch. 10] |
| Load is bounded by queues, not by hope | [SRE ch. 21] |
