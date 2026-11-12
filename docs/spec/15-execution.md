## 15. Execution semantics

### 15.1 Compile

On `StartRun`, `milld` loads the pipeline at a fixed content hash, resolves parts from the lockfile, evaluates static expressions, infers edges, validates, and builds the execution graph. The hash is recorded on the run.

### 15.2 Scheduling

- A node-run becomes **ready** when all its required inputs are complete (or, for `stream` mode, when upstream has started) and its `when` is true.
- Ready node-runs enter the queue in order: run priority, then pipeline order, then age. A run's priority is an integer, default 0, set by `mill run --priority`, `StartRun`, the trigger's config, or `policy.priority`.
- A node-run **starts** only when it can acquire every slot it needs: global (`max_concurrent_nodes`), per-runner, per-agent (`max_agents`), and per-run budget.
- `skipped` propagates: a node whose required inputs are all skipped is skipped, unless it's a `merge` with `any`.

### 15.3 Retries

`retry: { max, backoff: "exp(2s, x2, max 5m)", on: [exit:1, class:transient, class:rate_limited] }`. A part's `Fail` message says whether the error is retryable. Non-idempotent parts are never retried automatically after a crash (§15.7), only after clean failures they mark retryable.

- **Default policy.** A node without `retry` (and no `defaults.retry`) gets 2 retries on `transient` and `rate_limited` errors, with exponential backoff (honoring `retry_after` when the provider gives one), but only when the part is idempotent or marked the failure retryable. Side-effecting steps aren't re-run on a guess. `retry: { max: 0 }` turns it off.
- **Each retry is a new attempt** (§4): the failed attempt stays `failed`, and the next one waits out the backoff in `retrying`.
- **Workspaces.** Before a retry, a node's workspace is reset to the snapshot taken before the failed attempt, so every attempt starts clean. `retry: { keep_workspace: true }` lets an agent continue from what the failed attempt left.

### 15.4 Caching

With `cache` on, a node-run's key is the hash of: part id and version, resolved config, the input items' `data` and file hashes (not their `meta`, which holds run ids and timestamps), declared file inputs, and for a node with a workspace, the hash of the workspace tree (never its path). Everything is hashed as canonical JSON. A hit reuses stored output items and artifacts and marks the node-run `cached`. `mill cache ls | clear` manages it. Agent nodes are not cached by default.

### 15.5 Budgets

| Limit | Behavior when hit |
|---|---|
| `max_tokens`, `max_cost_usd` | No new agent node-runs start. Running ones are asked to finish (`Cancel` after the grace period). The run ends `cancelled` with reason `budget` (exit 9, `M4101`), and the board counts it as failed (§4). |
| `max_wall` | The run is cancelled with reason `budget`. It counts *active* time: time while an engine is running the run. A laptop asleep or a stopped daemon doesn't use it up. |
| `max_nodes` | Guards against runaway loops and fan-out. |
| `max_agents` | Concurrent agent cap within the run. |

`mill run --budget-tokens 500k` overrides from the CLI. Usage comes from `Usage` messages and agent `usage` detection. A model with no `price` in the registry (§11.1) records its cost as unknown, not zero; lint warns when a pipeline with `max_cost_usd` uses an unpriced model.

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
| `working` | neither | `crashed`. If the node has `on_error: fail`, the run ends `crashed` with `M4301` and keeps every finished node's output, so `mill rerun <run> --from <node>` continues from there. |
| `working` (terminal agent) | n/a | `crashed` (the PTY is gone). Its transcript is kept. |

**Stopping the daemon is a crash.** `mill daemon stop` and `restart` don't wait for live node-runs and don't cancel them: `milld` stops granting slots and exits within seconds, and on the next start the table above applies. One path ("crash, don't guess") for every way the daemon goes away. `--drain` waits for live node-runs to finish first, up to 10 minutes (`shutdown.drain_timeout`), then stops the same way. Queued work survives either way and resumes after the restart.

### 15.8 Errors

- A failed node with `on_error: fail` fails the run once no other branch can still produce outputs. With `policy.fail_fast` (default `true`) in-flight siblings are cancelled at once; with `false` they finish first.
- `on_error: continue` emits nothing on `out` and lets downstream decide (their inputs may be empty).
- `on_error: route` sends an error item (`class`, `message`, `node`, `exit_code`, `stderr_tail`) to the node's `error` port.

### 15.9 Replay and re-runs

- `mill rerun <run>`: a new run with the same inputs and pipeline version.
- `mill rerun <run> --from <node>`: a new run that reuses recorded outputs for every node upstream of `<node>`.
- `mill replay <run> --node <node>`: run one node again in isolation against its recorded inputs. Great for tuning a prompt or a transform.


---

[← spec index](index.md)
