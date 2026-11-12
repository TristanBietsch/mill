# Engine

How a pipeline file becomes an execution graph, and how the engine runs it: items, modes, scheduling, budgets, retries, cache, cancellation, recovery, sub-pipelines and loops. Refines spec §6, §7.4, §8, §15.

Layers ([layers.md](layers.md)): compile is L2 (`pipeline`, `expr`, `schema`); the engine, scheduler, budgets, retries and cache are L4; the reducer is L1 (`core/fold`, D31). Citations use the keys in [references.md](references.md); the sources table is §13.

## 1. Compile: YAML → Spec → IR

`pipeline.Compile(src Source, reg PartRegistry, opts) (*ir.Graph, Diagnostics)`. Pure function of its inputs; results cached by `(bundle hash, registry hash)` with `singleflight`.

| Step | Package | Output | Errors |
|---|---|---|---|
| 1. Load | `pipeline` | `Source{Path, Bytes, Refs map[path][]byte}`: the file plus every `@file` it references (prompts, templates, Lua scripts) | M2101 parse, M2102 missing `@file` |
| 2. Hash | `pipeline` | `BundleHash = sha256("mill-bundle-v1\n" + path + "\n" + sha256(bytes) + sorted(ref path + sha256))`. This is the version recorded on the run (invariant 10). Editing a prompt file is a new version. | |
| 3. Parse | `pipeline` (yaml/v4 Node API) | `Spec` with a `Pos{File, Line, Col}` on every field; unknown keys → warning with did-you-mean | M2101, M3001 unknown key |
| 4. Templates | `expr` | Every string scanned for `${{ … }}`. The scanner is CEL-aware: it tracks string literals so `"}}"` inside a CEL string doesn't close the expression. Each expression is parsed and type-checked with cel-go, and its references are collected by walking the checked AST (`nodes.X…`, `inputs.Y`, `secrets.Z`, `item`, `items`, `event`, `env`, `run`). | M3012 unknown node, M3013 unknown input, M3101 type error, M9201 undeclared secret |
| 5. Resolve parts | `pipeline.Resolver` port, implemented by `parthost` (L5) | `uses` → `PartRef{ID, Version, Capability, Manifest}` via resolution order and lockfile | M5101 not found, M5102 version |
| 6. Validate `with` | `schema` | Literal values validated against the part's config schema now; expression-valued fields validated against the schema's type when CEL can infer it, otherwise at runtime after evaluation | M3201 config invalid |
| 7. Infer edges | `pipeline` | data edges + order edges (§2) | M3020 cycle, M3021 ambiguous input |
| 8. Lint | `pipeline` | §7.7 checks + policy (destructive, runners), secrets in argv, unreachable nodes | M32xx, M9xxx |
| 9. Build IR | `pipeline/ir` | immutable `Graph` | |

```go
package ir

type Graph struct {
	Pipeline   string
	BundleHash string
	Nodes      []Node          // index = stable node number
	ByID       map[string]int
	Topo       []int           // topological order (ties broken by file order)
	Inputs     []InputDecl
	Outputs    []OutputExpr
	Triggers   []TriggerDecl
	Secrets    []string
	Budget     Budget
	Policy     Policy
	Defaults   Defaults
}

type Node struct {
	ID          string
	Index       int
	Part        PartRef
	With        Template        // tree of literals + compiled CEL programs
	Mode        Mode            // batch | each | stream
	Concurrency int
	When        *Program
	In          []PortRef       // data inputs, in order: {Node, Port}
	After       []int           // order-only deps (needs + implicit refs that aren't data inputs)
	Ports       []string        // declared output ports ("out", "error", "true", "false", "else", "case.x")
	Runner      string
	Workspace   WorkspaceMode   // inherit | git | temp | none
	Retry       RetryPolicy
	Timeout     time.Duration
	Cache       *CacheSpec
	OnError     OnError         // fail | continue | route
	Output      *schema.Schema
	IO          IOMode          // envelope | plain | text
	Flags       partproto.Flags // snapshot from manifest (idempotent, destructive, …)
	IsAgent     bool            // counts toward agent slots and max_agents
	Pos         Pos
}
```

Static expressions (only `inputs`, `env`, `run`, `event`) are evaluated once at run start; dynamic ones (`nodes`, `item`, `items`) when the node-run opens. `now` is fixed to the node-run's recorded start time so replay evaluates identically.

## 2. Edge inference

