# Resilience and stability

Status: review (2026-11-12 refinement, awaiting the user); D36 (breaker behavior) in [open-decisions.md](open-decisions.md).

How mill stays up and keeps runs honest when the things it drives misbehave: timeouts on every call out, retries with backoff, bounded queues and backpressure, bulkheads, circuit breakers, load shedding and a steady state on disk. Pattern names follow [RI ch. 5]; the failure modes they answer are [RI ch. 4]'s antipatterns. Layers per [layers.md](layers.md).

## 1. Principles

1. **Every integration point has a timeout, a bulkhead and a failure class** [RI ch. 4, Integration Points] [BM ch. 12]. mill's integration points are child processes, part processes, agent CLIs, MCP servers, HTTP endpoints, secret backends, `git`/`gh`/`docker`, SQLite and the filesystem.
2. **Hands-off changes the answer.** Nothing waits on a person, so a resilience mechanism may delay work (hold it in a queue) or fail it with a clear class, never park it waiting for input (ADR-001).
3. **Crash, don't guess** [RI ch. 5, Let It Crash] [BSRS ch. 9]: when state is uncertain (daemon died mid-step), record `crashed` and stop; never re-run side effects on a hunch (ADR-014).
4. **Bounded everything** [SRE ch. 22]: queues, retries, output sizes, disk use, goroutines. The overflow behavior of each bound is chosen and documented (§4).

## 2. Timeouts

Every call out of the process has a deadline [RI ch. 5, Timeouts]. Defaults are settings unless marked fixed.

| Call | Layer | Default | On expiry |
|---|---|---|---|
| Client dial + autostart wait | L5 `daemonctl` | 2 s dial, 5 s for a fresh `milld` to answer | M7001 with the reason and `help: mill daemon logs` |
| `Hello` → `Snapshot` | L5 `wire/client` | 5 s | M7001 |
| Intent reply (non-streaming) | L5 `wire/client` | 30 s; `mill wait --timeout` overrides for waits | class `timeout`, exit 7 |
| Connection write | L6 `daemon/server` | 5 s (fixed, rex) | client disconnected |
| Part `Describe` / `Open` | L5 `parthost` | 5 s / 30 s | M5201 `transient` |
| Part `Ping` → `Pong` | L5 `parthost` | every 30 s, 5 s to answer | restart with backoff, open contexts fail `transient` [DBI ch. 9] |
| Part `Cancel` → `Fail{cancelled}` | L4 `engine` | `cancel_grace` 10 s | process group killed |
| Node-run | L4 `engine` | `timeout`: 30 m non-agent, 2 h agent | teardown, `failed` class `timeout`, M4201 |
| Output drain after kill | L4 worker | 2 s or 1 MiB (fixed) | pipes closed, gap recorded |
| Run active wall time | L4 `engine/budget` | `budget.max_wall` (unset) | run cancelled, reason `budget`, M4102 |
| Secret backend `Get` | L4 `secrets` | 10 s per backend | next backend, then M9202 |
| `git` command | L4 `workspace` | 5 m (`workspace.git_timeout`) | node fails `transient` with the git stderr tail |
| `gh`, `docker`, `ssh` CLI call | L5 parts, L4 runners | inherits the node timeout; `docker pull` 10 m | per node policy |
| `http` part request | L5 | connect 10 s, total 60 s (`with.timeout`) | `transient` (5xx, connect) or `timeout` |
| MCP `initialize` / tool call | L5 `parthost` | 30 s / node timeout | `transient`; server restarted with backoff |
| CEL evaluation | L2 `expr` | 100 ms and cost 1 M (fixed) | M3110 |
| Lua: config phase / hook / transform | L5 `luaapi` | 2 s / 1 s / 1 s (node `timeout` for transforms) | M8101 or node failure |
| SQLite busy | L3 `store/index` | `busy_timeout` 5 s | read: retry once, then fall back to folding run files (slower, same answer); write: op re-queued |
| Gateway HTTP | L6 `gateway` | header 5 s, read 30 s, idle 60 s; webhook `wait` reply 30 s | 408/503 |
| Drafter (Ollama) | L5 `drafter` | health 1 s, generate 20 s | keyword fallback with M2212 (ADR-047) |
| Daemon stop | L6 `daemon/boot` | 5 s; `--drain` up to `shutdown.drain_timeout` 10 m | exit; recovery on next start |

