# Protocols

The daemon wire protocol (client ↔ milld, spec §16) and the parts protocol (milld ↔ part, spec §9.3) as concrete message catalogs. Both use one codec, `internal/wire/jsonl`, copied from rex's `internal/wire/protocol/codec.go`.

Layers ([layers.md](layers.md)): the codec is L0 (`wire/jsonl`); message structs are L1 (`wire/protocol`, `partproto`), the published languages other programs read [DDD ch. 14]; the client is L5 (`wire/client`), the server L6 (`daemon/server`), the part host L5 (`parthost`). Compatibility rules for both protocols: [versioning.md](versioning.md). Citations: [references.md](references.md); sources table in §4.

## 1. Shared envelope and codec

```json
{"v":1,"kind":"Intent","type":"StartRun","id":"c7","data":{…}}
```

| Field | Daemon protocol | Parts protocol |
|---|---|---|
| `v` | `wire.ProtocolVersion` = 1 | `partproto.Version` = 1 (independent constant; the two evolve separately) |
| `kind` | `Intent` (client → milld), `Event` (milld → client) | `Intent` (milld → part), `Event` (part → milld). Same values as the daemon so the codec is unchanged. The spec doesn't name parts-protocol kinds; this fills the gap. |
| `type` | message name | message name |
| `id` | correlation id; an intent with `id` gets exactly one reply event with the same `id` | correlation for request/response pairs (`Describe`, `Ping`, `PartCall`) |
| `ctx` | absent | context id for multiplexing (one node-run/trigger/sink session), on every message except `Describe`, `Ping`, `Pong`, `Shutdown`. Opaque, assigned by the host per process (`c17`), mapped to `(run, node-run)` in the host. New optional envelope field: additive, so no version bump. |
| `data` | payload object | payload object |

Codec rules (rex behavior kept): one object per line; max line 16 MiB (`ErrLineTooLong` is sticky and closes the connection after an `Error{line_too_long}`); `ErrMalformed` consumes the bad line and continues; `ErrUnsupportedVersion` closes after `Error{unsupported_version}`; `[]byte` fields are base64; unknown fields are ignored; reads resume after a deadline timeout without losing bytes. The daemon encodes events once (`EncodeEvent`) and fans the same bytes out to every subscriber.

## 2. Daemon protocol

### 2.1 Transport

Unix domain socket on macOS, Linux **and Windows** (AF_UNIX works on Windows 10 1803+, which rex already relies on). The spec's "named pipe on Windows" (§16) isn't needed; one transport means one code path and one set of tests. Socket file `0600` in a `0700` dir; the server checks the peer uid via `SO_PEERCRED` / `LOCAL_PEERCRED` (rex `peercred_*.go`) and drops other users. The gateway (M7) carries the same envelopes over WebSocket with a bearer token.

### 2.2 Handshake and versions

```jsonl
{"v":1,"kind":"Intent","type":"Hello","id":"h1","data":{"client":"mill-cli","client_version":"0.1.0","features":["acks","stream_input"]}}
{"v":1,"kind":"Event","type":"Snapshot","id":"h1","data":{"daemon_version":"0.1.0","protocol":1,"features":["acks","stream_input","repaint"],"data_dir":"/home/dev/.local/share/mill","pid":48213,"started_at":"2026-11-11T08:00:00Z","runs":[{"id":"0b9e…","pipeline":"issue-to-pr","number":42,"state":"working","nodes_done":4,"nodes_total":9,"tokens":812000,"cost_micro_usd":4100000,"started_at":"2026-11-11T12:04:00Z","text":"build · rate limited · retry in 40s"}],"pipelines":[{"name":"issue-to-pr","scope":"project","hash":"sha256:5d1c…","triggers":[{"id":"github","kind":"webhook","enabled":true}],"lint":{"errors":0,"warnings":1},"last":{"run":"0b9e…","number":42,"state":"working"},"next_fire":null}],"summary":{"failed":1,"running":2,"ok_today":14,"cost_today_micro_usd":6200000},"config_error":null}}
```

