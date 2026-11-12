# Storage

On-disk layout, event log records, items and artifacts, the SQLite index, durability and locking. Files are the truth (spec §2.5); `index.db` is a cache that `mill doctor --reindex` rebuilds from files.

## 1. Directory layout

Base dirs from `internal/paths` (spec §20). XDG variables are honored on Linux **and** macOS (`XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`, `XDG_RUNTIME_DIR`), then `MILL_CONFIG_DIR`, `MILL_DATA_DIR`, `MILL_STATE_DIR` (new), `MILL_SOCKET` override. Windows: config and data under `%APPDATA%\mill`, state and cache under `%LOCALAPPDATA%\mill`. Every directory mill creates is `0700`; every file `0600`, except CAS blobs (`0400`).

```
~/.config/mill/                     (config: user edits)
  init.lua  settings.lua  lua/  pipelines/  sounds/

~/.local/share/mill/                (data: must survive)
  daemon.lock                       held by milld for its lifetime (flock / LockFileEx)
  index.db  index.db-wal  index.db-shm
  live/<run-id>                     empty marker per live run (created at RunCreated, removed at RunFinished)
  runs/<run-id>/
    run.lock                        held by the run's owner (milld actor or a --local process)
    run.json                        derived snapshot, atomically rewritten on each run/node state change
    events.jsonl                    append-only event log (source of truth)
    pipeline/                       frozen copy of the pipeline bundle: <name>.mill.yaml + referenced files
    inputs.json                     run inputs (redacted)
    event.json                      trigger payload (redacted), if any
    items/<node-run>/<port>.jsonl   output items, one core.Item per line
    logs/<node-run>.out.log         raw stdout (PTY: the merged byte stream), redacted
    logs/<node-run>.err.log         raw stderr, redacted
    logs/<node-run>.part.log        structured Log messages from the part (JSONL)
    transcripts/<node-run>.screen   final vt10x screen dump for terminal agents
    scratch/<node-run>/             per node-run scratch (MILL_ARTIFACTS lives here); removed at end unless kept
  artifacts/sha256/<ab>/<cdef…>     content-addressed blobs (global, shared by all runs)
  artifacts/tmp/                    in-flight writes (same filesystem, for atomic rename)
  cache/<k[:2]>/<key>/entry.json + items/…   node output cache
  triggers/<pipeline>/<trigger>.json         persistent trigger state (last fire, seen keys, queued fires)
  plugins/                          installed plugins + user lock
  workspaces/                       default workspace_root (worktrees, temp dirs)

~/.local/state/mill/                (state: may be lost)
  milld.log  logs/tui.log  logs/cli-errors.log  tui-state.json  status.json  (prompt/completion status file)

socket: $XDG_RUNTIME_DIR/mill/milld.sock, else ~/.cache/mill/milld.sock (decision D20)
```

Notes:
- The spec puts artifacts "under the run's artifact store" (§6.3) and also in a global `artifacts/` (§20). We use the **global CAS** with per-run references: dedupe across runs and cache entries, cheap `rerun --from`, GC by reachability (§7).
- `node-run` in file names is the node-run id: `plan@1`, `label[3]@1`. `@`, `[` and `]` are legal on all three OSes. Node ids are `[a-z0-9][a-z0-9_-]*` (lint), so no escaping is needed.
- Socket paths are limited to 104 bytes on macOS and 108 on Linux. `paths.Socket()` checks the length and fails with M7002 and a hint to set `MILL_SOCKET`.

## 2. Identifiers

