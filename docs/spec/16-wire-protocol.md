## 16. Daemon wire protocol

Newline-delimited JSON over a local Unix domain socket, on macOS, Linux and Windows alike (Windows 10 1803 and later support AF_UNIX). The socket file is `0600` in a `0700` directory, and the daemon checks the peer's user id and drops other users. The path is in §20. Optionally the same messages go over WebSocket through the gateway.

### 16.1 Envelope

| Field | Required | Meaning |
|---|---|---|
| `v` | yes | Protocol version. `1`. Readers reject other values. |
| `kind` | yes | `Intent` (client → daemon) or `Event` (daemon → client) |
| `type` | yes | Message name |
| `id` | no | Correlation id. An intent with an `id` gets exactly one reply event with the same `id`: `Ack` (or its specific reply) or `Error`. |
| `data` | yes | Payload |

Maximum line length 16 MiB. `[]byte` fields are base64. Unknown fields are ignored; unknown intent types get `Error` `bad_intent`.

### 16.2 Handshake

1. Client sends `Hello` with `client` (program name) and `client_version`.
2. Daemon replies `Snapshot`: active runs, pipeline list, `daemon_version`.
3. Live events follow. Broadcasts are held until `Hello` completes.

### 16.3 Intents

| Intent | Payload | Reply |
|---|---|---|
| `Hello` | `client`, `client_version`, `features` | `Snapshot` |
| `ListPipelines` | `scope` | `Pipelines` |
| `GetPipeline` | `name` | `Pipeline` (source, compiled graph, lint results) |
| `SavePipeline` | `name`, `source`, `layout`, `base_hash`, `force` | `Ack` with new hash; `Error` `conflict` if `base_hash` is stale; `Error` `lint_failed` unless `force` |
| `EditPipeline` | `name`, `base_hash`, `ops` (node add/set/rm/mv, edge add/rm) | `Ack` with new hash and diff. CLI and TUI edits share this one implementation |
| `Lint` | `source` or `name` | `LintResult` |
| `Compile` | `pipe` (pipe expression) | `Pipeline` |
| `DraftPipeline` | `sentence`, optional `base` (the current proposal, for a refine) | `Draft`: the validated proposal (steps, trigger, agent, effort, name, description, a reason per choice), `drafter` (`ollama/<model>`, `agent/<id>` or `keywords`), and a `warning` (`M2212`, `M2213`) when it fell back to keyword rules (§18.8). Health of the drafter is in the `Snapshot` and in `DrafterHealth` events |
| `StartRun` | `pipeline` or `source`, `inputs`, `items`, `input_stream`, `budget`, `labels`, `priority`, `detach`, `dry_run` | `Ack` with `run_id`, `number` |
| `RunInput` | `run_id`, `items` (batches up to 1 MiB) | `Ack`; the client sends the next batch after it. Streams stdin larger than one line when `input_stream` is set |
| `RunInputEnd` | `run_id` | `Ack` |
| `RunNode` | `pipeline` or `source`, `node`, `items` (or `from_run`), `dry_run` | `Ack` with `run_id`; the node's items follow as `ItemEmitted` |
| `Estimate` | `pipeline` or `source` | `Estimate`: per node and total, low and high cost, tokens, time |
| `CancelRun` | `run_id`, `reason` | `Ack` |
| `DeleteRun` | `run_id` | `Ack` |
| `RerunFrom` | `run_id`, `node` (optional) | `Ack` with new `run_id` |
| `RetryNode` | `run_id`, `node_run_id` | `Ack` |
| `Subscribe` | `run_id` (empty for the whole board), optional `node_run_id`, `from_seq`, `replay`, `repaint` (view-only; there is no input intent) | `Ack` after replay |
| `Unsubscribe` | `run_id`, `node_run_id` | `Ack` |
| `Resize` | `node_run_id`, `cols`, `rows` | `Ack` |
| `ListRuns` | filters, `limit`, `cursor` | `Runs` |
| `GetRun` | `run_id` | `Run` |
| `GetItems` | `run_id`, `node`, `port`, `limit`, `cursor` | `Items` |
| `GetLogs` | `run_id`, `node_run_id`, `from_line`, `tail`, `raw` | `Logs` (so `mill logs` needs no file access) |
| `SetTrigger` | `pipeline`, `trigger`, `enabled` | `Ack` |
| `ListTriggers` | `pipeline` | `Triggers`: armed, next fire, last fire, errors |
| `FireTrigger` | `pipeline`, `trigger`, `event` | `Ack` with `run_id` (`mill trigger fire`) |
| `PartCall` | `part`, `method`, `data` (for part-specific UI, e.g. listing Slack channels) | part-defined |
| `ListParts` | `kind` | `Parts`: id, source, version, health (`mill part ls`) |
| `Stats` | `since`, `group_by` | `Stats` (`mill stats`, `:stats`) |
| `CacheList`, `CacheClear` | filters | `Cache` / `Ack` with the count removed |
| `Prune` | `older`, `dry_run` | `Pruned` |
| `Reload` | none | `Ack` once pipelines, parts and registries are reloaded |
| `Shutdown` | `drain` | `Ack` (best effort). Without `drain`, live node-runs are treated as crashed on the next start (§15.7) |

Commands that only touch local files (`config`, `secrets set`, `trust`, `lint` of a file) stay in the CLI and have no intent.

### 16.4 Events

`Snapshot`, `Ack`, `Error`, `RunAdded`, `RunUpdated`, `RunRemoved`, `NodeRunUpdated`, `ItemEmitted` (counts, not payloads, unless subscribed), `RunLog`, `NodeOutput` (PTY bytes for attached clients), `TriggerFired`, `PipelineChanged`, `PartStatus`, `Usage`, `DrafterHealth` (the drafter's backend, model and availability, sent when it flips).

### 16.5 Error codes

Protocol-level codes are listed below. Every `Error` event also carries the full error object from §29.3 (`data.error`: mill code, class, message, where, suggestions, help).

`bad_intent`, `bad_request`, `not_found`, `ambiguous`, `conflict`, `refused`, `bad_state`, `budget_exceeded`, `part_error`, `lint_failed`, `unauthorized` (gateway only), `line_too_long`, `unsupported_version`. After `line_too_long` or `unsupported_version` the daemon closes the connection.


---

[← spec index](index.md)
