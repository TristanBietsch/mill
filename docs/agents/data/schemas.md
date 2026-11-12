# Schemas

Data shapes on the wire and on disk. Status: planned. Sources: the spec (contract) and the backend plan (proposed detail). Once code exists, Go struct tags are the source of truth and this file follows them.

Field names are `snake_case`. Readers ignore unknown fields ([ADR-017](../DECISIONS.md#adr-017-additive-evolution)).

## Item envelope

What parts see on every edge ([spec §6.1](../../spec/06-data-model.md#61-items)).

| Field | Type | Meaning |
|---|---|---|
| `data` | any JSON | The payload. Expressions see it as `item` |
| `files` | list | Artifact refs: `name`, `path` (local view), `sha256`, `size`, `mime` |
| `ws` | object or null | Workspace ref: `kind` (`git`, `dir`, `volume`), `path`, `repo`, `branch`, `base` |
| `meta` | object | Lineage: `run`, `node`, `index`, `parents` (ids of source items), `ts` |

`io: plain` nodes see only `data`, one JSON value per line; `io: text` nodes see raw lines. mill re-wraps their output and carries `files`, `ws` and lineage forward.

Backend plan detail ([storage.md](../../plans/backend/storage.md) §2, §4): item id `<node-run>/<port>/<index>` (`plan@1/out/0`); items stored at `runs/<run-id>/items/<node-run>/<port>.jsonl`, one item per line, `path` stripped (computed at delivery); a line is visible downstream only after an `ItemsCommitted` event.

## Wire envelope

Every line on the daemon socket and every part's stdio ([spec §16.1](../../spec/16-wire-protocol.md#161-envelope)):

```json
{"v":1,"kind":"Intent","type":"StartRun","id":"c7","data":{"pipeline":"issue-to-pr","inputs":{}}}
```

| Field | Type | Notes |
|---|---|---|
| `v` | int | Protocol version (`1`); wire and parts versions are separate constants |
| `kind` | string | `Intent` or `Event` |
| `type` | string | Message name |
| `id` | string | Optional correlation id |
| `ctx` | string | Parts protocol only, proposed (backend S10): context id |
| `data` | object | Payload |

## Error object

Same shape in CLI `--json` output, protocol `Error` events (`data.error`), MCP and the gateway ([spec §29.3](../../spec/29-errors.md#293-json-output)).

| Field | Type | Notes |
|---|---|---|
| `code` | string | `M` + 4 digits |
| `class` | string | `user`, `not_found`, `ambiguous`, `conflict`, `refused`, `unreachable`, `transient`, `rate_limited`, `auth`, `timeout`, `budget`, `cancelled`, `agent`, `internal` |
| `message` | string | Headline |
| `where` | object | `file`, `line`, `col`, `node`, … |
| `suggestions` | string[] | `did you mean` candidates |
| `help` | string[] | Commands to run next |
| `retryable` | bool | |
| `cause` | object or null | Inner error, same shape |
| `docs` | string | `mill explain <code>` |

## Event log (`events.jsonl`)

Per run, append-only, written only by the run's owner ([spec §5.4](../../spec/05-architecture.md#54-invariants), invariant 4). The spec fixes the principle; the format below is the backend plan's proposal ([storage.md](../../plans/backend/storage.md) §3).

```json
{"seq":14,"ts":"2026-11-11T12:09:40.112Z","type":"NodeRunFinished","nr":"plan@1","data":{"state":"done","exit_code":0}}
```

| Field | Notes |
|---|---|
| `seq` | 1-based, gapless, per run |
| `ts` | RFC 3339 with ms, UTC, from the injected clock |
| `type` | Event type |
| `nr` | Node-run id (`<node>@<attempt>`, `<node>[<i>]@<attempt>`); absent for run-level events |
| `data` | Payload. The first line is always `RunCreated` with `log_v: 1` |

Proposed types: `RunCreated`, `RunStateChanged`, `NodeRunCreated`, `NodeRunStateChanged`, `NodeRunStarting` (write-ahead, before spawn), `NodeRunStarted`, `NodeRunFinished`, `NodeRunSkipped`, `WorkspaceCreated`, `WorkspaceCleaned`, `ItemsCommitted`, `ArtifactRegistered`, `Usage`, `RetryScheduled`, `Checkpoint`, `BudgetWarning`, `BudgetExceeded`, `CacheHit`, `SecretsResolved` (names only), `SubRunStarted`, `LoopIteration`, `RunCancelRequested`, `RunRecovered`, `Correction`, `Note`, `RunFinished`.

Never in the log: secret values, item payloads (in `items/`), raw output (in `logs/`).

## Run directory

`~/.local/share/mill/runs/<run-id>/` ([spec §20](../../spec/20-paths.md#20-files-and-paths)): `run.json` (derived snapshot), `events.jsonl`, `items/<node>/<n>.jsonl`, `logs/<node-run>.log` (raw output), `transcripts/`. The backend plan adds `run.lock`, `pipeline/` (frozen copy), `inputs.json`, `event.json`, split `.out.log`/`.err.log`/`.part.log`, and `scratch/` (storage.md §1).

Run id: 32 lowercase hex characters, random (backend D6, open). Run number: per-pipeline integer from 1, shown as `pipeline#42`.

## Pipeline file (`*.mill.yaml`)

Keys and rules: [../interfaces/pipeline-format.md](../interfaces/pipeline-format.md) and [spec §7](../../spec/07-pipeline-format.md#7-pipeline-file-format). Minimal shape:

```yaml
mill: 1
name: hello
nodes:
  greet:
    uses: sh
    with: { run: "echo '{\"greeting\":\"hi\"}'" }
    io: plain
outputs:
  greeting: ${{ nodes.greet.item.greeting }}
```

## Layout sidecar

`<name>.mill.layout.json` next to the pipeline ([spec §7.6](../../spec/07-pipeline-format.md#76-layout-sidecar)). Holds canvas positions only; missing means auto-layout; safe to delete or gitignore. Its fields are not specified yet; the editor plan ([../../plans/editor.md](../../plans/editor.md)) owns them. Rule: nothing in it may change what a pipeline does.

## Lockfile

`mill.lock` (project, `<repo>/.mill/`) and `~/.local/share/mill/plugins/lock` (user) ([spec §9.11](../../spec/09-parts-and-plugins.md#911-lockfile)). Per plugin:

| Field | Meaning |
|---|---|
| source | Registry name, `gh:` repo, git URL, dir, `.millpkg` or URL |
| version | Exact resolved version |
| archive hash | Hash of the package archive |
| executable hashes | Hash of every executable it ships |
| dependencies | Resolved `requires` |

Runs record which lock entry they used. `mill plugin sync --frozen` fails on any drift. File encoding is not specified; the backend plan proposes JSON with sorted keys (backend S26).

## Trust records

`mill trust` records approved files by hash ([spec §22](../../spec/22-security.md#22-security-and-trust)). Format not specified; the backend plan proposes JSON with sorted keys (backend S26).