| Id | Format | Example | Notes |
|---|---|---|---|
| run id | 32 lowercase hex chars from `crypto/rand` | `0b9e4c1a…` | Random, not time-ordered: short-id prefix matching (§17.3, ≥ 4 hex) needs uniformly random prefixes. UUIDv7/ULID put the timestamp first, so runs started the same minute share prefixes. Ordering comes from `created_at`. |
| run number | per-pipeline integer, from 1 | `issue-to-pr#42` | Allocated with `BEGIN IMMEDIATE` on `index.db` (works across milld and `--local` processes). Stored in `run.json` and `RunCreated`, so a rebuilt index recovers it; rebuild sets `counters.last_number = max(number)`. |
| node-run id | `<node>@<attempt>` / `<node>[<i>]@<attempt>` | `build@2`, `label[3]@1` | Unique within a run. Matches the CLI selector syntax `run/node[n]`. |
| item id | `<node-run>/<port>/<index>` | `plan@1/out/0` | Deterministic; used in `meta.parents`. |
| event seq | per-run monotonically increasing uint64 from 1 | | `Subscribe` resumes from a seq. |
| artifact id | sha256 hex | | |
| workspace id | `ws_<run-short>_<node>[_<i>]` | `ws_0b9e_build` | |

## 3. Event log

`events.jsonl`: one JSON object per line, UTF-8, `\n`-terminated, written only by the run's owner. Envelope:

```json
{"seq":14,"ts":"2026-11-11T12:09:40.112Z","type":"NodeRunFinished","nr":"plan@1","data":{…}}
```

| Field | Meaning |
|---|---|
| `seq` | 1-based, gapless |
| `ts` | RFC 3339 with ms, UTC, from the injected clock |
| `type` | event type (below) |
| `nr` | node-run id, absent for run-level events |
| `data` | payload; unknown fields ignored by readers (spec §2.12) |

The first line is always `RunCreated` and carries `"log_v":1` (event-log format version, independent of the wire and parts protocol versions).

### 3.1 Catalog with examples

All values synthetic.

