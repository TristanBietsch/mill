## 15. Execution semantics

### 15.1 Compile

On `StartRun`, `milld` loads the pipeline at a fixed content hash, resolves parts from the lockfile, evaluates static expressions, infers edges, validates, and builds the execution graph. The hash is recorded on the run.

### 15.2 Scheduling

- A node-run becomes **ready** when all its required inputs are complete (or, for `stream` mode, when upstream has started) and its `when` is true.
- Ready node-runs enter the queue in order: run priority, then pipeline order, then age.
- A node-run **starts** only when it can acquire every slot it needs: global (`max_concurrent_nodes`), per-runner, per-agent (`max_agents`), and per-run budget.
- `skipped` propagates: a node whose required inputs are all skipped is skipped, unless it's a `merge` with `any`.

### 15.3 Retries

`retry: { max, backoff: "exp(2s, x2, max 5m)", on: [exit:1, class:transient, class:rate_limited] }`. A part's `Fail` message says whether the error is retryable. Non-idempotent parts are never retried automatically after a crash (§15.7), only after clean failures they mark retryable.

### 15.4 Caching

With `cache` on, a node-run's key is the hash of: part id and version, resolved config, input items, and declared file inputs. A hit reuses stored output items and artifacts and marks the node-run `cached`. `mill cache ls | clear` manages it. Agent nodes are not cached by default.

### 15.5 Budgets

| Limit | Behavior when hit |
|---|---|
| `max_tokens`, `max_cost_usd` | No new agent node-runs start. Running ones are asked to finish (`Cancel` after the grace period). The run ends `cancelled` with reason `budget`. |
| `max_wall` | The run is cancelled. |
| `max_nodes` | Guards against runaway loops and fan-out. |
| `max_agents` | Concurrent agent cap within the run. |

`mill run --budget-tokens 500k` overrides from the CLI. Usage comes from `Usage` messages and agent `usage` detection.

### 15.6 Cancellation and teardown

`mill cancel <run>`: no new node-runs start. Every live node-run gets `Cancel`, then after a grace period its process group is killed, output is drained for a bounded time, and slots are released. Workspaces follow their cleanup policy. Deleting a run uses the same teardown and then removes its directory.

### 15.7 Daemon restarts and crash recovery

On startup, `milld` replays each unfinished run's event log:

| Node-run was | Part declares | Result |
|---|---|---|
| Terminal | n/a | Unchanged |
| `queued` / `pending` | n/a | Re-queued |
| `working` | `resumable` | `Resume` with the last checkpoint |
| `working` | `idempotent` | Re-run from the start |
| `working` | neither | `crashed`. The run fails with `M4301` and keeps every finished node's output, so `mill rerun <run> --from <node>` continues from there. |
| `working` (terminal agent) | n/a | `crashed` (the PTY is gone). Its transcript is kept. |

### 15.8 Errors

- A failed node with `on_error: fail` fails the run once no other branch can still produce outputs (in-flight siblings finish or are cancelled per `policy.fail_fast`).
- `on_error: continue` emits nothing on `out` and lets downstream decide (their inputs may be empty).
- `on_error: route` sends an error item (`class`, `message`, `node`, `exit_code`, `stderr_tail`) to the node's `error` port.

### 15.9 Replay and re-runs

- `mill rerun <run>`: a new run with the same inputs and pipeline version.
- `mill rerun <run> --from <node>`: a new run that reuses recorded outputs for every node upstream of `<node>`.
- `mill replay <run> --node <node>`: run one node again in isolation against its recorded inputs. Great for tuning a prompt or a transform.


---

[← spec index](index.md)