- `features` lets either side detect optional behavior without a version bump. The client compares `daemon_version` with its own: older daemon → `M7003` with `help: mill daemon restart` (spec §29.9); newer daemon is fine.
- No broadcast reaches a connection before its `Snapshot` (invariant 2): the connection's subscription is registered and the snapshot is enqueued under the same lock, so nothing is missed or doubled (rex `withOutputHeld` pattern).
- Intents before `Hello` → `Error{bad_intent, "send Hello first"}`.

### 2.3 Intents

`→` request, `←` reply. Every reply carries the intent's `id`.

| Intent | Payload | Reply | Notes |
|---|---|---|---|
| `Hello` | `client`, `client_version`, `features` | `Snapshot` | |
| `ListPipelines` | `scope?`, `filter?` | `Pipelines{items[]}` | |
| `GetPipeline` | `name` | `Pipeline{source, hash, layout, graph, lint}` | |
| `SavePipeline` | `name`, `source`, `layout?`, `base_hash`, `force?` | `Ack{hash}` / `Error{conflict}` M2201 / `Error{lint_failed}` | lint runs first unless `force` |
| `EditPipeline` | `name`, `base_hash`, `ops[]` (node add/set/rm/mv, edge add/rm) | `Ack{hash, diff}` | **new**: server-side `yamledit` so CLI and TUI edits share one implementation |
| `Lint` | `source` or `name` | `LintResult{diagnostics[]}` | |
| `Compile` | `pipe` | `Pipeline` | pipe expressions |
| `DraftPipeline` | `sentence`, `base?` | `Draft{steps, trigger, agent, effort, name, description, reasons, drafter}` | spec §16.3 (wizard, ADR-047); runs `drafter` (L5) in the daemon; M2212 on fallback |
| `StartRun` | `pipeline` or `source`, `inputs`, `items?`, `input_stream?`, `budget?`, `labels?`, `priority?`, `dry_run?` | `Ack{run_id, number}` | `RunAdded` is broadcast before the Ack |
| `RunInput` | `run_id`, `items[]` | `Ack` | **new**: streaming stdin when `input_stream:true`; batches ≤ 1 MiB; flow-controlled by Acks (client sends the next batch after the Ack) |
| `RunInputEnd` | `run_id` | `Ack` | **new** |
| `RunNode` | `pipeline` or `source`, `node`, `items` or `from_run`, `dry_run?` | `Ack{run_id}`; items follow as `ItemEmitted` | spec §16.3 (editor test-run of one node) |
| `Estimate` | `pipeline` or `source` | `Estimate{nodes[], total}` (low/high cost, tokens, time) | spec §16.3; from registry prices and `daily` history |
| `CancelRun` | `run_id`, `reason` | `Ack` (requested, not finished) | |
| `DeleteRun` | `run_id` | `Ack` after teardown and removal | |
| `RerunFrom` | `run_id`, `node?`, `budget?` | `Ack{run_id, number}` | |
| `RetryNode` | `run_id`, `node_run_id` | `Ack{node_run_id}` | manual attempt |
| `Subscribe` | `run_id?` (empty = board-wide), `node_run_id?`, `from_seq?`, `replay?`, `repaint?`, `items?` | `Ack{seq}` after replay | view-only; there is no input intent |
| `Unsubscribe` | `run_id?`, `node_run_id?` | `Ack` | **new** |
| `Resize` | `node_run_id`, `cols`, `rows` | `Ack` | |
| `ListRuns` | `pipeline?`, `state?`, `since?`, `limit`, `cursor` | `Runs{items[], cursor}` | from index |
| `GetRun` | `run_id` or `selector` | `Run{…, node_runs[]}` | selector resolution in the daemon: `Error{ambiguous}` lists matches |
| `GetItems` | `run_id`, `node`, `port?`, `item?`, `limit`, `cursor` | `Items{items[], cursor}` | |
| `GetLogs` | `run_id`, `node_run_id?`, `stream?`, `from_line?`, `tail?`, `raw?` | `Logs{lines[] or bytes}` | **new**: `mill logs` without reading files directly |
| `SetTrigger` | `pipeline`, `trigger`, `enabled` | `Ack` | |
| `ListTriggers` | `pipeline?` | `Triggers{items[]}` (armed, next fire, last fire, errors) | **new** |
| `FireTrigger` | `pipeline`, `trigger?`, `event?` | `Ack{run_id?}` | **new**: `mill trigger fire` |
| `PartCall` | `part`, `method`, `data` | `PartResult{data}` | |
| `ListParts` | `kind?` | `Parts{items[]}` (id, source, version, health) | **new**: `mill part ls`, boot `parts.load` |
| `Stats` | `since`, `group_by` | `StatsResult` | **new**: `mill stats`, `:stats`, digest |
| `CacheList` / `CacheClear` | filters | `Cache{…}` / `Ack{removed}` | **new** |
| `Prune` | `older`, `dry_run` | `PruneResult` | **new** |
| `Reload` | none | `Ack` once config, pipelines, parts reloaded | |
| `Shutdown` | `drain?` | `Ack` (best effort) | |