```jsonl
{"seq":1,"ts":"2026-11-11T12:04:00.501Z","type":"RunCreated","data":{"log_v":1,"run":"0b9e4c1a7d2f4e6b9a1c3d5e7f901234","pipeline":"issue-to-pr","number":42,"scope":"project","project_dir":"/home/dev/code/app","bundle_hash":"sha256:5d1c…","config_hash":"sha256:a07e…","lock":{"mill/github":"1.4.2"},"mill_version":"0.1.0","trigger":{"kind":"webhook","id":"github","fire_id":"f_19c2"},"inputs_ref":"inputs.json","event_ref":"event.json","budget":{"max_tokens":2000000,"max_cost_usd":15,"max_wall":"1h","max_agents":3},"labels":{"tag":"code"},"priority":0,"dry_run":false,"local":false,"parent":null}}
{"seq":2,"ts":"2026-11-11T12:04:00.502Z","type":"RunStateChanged","data":{"from":"queued","to":"working"}}
{"seq":3,"ts":"2026-11-11T12:04:00.503Z","type":"NodeRunCreated","nr":"ws@1","data":{"node":"ws","attempt":1,"item":null,"mode":"batch","part":"git.worktree@builtin","runner":"local","flags":{"idempotent":true,"destructive":false,"resumable":false}}}
{"seq":4,"ts":"2026-11-11T12:04:00.503Z","type":"NodeRunStateChanged","nr":"ws@1","data":{"from":"pending","to":"queued"}}
{"seq":5,"ts":"2026-11-11T12:04:00.504Z","type":"NodeRunStateChanged","nr":"ws@1","data":{"from":"queued","to":"working","grant":"g_118"}}
{"seq":6,"ts":"2026-11-11T12:04:00.611Z","type":"WorkspaceCreated","nr":"ws@1","data":{"id":"ws_0b9e_ws","kind":"git","path":"/home/dev/.local/share/mill/workspaces/0b9e/ws","repo":"/home/dev/code/app","branch":"mill/issue-1287","base":"3f2a9c1e…"}}
{"seq":7,"ts":"2026-11-11T12:04:01.320Z","type":"ItemsCommitted","nr":"ws@1","data":{"port":"out","through":0,"offset":412}}
{"seq":8,"ts":"2026-11-11T12:04:01.321Z","type":"NodeRunFinished","nr":"ws@1","data":{"state":"done","exit_code":null,"duration_ms":817,"items_out":{"out":1}}}
{"seq":9,"ts":"2026-11-11T12:04:01.330Z","type":"NodeRunStarting","nr":"plan@1","data":{"runner":"local","argv_redacted":["claude","-p","--output-format","json","--model","opus","<redacted:1204 bytes>"],"dir":"/home/dev/.local/share/mill/workspaces/0b9e/ws"}}
{"seq":10,"ts":"2026-11-11T12:04:01.402Z","type":"NodeRunStarted","nr":"plan@1","data":{"pid":48311,"pgid":48311,"proc_start":"2026-11-11T12:04:01.398Z","container":null}}
{"seq":11,"ts":"2026-11-11T12:06:12.000Z","type":"Usage","nr":"plan@1","data":{"tokens_in":180000,"tokens_out":30000,"cost_micro_usd":4950000,"model":"opus","source":"headless-json"}}
{"seq":12,"ts":"2026-11-11T12:09:40.100Z","type":"ArtifactRegistered","nr":"plan@1","data":{"name":"transcript.json","sha256":"9c1d…","size":48211,"mime":"application/json"}}
{"seq":13,"ts":"2026-11-11T12:09:40.110Z","type":"ItemsCommitted","nr":"plan@1","data":{"port":"out","through":0,"offset":1880}}
{"seq":14,"ts":"2026-11-11T12:09:40.112Z","type":"NodeRunFinished","nr":"plan@1","data":{"state":"done","exit_code":0,"duration_ms":338710,"items_out":{"out":1},"output_source":"headless-json","validated":true}}
{"seq":15,"ts":"2026-11-11T12:11:02.000Z","type":"NodeRunFinished","nr":"build@1","data":{"state":"failed","exit_code":1,"error":{"code":"M6201","class":"rate_limited","message":"claude is rate limited","retryable":true,"retry_after_ms":40000}}}
{"seq":16,"ts":"2026-11-11T12:11:02.001Z","type":"RetryScheduled","nr":"build@2","data":{"attempt":2,"of":3,"retry_at":"2026-11-11T12:11:42.001Z","delay_ms":40000,"reason":"class:rate_limited"}}
{"seq":17,"ts":"2026-11-11T12:20:00.000Z","type":"Checkpoint","nr":"sync@1","data":{"ref":"checkpoints/sync@1.json","size":220}}
{"seq":18,"ts":"2026-11-11T12:24:00.000Z","type":"BudgetWarning","data":{"limit":"max_tokens","used":1610000,"max":2000000}}
{"seq":19,"ts":"2026-11-11T12:30:00.000Z","type":"RunCancelRequested","data":{"reason":"user","by":"mill-cli"}}
{"seq":20,"ts":"2026-11-11T12:31:10.000Z","type":"RunRecovered","data":{"decisions":[{"nr":"build@2","was":"working","action":"crashed","code":"M4301","orphan_killed":true}]}}
{"seq":21,"ts":"2026-11-11T12:40:00.000Z","type":"LoopIteration","nr":"build@2","data":{"n":2,"check_exit":2,"tokens":91000}}
{"seq":22,"ts":"2026-11-11T12:41:00.000Z","type":"Correction","data":{"of_seq":11,"field":"cost_micro_usd","value":4500000,"why":"price table fixed"}}
{"seq":23,"ts":"2026-11-11T12:45:00.000Z","type":"RunFinished","data":{"state":"done","outputs_ref":"outputs.json","tokens_in":800000,"tokens_out":12000,"cost_micro_usd":4100000,"error":null}}
```

Other types: `NodeRunSkipped{reason}` (folded as a state change with reason), `CacheHit{key, source_run}`, `WorkspaceCleaned{policy}`, `SecretsResolved{names}` (names only), `BudgetExceeded{limit}`, `SubRunStarted{child}`, `Note{text}`. Corrections are new events (invariant 4).

Large payloads (inputs, trigger event, outputs, checkpoints) go in side files referenced by name, so the log stays small and fast to replay.

### 3.2 What is never in the log

Secret values (the redactor runs on every event before append; argv is logged through rex's argv redactor), raw item payloads (they're in `items/`), raw output (in `logs/`).

## 4. Items and logs