A slow `fsync` can't be interrupted; appends over 1 s are logged and counted (`store_fsync_slow_total`) so `mill doctor` can point at a slow disk.

## 3. Retries and backoff

Node retries are specified in [engine.md](engine.md) §7 (policy grammar, default policy ADR-037, workspace reset ADR-044/D11). The rules that make them safe:

- **Only idempotent work or failures the part marks retryable** are retried by default; a side-effecting step is never re-run on a guess [BM ch. 12] [SRE ch. 24].
- **Full jitter** on every computed delay, seeded per node-run so fake-clock tests are deterministic. Retries from many runs against one provider don't synchronize [RI ch. 4, Dogpile].
- **`retry_after` wins** when the provider gives one.
- **Retry budget per run:** attempts count toward `budget.max_nodes`, so a retry loop can't run forever [RI ch. 5, Governor].
- **Infrastructure retries** (not node retries) use capped exponential backoff with jitter too: part and MCP server restarts (1 s → 60 s), `wire/client` reconnects (100 ms → 5 s), indexer re-fold (immediate, then 1 s).

## 4. Bounded queues and backpressure

The inventory of every queue in `milld` and `--local`. Overflow behavior is one of **block** (backpressure upstream), **drop** (count it), **coalesce** (keep the latest) or **refuse** (error to the sender) [RI ch. 5, Create Back Pressure] [CIG ch. 4, Queuing].

