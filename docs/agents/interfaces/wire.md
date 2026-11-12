# Wire protocol (agents)

Contract summary of the daemon protocol (client ↔ `milld`). Reference: **[spec §16](../../spec/16-wire-protocol.md#16-daemon-wire-protocol)**. Message catalog with JSON examples: [../../plans/backend/protocols.md](../../plans/backend/protocols.md) §2. User doc (when shipped): [../../protocol.md](../../protocol.md). Code: [../modules/protocol.md](../modules/protocol.md), [../modules/client.md](../modules/client.md), [../modules/server.md](../modules/server.md). Status: planned.

## Transport

Newline-delimited JSON over a local socket. Spec says Unix domain socket, named pipe on Windows; the backend plan proposes AF_UNIX everywhere (backend S6, D22). Optionally the same messages over WebSocket through the gateway.

## Envelope

| Field | Required | Meaning |
|---|---|---|
| `v` | yes | Protocol version, `1`. Other values are rejected |
| `kind` | yes | `Intent` (client → daemon) or `Event` (daemon → client) |
| `type` | yes | Message name |
| `id` | no | Correlation id: an intent with `id` gets exactly one reply with the same `id` (`Ack`, its specific reply, or `Error`) |
| `data` | yes | Payload |

Max line 16 MiB. Bytes are base64. Unknown fields ignored; unknown intent types → `Error` `bad_intent`.

## Handshake

1. Client sends `Hello` (`client`, `client_version`).
2. Daemon replies `Snapshot` (active runs, pipelines, `daemon_version`).
3. Live events follow. Nothing is broadcast to a connection before `Hello` completes.

## Intents

`Hello`, `ListPipelines`, `GetPipeline`, `SavePipeline` (with `base_hash`; stale → `conflict`), `Lint`, `Compile` (pipe expression), `StartRun`, `CancelRun`, `DeleteRun`, `RerunFrom`, `RetryNode`, `Subscribe` (view-only, `replay`, `repaint`), `Resize`, `ListRuns`, `GetRun`, `GetItems`, `SetTrigger`, `PartCall`, `Reload`, `Shutdown`. Payloads: [§16.3](../../spec/16-wire-protocol.md#163-intents).

There is no input intent. Clients never type into a node (hands-off, [ADR-022](../DECISIONS.md#adr-022-view-only-attach)).

The backend plan adds intents the CLI needs (trigger fire/list, stats, cache, prune, part list, logs, streamed stdin, node/edge edits; backend S13). Not accepted yet.

## Events

`Snapshot`, `Ack`, `Error`, `RunAdded`, `RunUpdated`, `RunRemoved`, `NodeRunUpdated`, `ItemEmitted` (counts unless subscribed), `RunLog`, `NodeOutput` (PTY bytes for attached clients), `TriggerFired`, `PipelineChanged`, `PartStatus`, `Usage`.

## Errors

Protocol codes: `bad_intent`, `bad_request`, `not_found`, `ambiguous`, `conflict`, `refused`, `bad_state`, `budget_exceeded`, `part_error`, `lint_failed`, `unauthorized` (gateway), `line_too_long`, `unsupported_version`. The daemon closes the connection after the last two. Every `Error` also carries `data.error`, the full [§29.3](../../spec/29-errors.md#293-json-output) object.

## Change rules

- Additive (new optional field, new message type): no version bump.
- Changing a meaning: bump `v`.
- The wire version is independent of the parts protocol version and the binary version.