- **Items:** `items/<nr>/<port>.jsonl`, one `core.Item` per line, appended by the actor, `path` fields stripped (local views are computed at delivery). A line is visible downstream only after an `ItemsCommitted{through, offset}` event. Recovery truncates any bytes past the last committed offset.
- **Raw logs:** `logs/<nr>.out.log` / `.err.log` hold the bytes the process wrote, passed through the streaming redactor and nothing else (no ANSI stripping, no line splitting). `mill logs --raw` prints them; `mill logs` renders sanitized lines from them. When the disk fills, rex's `pausingTranscript` behavior applies: drop bytes for 30 s, retry, record the gap.
- **Part logs:** `Log` messages as JSONL `{ts, level, msg, attrs}`.
- Log rotation per node-run: cap at `logs.max_bytes_per_node_run` (default 256 MiB); beyond it keep the first 16 MiB and a rolling last 64 MiB, record the gap.

## 5. Artifacts (CAS)

- **Write:** stream into `artifacts/tmp/<random>` while hashing (sha256), `fsync`, then `rename` to `artifacts/sha256/<ab>/<rest>` (if it already exists, delete the temp: dedupe). `chmod 0400`. On Unix also `fsync` the parent directory once per new shard dir.
- **Register:** the actor appends `ArtifactRegistered` and the indexer adds an `artifact_refs` row.
- **Deliver:** `FileRef.Path` for downstream nodes is a read-only materialization in the consumer's scratch dir under its original name, made by hardlink (same filesystem) or copy. Agents and tools see `inputs/<name>`, not a hash path. `os.Root` is used for all of it.
- **Collect from `MILL_ARTIFACTS`:** at node-run end, every regular file under the dir (no symlinks followed, size cap `artifacts.max_file` default 2 GiB) is put into the CAS and attached to the node's output items' `files`.
- **External backend (S3, later):** `cas.Store` is an interface; a remote backend keeps the local CAS as a read-through cache. Not in v0.

## 6. SQLite index

Driver: `modernc.org/sqlite`. Opened by milld read-write; by the CLI read-only (`mode=ro`) for listing when the daemon is down; by `--local` read-write for its one run. Pragmas on every connection:

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;    -- index is a cache; durability comes from events.jsonl
PRAGMA busy_timeout = 5000;
PRAGMA foreign_keys = ON;
PRAGMA temp_store = MEMORY;
```

```sql
CREATE TABLE meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
); -- schema_version, built_at, mill_version, data_dir

CREATE TABLE counters (
  pipeline    TEXT PRIMARY KEY,
  last_number INTEGER NOT NULL
);

CREATE TABLE runs (
  id              TEXT PRIMARY KEY,
  pipeline        TEXT NOT NULL,
  number          INTEGER NOT NULL,
  scope           TEXT NOT NULL,            -- project | user | adhoc
  project_dir     TEXT,
  bundle_hash     TEXT NOT NULL,
  state           TEXT NOT NULL,
  reason          TEXT,                     -- cancel reason, budget limit
  error_code      TEXT,
  error_class     TEXT,
  error_message   TEXT,                     -- redacted headline only
  failed_node     TEXT,
  result_text     TEXT,                     -- short line for the board ("draft PR #1293")
  trigger_kind    TEXT,
  trigger_id      TEXT,
  parent_run      TEXT REFERENCES runs(id) ON DELETE SET NULL,
  priority        INTEGER NOT NULL DEFAULT 0,
  dry_run         INTEGER NOT NULL DEFAULT 0,
  local           INTEGER NOT NULL DEFAULT 0,
  created_at      INTEGER NOT NULL,         -- unix ms
  started_at      INTEGER,
  finished_at     INTEGER,
  tokens_in       INTEGER NOT NULL DEFAULT 0,
  tokens_out      INTEGER NOT NULL DEFAULT 0,
  cost_micro_usd  INTEGER NOT NULL DEFAULT 0,
  cost_unknown    INTEGER NOT NULL DEFAULT 0,
  nodes_total     INTEGER NOT NULL DEFAULT 0,
  nodes_done      INTEGER NOT NULL DEFAULT 0,
  labels_json     TEXT,
  run_dir         TEXT NOT NULL,            -- absolute; differs for --store imports
  last_seq        INTEGER NOT NULL,         -- last folded event
  log_size        INTEGER NOT NULL,         -- events.jsonl size when folded (staleness check)
  UNIQUE (pipeline, number)
);
CREATE INDEX runs_by_pipeline ON runs (pipeline, created_at DESC);
CREATE INDEX runs_by_created  ON runs (created_at DESC);
CREATE INDEX runs_live        ON runs (state) WHERE state IN ('queued','working');
CREATE INDEX runs_by_prefix   ON runs (id);  -- prefix search: id >= ? AND id < ?

