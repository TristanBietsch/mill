## 16. Daemon wire protocol

Newline-delimited JSON over a local socket (Unix domain socket; named pipe on Windows). Optionally the same messages over WebSocket through the gateway.

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
| `Hello` | `client`, `client_version` | `Snapshot` |
| `ListPipelines` | `scope` | `Pipelines` |
| `GetPipeline` | `name` | `Pipeline` (source, compiled graph, lint results) |
| `SavePipeline` | `name`, `source`, `layout`, `base_hash` | `Ack` with new hash; `Error` `conflict` if `base_hash` is stale |
| `Lint` | `source` or `name` | `LintResult` |
| `Compile` | `pipe` (pipe expression) | `Pipeline` |
| `StartRun` | `pipeline` or `source`, `inputs`, `items`, `budget`, `labels`, `detach`, `dry_run` | `Ack` with `run_id`, `number` |
| `RunNode` | `pipeline` or `source`, `node`, `items` (or `from_run`), `dry_run` | `Ack` with `run_id`; the node's items follow as `ItemEmitted` |
| `Estimate` | `pipeline` or `source` | `Estimate`: per node and total, low and high cost, tokens, time |
| `CancelRun` | `run_id`, `reason` | `Ack` |
| `DeleteRun` | `run_id` | `Ack` |
| `RerunFrom` | `run_id`, `node` (optional) | `Ack` with new `run_id` |
| `RetryNode` | `run_id`, `node_run_id` | `Ack` |
| `Subscribe` | `run_id`, optional `node_run_id`, `replay`, `repaint` (view-only; there is no input intent) | `Ack` after replay |
| `Resize` | `node_run_id`, `cols`, `rows` | `Ack` |
| `ListRuns` | filters, `limit`, `cursor` | `Runs` |
| `GetRun` | `run_id` | `Run` |
| `GetItems` | `run_id`, `node`, `port`, `limit`, `cursor` | `Items` |
| `SetTrigger` | `pipeline`, `trigger`, `enabled` | `Ack` |
| `PartCall` | `part`, `method`, `data` (for part-specific UI, e.g. listing Slack channels) | part-defined |
| `Reload` | none | `Ack` once pipelines, parts and registries are reloaded |
| `Shutdown` | none | `Ack` (best effort) |

### 16.4 Events

`Snapshot`, `Ack`, `Error`, `RunAdded`, `RunUpdated`, `RunRemoved`, `NodeRunUpdated`, `ItemEmitted` (counts, not payloads, unless subscribed), `RunLog`, `NodeOutput` (PTY bytes for attached clients), `TriggerFired`, `PipelineChanged`, `PartStatus`, `Usage`.

### 16.5 Error codes

Protocol-level codes are listed below. Every `Error` event also carries the full error object from §29.3 (`data.error`: mill code, class, message, where, suggestions, help).

`bad_intent`, `bad_request`, `not_found`, `ambiguous`, `conflict`, `refused`, `bad_state`, `budget_exceeded`, `part_error`, `lint_failed`, `unauthorized` (gateway only), `line_too_long`, `unsupported_version`. After `line_too_long` or `unsupported_version` the daemon closes the connection.


---

[← spec index](index.md)