The spec (§7.4) says a reference creates an edge `a.out → b.in`. Taken literally, `pr` in §7.5 references `review` and `plan`, so its input stream would be review's items **and** plan's items concatenated, which isn't what anyone means. The rule we implement separates the two things a reference can mean:

1. **Every reference** to `nodes.X` creates an **order edge** X → this node (this node waits for X to be terminal and can read X's outputs through `nodes.X`).
2. **The data input** (`in`, the item stream that `item`/`items` and the part's stdin see) comes from:
   - `in:` if given (a node, `node.port`, a list of them, or an expression);
   - else explicit `edges:` into this node;
   - else the **primary upstream**: among the nodes this node references, the one that every other referenced node is upstream of (the deepest one). `pr` references `review` and `plan`; `plan` is upstream of `review`, so `review` is primary.
   - If referenced nodes have no single deepest one (two independent branches), lint fails with **M3021 ambiguous input** and suggests `in: [a, b]` (concat) or a `merge` node.
   - A node that references nothing and has no `in` is a source: it gets the run's input items (from `--stdin` / trigger) if it's the first source in file order, otherwise no items.
3. `needs:` adds order edges only.
4. Ports: `in: scope` means `scope.out`; `in: scope.else` names a port. Branching parts (`if`, `switch`, `assert` with `else: route`, `on_error: route`) emit on named ports; a node fed only by a port that received zero items and whose producer finished is `skipped` (§4).
5. Cycles: any cycle in data or order edges is M3020. Loops are expressed with `agent.loop` (single node), `pipeline` (a sub-pipeline), or `loop` with a `with.body` list of node ids (§11); the compiler treats a loop's body as one unit, so the graph is always a DAG.

`mill show --graph` prints both edge kinds so the inference is never invisible.

## 3. Items and streams

**Storage first.** Every output item is appended to `runs/<id>/items/<node-run>/<port>.jsonl` (one `core.Item` envelope per line, redacted) before downstream can see it. The run actor then writes an `ItemsCommitted{nr, port, through_index, bytes}` event. Downstream readers read the file up to the committed index. Consequences:
- Memory stays flat: no node's output is held in RAM.
- `mill items`, `rerun --from`, replay and cache all read the same files.
- A crash never exposes an item downstream that isn't on disk.

**Ordering.** Items keep their order along an edge. For `each` nodes, the node's output on a port is the concatenation of per-item outputs **in input order**, not completion order (deterministic, matches what a reader expects from a filter). Lineage: every output item's `meta.parents` holds the ids of the input items it came from (for per-call parts the host sets it to all inputs of that call; `each` gives exactly one).

**Size limits.** One item line ≤ 16 MiB (codec cap). Larger payloads must be artifacts; a part emitting a bigger item fails with M4103 and a hint. In batch mode, materializing `items` for CEL is capped (`limits.items_in_memory`, default 64 MiB or 100k items) → M4104 suggesting `each` or a file.

**IO modes for exec-style parts (sh, exec, check):**

| `io` | stdin gets | stdout parsed as |
|---|---|---|
| `envelope` | one `core.Item` JSON per line | `core.Item` per line (port from `meta.port`, default `out`) |
| `plain` (default for sh/exec) | `data` JSON per line | one JSON value per line → item `data`; non-JSON line → M4105 unless `lenient: true` (then `{"text": line}`) |
| `text` | `data` rendered as text per line (strings raw, other JSON compact) | each line → `{"text": "<line>"}`… or the raw string if the node sets `text_as: string` |

Files, workspace and lineage are carried forward by the host for `plain` and `text`: outputs inherit the union of inputs' `files` and the first input's `ws` (§6.1).

**Stream mode** (M7, not v0): producer and consumer node-runs run at the same time. The consumer reads the producer's items file as it grows (tail with a condition variable signaled on `ItemsCommitted`). Backpressure: the host stops reading the producer's stdout when the consumer lags more than `stream.window` items (default 256) behind; OS pipe buffers then block the producer. A stream node is `working` until its input ends and it exits.

## 4. Node-run state machine

```
            ┌─────────── when=false / all inputs skipped ───────────► skipped
pending ────┼─ cache hit ──────────────────────────────────► cached
            └─ inputs ready ─► queued ─ grant ─► working ─┬─► done
                                                          ├─► failed ──(retry policy matches)──► [new node-run: retrying ─ backoff ─► queued]
                                                          ├─► cancelled
                                                          └─► crashed (recovery only)
```

- `retrying` belongs to the **next** attempt. Attempt N ends `failed` (terminal, invariant 6). The actor immediately creates attempt N+1 in `retrying` with `retry_at`; when the timer fires it moves to `queued`. This keeps every terminal state final and still shows "retrying" on the board.
- A cache hit goes `pending → cached` without taking any slot: the lookup happens in the run actor before the node-run is enqueued.
- `skipped` reasons are recorded: `when_false`, `branch_not_taken`, `upstream_skipped`, `upstream_failed` (with `on_error: fail` upstream, downstream of a failure is marked `skipped` with that reason when the run ends, so the run popup can show why).

Readiness (evaluated by the run actor after every terminal transition, O(out-degree)):
1. All order-edge predecessors terminal.
2. Data predecessors terminal (batch/each). For stream, predecessors `working` or terminal.
3. If every data predecessor is `skipped`, or every input port it reads received zero items from a branching part → `skipped` (except `merge` with `any`, spec §15.2).
4. If any predecessor `failed` with `on_error: fail` → not ready; it will be `skipped(upstream_failed)` at run end.
5. Evaluate `when` (dynamic, can read `nodes.*`). False → `skipped(when_false)`.
6. Cache check (§8). Hit → `cached`.
7. Mode expansion: batch → one node-run `<node>@1`. each → one node-run per input item `<node>[i]@1`, enqueued in input order, capped by `concurrency` (a slot pool named `node:<run>/<node>`).
8. Enqueue to scheduler.

## 5. Scheduler and slots

One goroutine, one ready queue across all runs.

**Slot pools.**

| Pool | Size from | Needed by |
|---|---|---|
| `runs` | `max_concurrent_runs` (8) | taken when a run moves `queued → working`; held until the run is terminal |
| `nodes` | `max_concurrent_nodes` (16) | every node-run except control/data parts that declare `flags.light` (map, filter, if, assert, …, run inline in the actor's worker pool and don't count) |
| `agents` | `max_concurrent_agents` (4) | node-runs whose part `IsAgent` |
| `agent:<id>` (new) | registry `max_concurrent` for that agent (unset = unlimited) | agent node-runs of that agent id; held while its circuit breaker is open ([resilience.md](resilience.md) §6, D36) |
| `runner:<id>` | `runners.<id>.max_concurrent` (unset = unlimited) | node-runs on that runner |
| `run-agents:<run>` | pipeline `budget.max_agents` | agent node-runs of that run |
| `node:<run>/<node>` | node `concurrency` (default 4 for `each`) | `each` node-runs |
| `parts:<id>` | manifest `max_concurrent` (optional) | node-runs of that part |

**Grant algorithm.** On every change (enqueue, release, resize):
1. Sort the queue by `(run priority desc, run created asc, node topo index asc, enqueued at asc)`. Run priority defaults to 0; `mill run --priority`, trigger config and `policy.priority` can set it (the spec mentions priority but never defines where it's set).
2. Walk the queue. For each request, if **all** its pools have capacity, take them atomically (single goroutine, no partial holds) and send the `Grant`.
3. If not, skip it, but **reserve**: a request that has been skipped for longer than `sched.starvation_after` (30 s) blocks later requests that need any of the same pools. This prevents a stream of small node-runs from starving a wide agent fan-out.
4. Grants are delivered on a buffered channel per run actor; a run actor that has since cancelled returns the grant immediately.
5. A pool whose breaker is open grants nothing until its half-open probe; requests needing it stay queued and show why on the board (D36).

Slot pools are bulkheads [RI ch. 5, Bulkheads]; the single scheduler goroutine is confinement [CIG ch. 4]. `Engine.Start` refuses new runs past `max_queued_runs` (1000) with M4108 [RI ch. 5, Create Back Pressure].

**Invariant 7.** Every `Grant` carries an id. The actor records `GrantTaken{id}` in memory and must call `Release(id)` when the node-run reaches a terminal state. `sched.AssertDrained()` runs in every engine test's cleanup.

## 6. Budgets

Ledger per run, folded from events: `tokens_in`, `tokens_out`, `cost_micro_usd`, `nodes_started`, `wall_active`, `agents_live`. Usage arrives as `Usage` messages (deltas) from parts and from agent output parsing. Cost = part-supplied cost, else `tokens × price` from the agent registry; unknown price ⇒ cost unknown for that usage, recorded as such.

| Limit | Check point | On hit |
|---|---|---|
| `max_tokens`, `max_cost_usd` | after every Usage fold; before granting an agent node-run | Append `BudgetExceeded`; cancel the run with reason `budget` (teardown of live node-runs with `Cancel` then kill after grace). Run ends `cancelled`, error M4101, CLI exit 9. |
| warning | 80 % of any limit | `BudgetWarning` event once per limit; `budget_warning` Lua hook |
| `max_wall` | timer in the actor | cancel, reason `budget`, M4102 |
| `max_nodes` | before creating a node-run (counts attempts and each-expansion) | cancel the run, reason `budget`, M4107 |
| `max_agents` | slot pool `run-agents:<run>` | waits, doesn't fail |
| `max_cost_usd` with unpriced models | lint | warning M3301: "claude/opus has no price; max_cost_usd can't be enforced for it" |

Overshoot is bounded by what's in flight: an agent call that's already running finishes its current turn before `Cancel` lands. `Open.budget` passes remaining tokens so parts that can cap themselves (an agent with `--max-tokens`) do.

`max_wall` counts time the run is live **while an engine is running it** (`wall_active`), not calendar time: a laptop that sleeps overnight doesn't cancel every run on wake. Decision D16 (default accepted, spec §15.5).

## 7. Retries and backoff

```yaml
retry: { max: 3, backoff: "exp(2s, x2, max 5m)", on: [exit:1, class:transient, class:rate_limited, code:M6201] }
```

- Grammar: `exp(<initial>, x<factor>, max <cap>)`, `linear(<initial>, +<step>, max <cap>)`, `fixed(<d>)`. Parsed in `engine/retry`, fuzzed.
- Jitter: full jitter on the computed delay (`rand(0.5d, 1.0d)`), seeded per node-run so tests with a fake clock are deterministic.
- `rate_limited` with `retry_after`: delay = max(backoff, retry_after).
- Matchers: `exit:N`, `class:C`, `code:Mxxxx` or plugin code. A `Fail` matches if any matcher matches.
- **Default policy** (when the node and `defaults.retry` set nothing): `{max: 2, on: [class:transient, class:rate_limited], backoff: "exp(2s, x2, max 5m)"}`, applied only if the part is `idempotent` **or** the failure says `retryable: true`. Non-idempotent parts that fail without `retryable` are never retried by default (they may have had side effects).
- Workspace on retry: for node-runs with a git workspace, the worker snapshots the worktree before the attempt (`git stash create` + recorded tree sha, or a commit on a hidden ref `refs/mill/snap/<nr>`) and resets to it before the next attempt, unless `retry.keep_workspace: true`. A retried agent starts clean instead of on top of a half-finished edit. Decision D11 (default accepted, spec §15.3).
- `max` counts attempts after the first. `mill run`'s `RetryNode` intent creates a manual attempt that doesn't count against `max`.

## 8. Cache

Key = `sha256(canonical_json({`
`  "v": 1,`
`  "part": "<id>@<version>", "capability": "...",`
`  "with": <resolved with, secrets replaced by "secret:<NAME>">,`
`  "inputs": [ { "data": <data>, "files": [<sha256>…] } … ],   // meta excluded: run ids and timestamps would make every key unique`
`  "ws": <tree sha of the workspace at node start, or null>,     // not the path, which differs per run`
`  "file_inputs": [<path>:<sha256>…],                           // declared via cache.files globs`
`  "key": <value of cache key expression, if given>`
`}))`

Canonical JSON = RFC 8785 (JCS): sorted keys, no insignificant whitespace, ECMAScript number formatting. Implemented in `engine/cache/jcs.go` (≈150 lines, fuzzed against `encoding/json` round-trips).

- Lookup happens in the run actor before enqueue. Hit: copy (hardlink) the cached item files into the node-run's items dir, register artifact refs, append `NodeRunFinished{state: cached, cache_key, source_run}`.
- Store: after `done`, if caching is on, write `cache/<k[:2]>/<k>/entry.json` + items files (atomic dir rename). Artifacts are referenced by sha; the CAS GC treats cache entries as roots.
- Never cached: agent parts (unless the node sets `cache: true` explicitly), destructive parts (lint error M3305 if `cache` set), parts declaring `flags.nondeterministic`.
- `mill cache ls | clear [--pipeline p] [--older 7d]`.

## 9. Cancellation and teardown

One function, `teardown(nr, reason)`, used by cancel, delete, budget, `max_wall`, timeout, fail-fast and shutdown (invariant 8):

1. Actor marks the node-run `cancelling` in memory (not a state; nothing is written that a client would read as `failed`).
2. Worker sends `Cancel` to the part session.
3. For process-backed node-runs, worker sends `Interrupt` (SIGTERM to the group; `CTRL_BREAK_EVENT` on Windows).
4. Wait `cancel_grace` (default 10 s, setting; per-part override `flags.grace`).
5. `Kill` (SIGKILL to the group; `TerminateJobObject`). Idempotent.
6. Drain stdout/stderr for at most 2 s or 1 MiB, whichever first; then close pipes.
7. `Close` the part session.
8. Actor appends `NodeRunFinished{state: cancelled, reason}` (timeouts use `failed` with class `timeout` instead, M4201, because a timeout is a failure of that node, not a cancel of the run).
9. Release grants. Workspace cleanup follows policy (on cancel: `keep_workspaces` decides; default `on_failure` keeps it).

Exit codes of killed processes are ignored once `cancelling` is set, which is how "no transient failed state" holds.

`fail_fast` (policy, default `true`): when a node fails with `on_error: fail`, in-flight siblings are torn down with reason `fail_fast` and nothing new starts. With `false`, in-flight ones finish and nothing new starts on paths that can't reach an output.

**Run end.** The run is terminal when no node-run is live and no node can become ready. State: `done` if no node-run ended `failed`/`crashed` with `on_error: fail`; `cancelled` if a cancel was requested; `crashed` if any node-run is `crashed` (M4301); else `failed`. Outputs are evaluated for `done` runs and, best-effort, for others (missing nodes → `null`).

## 10. Crash recovery replay

On startup (`Engine.Recover`), for each run in `live/` whose `run.lock` is free:

1. **Replay.** Read `events.jsonl` sequentially through the same reducer used live (`fold.Apply(state, ev)` in `core/fold`, L1; the index rebuild uses the same function, D31). A torn last line (no trailing newline or invalid JSON at EOF) is truncated after copying it to `events.jsonl.torn-<ts>`; anything invalid before the last line is corruption → run `crashed` with M4302 and `mill doctor` reports it.
2. **Orphans.** For each node-run whose last state is `working`, `NodeRunStarted` recorded `{pid, pgid, start_time}` (and container name). If a process with that pgid exists and its start time matches (`/proc/<pid>/stat` on Linux, `sysctl kern.proc.pid` on macOS; Windows needs nothing: Job Objects with `KILL_ON_JOB_CLOSE` died with milld), kill the group. Containers: `docker rm -f` by label. This is done **before** any re-run so an idempotent restart never runs twice in parallel.
3. **Decide** per spec §15.7: `queued`/`pending`/`retrying` → re-enqueue (retry timers re-armed from `retry_at`); `working` + `resumable` → open the part with `Resume(last Checkpoint)`; `working` + `idempotent` → new attempt with reason `daemon_restart`; otherwise `crashed`, M4301, transcript and partial items kept. Terminal agents (PTY) are always `crashed`.
4. Item files may have uncommitted tails (written but no `ItemsCommitted` event): truncate them to the last committed byte offset recorded in the event.
5. Append `RunRecovered{decisions}` so the audit trail shows what happened.
6. Runs with a `crashed` node-run and `on_error: fail` end `crashed`; others continue.

Determinism requirement for this to work: the reducer is pure (no clock reads, no I/O; everything time-dependent comes from event timestamps), and every decision that depends on the outside world (cache hit, `when` result, retry delay chosen, slot grant) is recorded as an event before it takes effect.

## 11. Sub-pipelines and loops

**`pipeline` part.** `uses: pipeline, with: { name: summarize, inputs: {...} }`.
- Compiles the child at its current bundle hash (recorded in the parent's event), starts a **child run** with `parent: {run, node_run}`. Child runs are real runs (own number, own dir, listed under the child pipeline) so `mill logs` and the board work unchanged.
- Budget: child limits = min(own budget, parent remaining). Child usage is added to the parent's ledger as it arrives (`Usage` forwarded).
- Cancel and teardown propagate parent → child. Child failure → the `pipeline` node-run fails with the child's error as cause.
- Child `outputs` become the node's items (one item per child run; in `each` mode one child run per item).
- Recursion: compile-time check for direct cycles by name; runtime depth limit 8 (M4106).
- Crash recovery: the child run is recovered on its own; the parent's `pipeline` node-run is `resumable` by construction (its checkpoint is the child run id), so it re-attaches.

**`agent.loop`.** One node-run. Internally: run agent → run `until` check in the same workspace through the same runner → if it fails, append the failure tail to the next prompt → repeat up to `max_iters`. Each iteration is recorded as `LoopIteration{n, check_exit, usage}` events so the run popup can show `loop 2/3`. Output: last agent result plus `iterations`, `passed`. Budget is checked between iterations.

**`loop` part (generic, M8).** Shape decided 2026-11-11 (editor decision 4, spec §10.3): `with: { body: [patch, test], until: ${{ … }}, max_iters: N }`, where `body` lists node ids in the same pipeline. The first body node reads `in: <loop>.body`; downstream reads `<loop>` (port `out`, the last try when `until` held) or `<loop>.exhausted` (the last try when `max_iters` ran out). `loop.iter`, `loop.max` and `loop.prev` are available in body expressions. The compiler contracts the body into the loop node for the outer DAG; each iteration creates a fresh set of node-runs for the body nodes (`<node>@iter<n>`), recorded with `LoopIteration` events, and `until` is evaluated after the last body node finishes. Bounded by `max_iters` and `budget.max_nodes`. (This replaces the earlier sub-pipeline-body proposal in S30; a sub-pipeline can still be looped by putting a `pipeline` node in the body.)

**`agent.fanout`** expands to N node-runs internally (one per agent/model/seed) with N workspaces; it's a part, not a graph rewrite, so the graph stays as written. Its output is N items, one per candidate, each carrying its `ws`.

## 12. Defaults worth writing down

| Setting | Default | Why |
|---|---|---|
| node `timeout` | 30 m for non-agent nodes, 2 h for agent nodes | runaway protection without a budget |
| `cancel_grace` | 10 s | agents need a moment to flush |
| `each` concurrency | 4 | |
| `fail_fast` | true | saves tokens |
| retry | §7 default policy | |
| `keep_workspaces` | `on_failure` | spec |
| `stream.window` | 256 items | |
| run priority | 0 | |

## 13. Sources

| Decision | Section | Follows |
|---|---|---|
| Compile is a pure, cached function of the bundle | §1 | [DDIA ch. 12] (derived data from immutable inputs) |
| Bundle hash as the run's version | §1 | [DDIA ch. 4] |
| Steps as filters over item streams; Unix pipeline model | §2–3 | [FSA ch. 11] [EIP, Pipes and Filters] [TAOUP ch. 11] [DDIA ch. 10] |
| References order, primary upstream carries data (ADR-028) | §2 | [EIP, Pipes and Filters] |
| Branch ports, error port | §2, §9 | [EIP, Content-Based Router] [EIP, Dead Letter Channel] |
| Items committed to files before downstream sees them | §3 | [EIP, Guaranteed Delivery] [DDIA ch. 11] |
| `each` mode and `merge` | §3–4 | [EIP, Splitter] [EIP, Aggregator] |
| Large payloads as artifacts | §3 | [EIP, Claim Check] |
| Stream mode backpressure by window | §3 | [RI ch. 5, Create Back Pressure] [DDIA ch. 11] |
| Terminal states final; `retrying` is the next attempt | §4 | [DDD ch. 6] (aggregate invariants) |
| One scheduler, slot pools as bulkheads, aging against starvation | §5 | [CIG ch. 4] [RI ch. 5, Bulkheads] |
| Budgets stop runaway work | §6 | [RI ch. 5, Governor] |
| Active wall time, not calendar time | §6 | [DDIA ch. 8] |
| Retries: idempotent only by default, full jitter, `retry_after` | §7 | [BM ch. 12] [RI ch. 4, Dogpile] [SRE ch. 24] |
| Canonical JSON cache keys over data, not metadata | §8 | [DDIA ch. 4] |
| One teardown path, exit codes ignored once cancelling | §9 | [BSRS ch. 6] [RI ch. 5, Timeouts] |
| Fail fast on the first failure by default | §9 | [RI ch. 5, Fail Fast] |
| Replay through the same pure reducer; record every outside decision | §10 | [DDIA ch. 11] [DBI ch. 5] |
| Kill orphans before re-running; non-idempotent work `crashed` | §10 | [RI ch. 5, Let It Crash] [BSRS ch. 9] |
| Child runs as real runs; budgets propagate | §11 | [RI ch. 5, Governor] (budgets propagate to children) |