The spec's intent table (§16.3) has no wire path for several CLI commands; the **new** rows close those gaps. Commands that only touch local files (`config`, `secrets set`, `trust`, `lint` of a file) stay in the CLI.

### 2.4 Events

| Event | When | Payload (key fields) |
|---|---|---|
| `Snapshot` | reply to `Hello` | above |
| `Ack` | success reply | `intent`, plus intent-specific fields |
| `Error` | failure reply or protocol violation | `code` (protocol), `error` (§29.3 object) |
| `RunAdded`, `RunUpdated`, `RunRemoved` | board-wide | run summary (same shape as `Snapshot.runs[]`); `RunUpdated` is coalesced per run to ≤ 10/s |
| `NodeRunUpdated` | subscribers of that run | `run_id`, `node_run_id`, `node`, `state`, `reason`, `text`, `progress`, `usage`, `started_at`, `finished_at`, `error?` |
| `ItemEmitted` | subscribers of that run | counts per port; payloads only if `Subscribe.items` |
| `RunLog` | subscribers of that run | one rendered log line `{ts, node_run_id, level, msg}` |
| `NodeOutput` | subscribers of that node-run | `bytes` (base64 PTY/stdout chunk), `seq`; first one is the repaint when `repaint:true` |
| `RunEvent` | subscribers with `replay` or `from_seq` | raw store event (redacted) — the faithful-replay channel for `mill watch --json` and `mill events` |
| `TriggerFired` | board-wide | `pipeline`, `trigger`, `run_id?`, `skipped?`, `reason` |
| `PipelineChanged` | board-wide | `name`, `hash`, `lint` summary, `removed?` |
| `PartStatus` | board-wide | `part`, `health`, `error?` |
| `Usage` | board-wide, coalesced 1/s | today totals |
| `ConfigStatus` | board-wide | `ok` or the M8xxx error (the line under the board header, §29.8) |

### 2.5 Examples

Start, follow and finish a run:

```jsonl
{"v":1,"kind":"Intent","type":"StartRun","id":"c2","data":{"pipeline":"morning-brief","inputs":{"since":"24h"},"labels":{"by":"cli"}}}
{"v":1,"kind":"Event","type":"RunAdded","data":{"id":"5e21…","pipeline":"morning-brief","number":64,"state":"queued"}}
{"v":1,"kind":"Event","type":"Ack","id":"c2","data":{"intent":"StartRun","run_id":"5e21…","number":64}}
{"v":1,"kind":"Intent","type":"Subscribe","id":"c3","data":{"run_id":"5e21…","replay":true}}
{"v":1,"kind":"Event","type":"RunEvent","data":{"run_id":"5e21…","event":{"seq":1,"type":"RunCreated","…":"…"}}}
{"v":1,"kind":"Event","type":"Ack","id":"c3","data":{"intent":"Subscribe","seq":3}}
{"v":1,"kind":"Event","type":"NodeRunUpdated","data":{"run_id":"5e21…","node_run_id":"fetch@1","node":"fetch","state":"working"}}
{"v":1,"kind":"Event","type":"RunUpdated","data":{"id":"5e21…","state":"done","outputs":{"path":"~/notes/brief.md"},"duration_ms":48000}}
```