CREATE TABLE node_runs (
  run_id          TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  id              TEXT NOT NULL,            -- plan@1, label[3]@1
  node            TEXT NOT NULL,
  item_index      INTEGER,
  attempt         INTEGER NOT NULL,
  state           TEXT NOT NULL,
  reason          TEXT,
  part            TEXT NOT NULL,
  runner          TEXT,
  agent           TEXT,
  model           TEXT,
  queued_at       INTEGER,
  started_at      INTEGER,
  finished_at     INTEGER,
  exit_code       INTEGER,
  error_code      TEXT,
  error_class     TEXT,
  tokens_in       INTEGER NOT NULL DEFAULT 0,
  tokens_out      INTEGER NOT NULL DEFAULT 0,
  cost_micro_usd  INTEGER NOT NULL DEFAULT 0,
  items_in        INTEGER NOT NULL DEFAULT 0,
  items_out       INTEGER NOT NULL DEFAULT 0,
  cache_key       TEXT,
  PRIMARY KEY (run_id, id)
);
CREATE INDEX node_runs_by_agent ON node_runs (agent, model, finished_at);

CREATE TABLE artifact_refs (
  sha256      TEXT NOT NULL,
  run_id      TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  node_run_id TEXT NOT NULL,
  name        TEXT NOT NULL,
  size        INTEGER NOT NULL,
  mime        TEXT,
  PRIMARY KEY (sha256, run_id, node_run_id, name)
);
CREATE INDEX artifact_refs_by_sha ON artifact_refs (sha256);

CREATE TABLE cache_entries (
  key          TEXT PRIMARY KEY,
  pipeline     TEXT NOT NULL,
  node         TEXT NOT NULL,
  part         TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  last_hit_at  INTEGER,
  hits         INTEGER NOT NULL DEFAULT 0,
  size         INTEGER NOT NULL,
  source_run   TEXT
);
CREATE TABLE cache_artifacts (
  key     TEXT NOT NULL REFERENCES cache_entries(key) ON DELETE CASCADE,
  sha256  TEXT NOT NULL,
  PRIMARY KEY (key, sha256)
);