| Queue | Layer | Producer → consumer | Capacity | When full |
|---|---|---|---|---|
| Connection outbound | L6 | bus/handlers → conn writer | 1024 lines | **disconnect** the client; it reconnects and resubscribes `from_seq` |
| Run actor inbox | L4 | workers → actor | 1024 messages | **block**: `Emitter.Item` blocks, which stops reading the part's stdout, which blocks the part (pipe backpressure) |
| Scheduler requests | L4 | actors → scheduler loop | 4096 | **block** briefly (the loop never blocks, so it drains) |
| Grants | L4 | scheduler → actor | 256 per run (never fills: grants ≤ that run's outstanding requests) | invariant checked; overflow is M0001 |
| Indexer ops | L3 | actors → indexer | 4096 | **drop** the op and mark the run `index_dirty`; re-folded from its log later |
| Sink delivery | L4 | bus → sink worker | 1000 events per sink | **drop** and count (`sink_dropped_total{sink}`); optional never blocks core (principle 10) |
| Lua hook dispatcher | L5 | bus → dispatcher | 256, coalescing per hook and run (rex pattern) | **coalesce**, then drop oldest |
| Status writer | L6 | bus → status file | 1 | **coalesce** (latest wins) |
| Part input, long-lived parts | L5 | host → part (per `ctx`) | `Open.window` 64 items, credit-based | **block** that context only |
| Stream mode window (M7) | L4 | producer → consumer node | `stream.window` 256 items | **block** the producer (stop reading its stdout) |
| `RunInput` (streamed stdin) | L6 | client → daemon | one ≤ 1 MiB batch in flight | **block** the client until `Ack` |
| Trigger fires | L5 | arms → coordinator | 256 | **block** for cron/poll; webhook answers 503 (**refuse**) |
| Trigger `queue` policy | L5 | coordinator, persisted | `trigger.max_queued` 100 per trigger | **drop** oldest with `TriggerFired{skipped, reason: queue_full}` |
| Queued runs | L4 | `Engine.Start` | `max_queued_runs` 1000 | **refuse** with M4108 (class `transient`) |
| Gateway requests | L6 | network → handlers | 10 req/s per path token bucket, body ≤ 1 MiB | **refuse** 429 / 413 [RI ch. 5, Shed Load] |
| Event coalescing | L6 | bus → clients | `RunUpdated` ≤ 10/s per run, `NodeOutput` ≥ 16 ms apart, `Usage` 1/s | **coalesce** |
| Items in CEL `items` | L2/L4 | item files → evaluator | `limits.items_in_memory` 64 MiB or 100k items | **refuse**: M4104 suggesting `each` |
| List APIs (`ListRuns`, `GetItems`, `GetLogs`) | L6 | index/files → client | `limit` ≤ 1000 per page, cursor | paginate [RI ch. 4, Unbounded Result Sets] |

## 5. Bulkheads

Failure in one compartment must not sink the others [RI ch. 5, Bulkheads]:

| Bulkhead | Isolates |
|---|---|
| Slot pools (`runs`, `nodes`, `agents`, `agent:<id>` new, `runner:<id>`, `run-agents:<run>`, `node:<run>/<node>`, `parts:<id>`) | one agent, runner, run or node can't take every slot ([engine.md](engine.md) §5). `agent:<id>` (registry `max_concurrent`, unset = unlimited) is added so one provider's limit doesn't block other agents |
| Process group / Job Object per node-run | a runaway step and its children are killed as a unit |
| One process per per-call part; one pool per long-lived part id | a crashing part takes down only its contexts |
| One run actor per run, one redactor per run | a corrupt log or a stuck run affects that run only |
| One Lua state per plugin, serialized | a slow Lua part blocks only its own plugin |
| Per-connection queue and writer | a slow client affects only itself |
| Workspace per node-run (per item in `each`) | parallel agents don't edit the same tree |
| Gateway on its own listener and goroutines | webhook floods don't touch the local socket |

## 6. Circuit breakers

Per **agent id** and per **HTTP host** (the `http` part, MCP over HTTP, the Vault backend, the drafter's Ollama) [RI ch. 5, Circuit Breaker] [BM ch. 12]. Lives in `engine/sched` (agents) and a small shared helper in `parthost` (hosts). Decision D36.

| State | Enters when | Behavior |
|---|---|---|
| closed | start; a probe succeeded | normal |
| open | 5 consecutive failures of class `transient`, `rate_limited` or `unreachable` within 2 min, or one `rate_limited` with `retry_after` ≥ 60 s | **agents:** the scheduler stops granting `agent:<id>` slots; node-runs stay `queued` with board text `claude rate limited · resumes 12:41` (no attempt is spent). **HTTP hosts:** calls fail at once with class `transient` and `retry_after` = remaining cooldown, so node retry policy applies |
| half-open | cooldown over: max(`retry_after`, 30 s), doubling per re-open up to 10 m | one probe node-run (or request) allowed; success → closed, failure → open |

Classes `user`, `auth` and `agent` don't trip the breaker: retrying them won't help, and they already fail fast (an expired login fails every node with M6202 and its help line). Breaker state is daemon memory, logged and exposed as `breaker_state{key}` and in `mill doctor`; it doesn't go in run logs because holding a node-run is a scheduling delay, not a decision that changes results. `max_wall` and node timeouts still apply while held, so a long outage ends runs with a budget or timeout error instead of waiting forever.

This departs from the book's fail-fast breaker for agents on purpose (see [references.md](references.md), RI): failing would burn retry attempts during a provider outage that holding rides out.

## 7. Other stability patterns in mill

| Pattern [RI ch. 5] | Where in mill |
|---|---|
| Fail Fast | compile, lint, trust, secrets (M9202) and budget pre-checks run before any node-run starts; unknown agent CLI fails before spawn (M6101) |
| Let It Crash | daemon stop is a crash (ADR-042); a panic in the run actor or scheduler exits the daemon; recovery replays ([operations.md](operations.md) §3) |
| Handshaking | `Hello`/`Snapshot` with versions and features; part `Describe` and health pings |
| Governor | budgets: tokens, cost, active wall time, nodes, agents ([engine.md](engine.md) §6); `max_queued_runs` |
| Steady State | §8 |
| Test Harnesses | echo agent, `mill-test-part` (crash on Open, ignore Cancel, never Pong, huge items), fake CLIs, crash-point exploration ([testing.md](testing.md)) [RI ch. 17] |
| Decoupling Middleware | not used: the bus is in-process; sinks are the async edge |

## 8. Steady state

Everything that grows has a bound and something that sweeps it [RI ch. 5, Steady State] [SRE ch. 26]. Policies and mechanisms are in [data-lifecycle.md](data-lifecycle.md); the bounds:

| Grows | Bound |
|---|---|
| Run dirs | `retention` 30 d by `finished_at` |
| Cache | `cache.ttl` 30 d since last hit, `cache.max_bytes` 5 GiB LRU |
| Artifacts | mark-and-sweep from remaining runs and cache, 1 h grace |
| Node-run logs | `logs.max_bytes_per_node_run` 256 MiB (first 16 MiB + rolling last 64 MiB) |
| `milld.log` | 10 MiB × 5 files |
| Trigger seen keys | `dedupe_window` and 10k keys per trigger |
| `tmp/`, `scratch/` | swept on start (> 24 h) |
| Workspaces | per-node policy + `keep_workspaces`; orphan sweep daily |
| Goroutines | proportional to live node-runs and connections; `goroutines` gauge in `Stats` |
| Memory | items stream through files; CEL `items` cap; codec line cap 16 MiB |

## 9. Antipattern checklist

Reviewed for every milestone that adds an integration point ([milestones.md](milestones.md)) [RI ch. 4]:

| Antipattern | mill's answer |
|---|---|
| Integration Points | §2 timeouts, §5 bulkheads, §6 breakers, classes per failure |
| Chain Reactions | per-part processes; restart backoff; one part's crash fails only its contexts |
| Cascading Failures | bounded queues (§4); slow clients dropped; sinks drop |
| Blocked Threads | actor rule ([concurrency.md](concurrency.md) §2); every wait has a deadline; heartbeats find stalls |
| Self-Denial Attacks | trigger dedupe and concurrency policies; `parallel N` caps |
| Unbalanced Capacities | `agent:<id>` pools sized to the provider; `max_concurrent_agents` |
| Dogpile | cron jitter; `catch_up: last`; full-jitter backoff |
| Force Multiplier | prune and GC refuse live runs and parents of live runs; GC grace; `--dry-run` for prune |
| Slow Responses | timeouts everywhere; coalesced events |
| Unbounded Result Sets | paginated list intents; items in files; `items_in_memory` cap |
| Users, Scaling Effects | single user, local: n/a beyond gateway limits |

## 10. Tests

| Test | Guards |
|---|---|
| `TestTimeouts_Table`: each row of §2 with a fake that never answers, under `synctest` | §2 |
| `TestQueue_<name>_Overflow` per row of §4 | §4 |
| `TestBreaker_HoldsQueue_NoAttemptSpent` with `@echo:rate-limit` | §6, D36 |
| `TestBulkhead_AgentPoolIsolation`: agent A held, agent B still granted | §5 |
| Chaos: `mill-test-part` crash/ignore-cancel/never-pong matrix | §7 |
| Soak (M11): a week of dogfood pipelines with disk, goroutine and memory gauges flat | §8 |

## 11. Sources

| Decision | Follows |
|---|---|
| Timeouts on every integration point | [RI ch. 5, Timeouts] [RI ch. 4, Integration Points] [BM ch. 12] |
| Retry only idempotent or marked-retryable work; full jitter | [BM ch. 12] [SRE ch. 24] [RI ch. 4, Dogpile] |
| Bounded queues with explicit overflow | [RI ch. 5, Create Back Pressure] [SRE ch. 22] [CIG ch. 4, Queuing] |
| Slot pools and process groups as bulkheads | [RI ch. 5, Bulkheads] |
| Breakers per agent and host, holding instead of failing | [RI ch. 5, Circuit Breaker] (departure noted) |
| Shed load at the gateway, drop slow clients | [RI ch. 5, Shed Load] [SRE ch. 21] |
| Budgets as governor | [RI ch. 5, Governor] |
| Crash, don't guess | [RI ch. 5, Let It Crash] [BSRS ch. 9] |
| Steady state on disk | [RI ch. 5, Steady State] [SRE ch. 26] |
| Part health by heartbeat | [DBI ch. 9] [CIG ch. 5] |