Error with the §29.3 object:

```json
{"v":1,"kind":"Event","type":"Error","id":"c9","data":{"code":"not_found","error":{"code":"M1201","class":"not_found","message":"pipeline \"isue-to-pr\" not found","where":{"pipeline":"isue-to-pr"},"suggestions":["issue-to-pr"],"help":["mill ls"],"retryable":false,"cause":null,"docs":"mill explain M1201"}}}
```

### 2.6 Flow control and ordering

- Per connection: one bounded outbound queue (1024 lines) drained by one writer goroutine with a 5 s write timeout (rex `clientConn`). A client that falls behind is **disconnected**, never allowed to stall the daemon; it reconnects, re-`Hello`s and re-subscribes `from_seq`.
- Ordering: on one connection, events are delivered in the order the daemon enqueued them; "broadcast before Ack" holds because both go through the same queue.
- High-rate events are coalesced at the source: `RunUpdated` ≤ 10/s per run, `NodeOutput` chunks ≥ 16 ms apart (merged), `Usage` 1/s.
- `RunInput` is ack-paced (one batch in flight), so a fast `jq | mill run --stdin` never floods the daemon.

### 2.7 Versioning

Per-surface rules, the compatibility matrix and the upgrade path are in [versioning.md](versioning.md) [RI ch. 14].

Additive changes (new optional fields, new intents/events, new `features`) don't bump `v`. Meaning changes bump `v`; the daemon then speaks both versions for one minor release (dispatch on `v` per line). Unknown intent → `Error{bad_intent}`; unknown event → ignored by clients.

## 3. Parts protocol

### 3.1 Process model and transport