CREATE TABLE usage_daily (            -- for mill stats / digest / board "$6.20 today"
  day        TEXT NOT NULL,           -- YYYY-MM-DD local
  pipeline   TEXT NOT NULL,
  agent      TEXT NOT NULL DEFAULT '',
  model      TEXT NOT NULL DEFAULT '',
  runs       INTEGER NOT NULL DEFAULT 0,
  failed     INTEGER NOT NULL DEFAULT 0,
  tokens_in  INTEGER NOT NULL DEFAULT 0,
  tokens_out INTEGER NOT NULL DEFAULT 0,
  cost_micro_usd INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (day, pipeline, agent, model)
);
```

**Write path.** Run actors send `IndexOp`s (fold results, not raw events) to the indexer goroutine on a bounded channel (4096). The indexer batches up to 200 ops or 50 ms per transaction. If the channel is full, the actor marks the run `index_dirty` in memory and the indexer re-folds that run from its log when it catches up; nothing blocks on SQLite.

**Staleness and rebuild.** On milld start the indexer compares, for every live run and every run modified since `meta.built_at`, `runs.log_size` with the file size and re-folds if they differ. `mill doctor --reindex` (or a missing/corrupt/old-schema `index.db`) rebuilds: create `index.db.new`, fold every `runs/*/events.jsonl` with the same reducer as the engine (parallel, `GOMAXPROCS` workers), set counters to max numbers, then atomically rename over the old file. Schema migrations are therefore "rebuild", never `ALTER`.

**Reads.** `ListRuns`, selectors (`issue-to-pr#42`, `#last`, prefix `0b9e`), `mill stats`, board summary, completion. Prefix search uses a range scan: `id >= '0b9e' AND id < '0b9f'`.

## 7. Retention and GC

| What | Policy | Mechanism |
|---|---|---|
| Finished runs | `retention` (30d) by `finished_at`; `mill prune --older 30d`; never prune a run referenced as `parent` of a live run | daily job in milld + on demand; deletes run dir (teardown path first if somehow live), index rows cascade |
| Cache entries | `cache.ttl` (30d since last hit) and `cache.max_bytes` (5 GiB, LRU) | same job |
| Artifacts | mark-and-sweep: roots = `artifact_refs` of remaining runs + `cache_artifacts`; sweep blobs not in roots **and** older than 1 h (grace for in-flight writes not yet indexed) | same job; `mill prune --dry-run` prints what would go |
| Workspaces | per node policy (`on_success`, `always`, `never`, `keep: 7d`) + setting `keep_workspaces`; orphan sweep of `workspace_root/<run>` dirs whose run is terminal and past keep | workspace manager at run end + daily |
| Triggers state | seen-key sets bounded by `dedupe_window` and a max of 10k keys per trigger | trigger host |
| `tmp/` and `scratch/` | remove anything older than 24 h on start | boot |

Pruning a run whose items are needed by `rerun --from` of a newer run is fine: the newer run copied (hardlinked) what it reused.

## 8. Durability (fsync policy)

| Write | Policy |
|---|---|
| `events.jsonl` | `O_APPEND`, single writer. Two durability levels per append: **Sync** (write + `fsync` before returning) for `RunCreated`, `NodeRunStarting` (write-ahead: recorded before the process is spawned), `NodeRunFinished`, `Checkpoint`, `RetryScheduled`, `RunCancelRequested`, `RunFinished`, `RunRecovered`; **Group** for everything else (`fsync` at most every 100 ms while dirty, and always before a Sync append). Clients only see Group events after they're written (not necessarily fsynced); losing the last ≤100 ms of Usage/Items events on power loss is acceptable because the next Sync event's fold re-derives counts from item files. |
| item files | written, then `fsync` before the `ItemsCommitted` event that exposes them |
| artifacts | temp + `fsync` + rename (+ dir fsync on new shard) |
| `run.json`, trigger state, settings.lua, pipeline saves | temp file in the same dir + `fsync` + rename (rex `WriteMeta` pattern) |
| `index.db` | `synchronous=NORMAL` (cache) |
| logs | no fsync (best effort; raw output loss on power cut is acceptable and documented) |

macOS note: `fsync` doesn't flush the drive cache; `F_FULLFSYNC` does but costs ~10–20 ms. Use `F_FULLFSYNC` only for `RunFinished` and `RunCreated`; plain `fsync` elsewhere.

## 9. Locking

| Lock | Holder | Purpose |
|---|---|---|
| `data/daemon.lock` | milld for its lifetime | one daemon per data dir (rex `AcquireLock`). A second milld exits with M7005 naming the pid. |
| `runs/<id>/run.lock` | the run's owner: a milld actor or a `--local` process | recovery skips runs whose lock is held (a `--local` run in progress isn't "crashed"); `mill rm` refuses (class `refused`, exit 6) while another process holds it |
| `index.db` | SQLite (WAL) | milld and `--local` processes both write; `busy_timeout` serializes |
| pipeline files | none; compare-and-swap on content hash | `SavePipeline{base_hash}` → `conflict` (M2201) if the file changed |
| `triggers/*.json` | milld only (trigger host) | |
| CAS | none (atomic rename, content-addressed) | |
| workspaces | per-run directories, owned by the run | |

Both lock kinds are advisory and released by the OS when the process dies, so a crash never leaves a stale lock.

## 10. `--store` and portability

A run directory is self-contained except for artifacts. `mill run --local --store ./mill-runs` writes runs and a private CAS under that directory (`./mill-runs/runs`, `./mill-runs/artifacts`, `./mill-runs/index.db`) so CI can upload it as one build artifact; `mill runs --store ./mill-runs` opens it read-only and rebuilds its index in memory if needed.