stdin/stdout JSONL, stderr captured to `logs/<part>.stderr.log` (and to the node-run's part log while a single context is open). Two process kinds, declared in the manifest (`lifetime: per_call | long_lived`). Long-lived parts multiplex contexts by `ctx`. Stdio only (spec §30 item 4 leans that way; remote parts go through runners).

### 3.2 Message catalog

milld → part (`kind: Intent`):

| Type | `id` | `ctx` | Payload |
|---|---|---|---|
| `Describe` | yes | no | `{protocol: 1, host_version}` |
| `Open` | no | yes | `{capability, context:{run, node, node_run, attempt, item_index, dry_run}, with, workspace, secrets:{NAME: value}, budget:{tokens, cost_micro_usd, wall_ms}, scratch, window}` |
| `Item` | no | yes | `{port, item}` (envelope `core.Item`, `files[].path` filled for local reads) |
| `EndOfInput` | no | yes | `{}` |
| `Credit` | no | yes | `{n}` output credit (stream mode, §3.4) |
| `Cancel` | no | yes | `{reason, grace_ms}` |
| `Resume` | no | yes | `{checkpoint}` (instead of a fresh run after `Open`) |
| `Close` | no | yes | `{}` |
| `Ping` | yes | no | `{}` |
| `PartCall` | yes | no | `{method, data}` |
| `Shutdown` | no | no | `{grace_ms}` |

part → milld (`kind: Event`):

| Type | `id` | `ctx` | Payload |
|---|---|---|---|
| `Manifest` | reply to `Describe` | no | below |
| `Item` | no | yes | `{port, item}` (part may send bare `{port, data}`; host builds the envelope) |
| `Credit` | no | yes | `{n}` input credit (§3.4) |
| `Log` | no | yes/no | `{level, msg, attrs}` |
| `Progress` | no | yes | `{pct, msg}` |
| `Usage` | no | yes | `{tokens_in, tokens_out, cost_micro_usd?, model?}` (deltas) |
| `Artifact` | no | yes | `{name, path, mime}` (file in the scratch dir; host moves it into the CAS) or `{name, data_b64}` for ≤ 1 MiB |
| `Checkpoint` | no | yes | `{data}` (≤ 1 MiB) |
| `Done` | no | yes | `{outputs_summary?, result_text?}` |
| `Fail` | no | yes | `{class, code, message, retryable, retry_after_ms?, exit_code?, details?}` |
| `Pong` | reply | no | `{load?}` |
| `PartResult` | reply | no | `{data}` |
| `Error` | reply or `ctx` | opt | `{code: "unsupported" \| "bad_request" \| "internal", message}` |

`Manifest` example:

```json
{"v":1,"kind":"Event","type":"Manifest","id":"d1","data":{
  "id":"acme.linear","version":"1.2.0","protocol":1,"lifetime":"long_lived","max_contexts":16,
  "capabilities":[
    {"kind":"node","id":"linear.issue.create","glyph":"◩",
     "config_schema":{"type":"object","required":["team","title"],"properties":{"team":{"type":"string"},"title":{"type":"string"},"body":{"type":"string"},"labels":{"type":"array","items":{"type":"string"}}}},
     "ports":{"in":["in"],"out":["out","error"]},
     "output_schema":{"type":"object","properties":{"id":{"type":"string"},"url":{"type":"string"}}},
     "flags":{"streaming":false,"cancellable":true,"resumable":false,"idempotent":false,"destructive":true,"needs_secrets":["LINEAR_API_KEY"],"agent":false},
     "calls":["teams","labels"]}
  ]}}
```

A per-call `sh`-like node-run, end to end:

```jsonl
{"v":1,"kind":"Intent","type":"Describe","id":"d1","data":{"protocol":1,"host_version":"0.5.0"}}
{"v":1,"kind":"Event","type":"Manifest","id":"d1","data":{"id":"my.upper","version":"0.1.0","protocol":1,"lifetime":"per_call","capabilities":[{"kind":"node","id":"my.upper","flags":{"idempotent":true}}]}}
{"v":1,"kind":"Intent","type":"Open","ctx":"c17","data":{"capability":"my.upper","context":{"run":"0b9e…","node":"upper","node_run":"upper@1","attempt":1},"with":{},"scratch":"/…/scratch/upper@1","window":64}}
{"v":1,"kind":"Intent","type":"Item","ctx":"c17","data":{"port":"in","item":{"data":{"t":"hi"},"meta":{"id":"src@1/out/0"}}}}
{"v":1,"kind":"Intent","type":"EndOfInput","ctx":"c17","data":{}}
{"v":1,"kind":"Event","type":"Item","ctx":"c17","data":{"port":"out","data":{"t":"HI"}}}
{"v":1,"kind":"Event","type":"Done","ctx":"c17","data":{}}
{"v":1,"kind":"Intent","type":"Close","ctx":"c17","data":{}}
```

### 3.3 Rules

1. A context ends with exactly one `Done` or `Fail`. Anything after it on that `ctx` is dropped and logged. A part that exits without one → `Fail{class: internal, code: M5202 "part exited without finishing"}` with the stderr tail.
2. After `Cancel`, the part must send `Fail{class: cancelled}` within `grace_ms`, or the process group is killed. For long-lived parts, only that context is failed; the process is killed only if it ignores two cancels in a row.
3. Secrets arrive only in `Open.secrets`, only the names granted (spec §9.3). The host redacts them out of everything the part sends back.
4. `Health`: idle long-lived parts get `Ping` every 30 s; no `Pong` in 5 s → restart with backoff (1 s → 60 s), open contexts fail `transient` (retryable).
5. `Describe` results are cached by executable hash; a changed binary is re-described (and refused if it doesn't match the lock, M5301).
6. Unknown message types are ignored; a part that can't do something answers `Error{unsupported}`.

### 3.4 Flow control

- **Per-call parts** (one context per process): OS pipe backpressure is enough. The host writes `Item`s from a dedicated goroutine; a part that stops reading blocks only its own writer.
- **Long-lived parts** (shared stdin): credit-based so one slow context can't block others. `Open.window` = initial input credit (default 64 items). The host sends at most `window` un-credited `Item`s per context; the part returns `Credit{n}` as it consumes. Parts that never send `Credit` get one window per context, which suffices for batch nodes that read everything then work.
- **Output** is always read eagerly by the host (and written to disk), so parts never block on output — except in `stream` mode, where the host sends output `Credit` and the part must not exceed it.

### 3.5 Built-ins in-process, same protocol

Built-ins implement `partproto.Part` (Go interface in [architecture.md](architecture.md) §4.2) whose methods carry **the same Go structs** that the stdio transport marshals (`OpenData`, `ItemData`, `FailData` …). Three mechanisms keep built-ins honest:

1. **Loopback transport.** `partproto/loopback` wraps any in-process `Part` in a pair of `io.Pipe`s and the real JSONL codec. The conformance suite (`partproto/conformance`) runs every built-in through both the direct path and loopback; results must be identical. CI also runs the full example suite with `MILL_PARTS_LOOPBACK=1`.
2. **`milld part serve <id>`** (hidden subcommand) exposes any built-in over stdio. A user part can wrap or shadow a built-in by id (spec §10), and `plugins.sandbox = "container"` can later run built-ins in containers too.
3. **No private hooks.** What built-ins get beyond the protocol is `HostServices` (runner, workspace ops, sub-runs, expression evaluator, clock), all interfaces declared in `partproto` (L1) and implemented in L4 [CA ch. 11]. The agent built-ins also receive the run's frozen agent registry from `parthost` at construction (architecture.md §4.2). Each has a protocol equivalent or is explicitly "host-only" in the manifest (`requires_host: ["runner"]`); external parts needing those use the SDK's equivalents (they run their own processes under the runner the host selected for them).

Lua parts implement the same lifecycle through function calls (`run(ctx, item)`, `ctx.emit`, `ctx.fail`), adapted to `partproto.Part` by `parthost/luapart`.

### 3.6 SDK (Go first)

`sdk/go/millpart`: `millpart.Serve(manifest, handlers)` handles Describe, Open/Item/EndOfInput/Cancel/Close, credits, pings and panics (→ `Fail{internal}`). It imports only `internal/wire/jsonl`-equivalent code copied into the SDK module (the SDK is a separate module so plugin authors don't depend on mill's internals). TS and Python SDKs follow the same message tables (M9).

## 4. Sources

| Decision | Section | Follows |
|---|---|---|
| One JSONL envelope for both protocols; text on the wire | §1 | [TAOUP ch. 5] [EIP, Envelope Wrapper] |
| `id` correlates request and reply; `v` marks the format | §1 | [EIP, Correlation Identifier] [EIP, Format Indicator] |
| Additive evolution, unknown fields ignored; v and v−1 | §1, §2.7 | [DDIA ch. 4] [RI ch. 14] |
| Loose contracts plus a conformance suite as the consumer test | §3.5 | [SAHP ch. 13] |
| Handshake with versions and features | §2.2 | [RI ch. 5, Handshaking] |
| No broadcast before `Hello`; broadcast before `Ack` on one queue | §2.2, §2.6 | [EIP, Durable Subscriber] (resume `from_seq`) |
| Bounded per-connection queue, drop slow clients | §2.6 | [RI ch. 5, Create Back Pressure] [SRE ch. 22] |
| Coalesced high-rate events | §2.6 | [RI ch. 5, Shed Load] |
| Ack-paced `RunInput`; credits for multiplexed parts | §2.6, §3.4 | [RI ch. 5, Create Back Pressure] [CIG ch. 4, Queuing] |
| Parts as processes over stdio | §3.1 | [TAOUP ch. 7] [FSA ch. 12] |
| Exactly one `Done`/`Fail` per context; kill after ignored cancels | §3.3 | [RI ch. 5, Timeouts] |
| Health pings, restart with backoff | §3.3 | [DBI ch. 9] [CIG ch. 5, Healing Unhealthy Goroutines] |
| Built-ins in-process behind the same contract | §3.5 | [FSA ch. 12] (microkernel plug-ins) |
| SDK as a separate module | §3.6 | [SAHP ch. 8] |
