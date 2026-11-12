# Architecture

Packages, boundaries, core interfaces, who owns which goroutine, and how a run moves from trigger to terminal state. Refines spec §5 and §27.4.

## 1. Shape in one paragraph

The engine is a **library** (`internal/engine`) that takes a compiled pipeline, a run store, a part host and a clock, and executes runs. `milld` wraps it with a socket server, a trigger host, config reload and an index. `mill run --local` (and the weekend prototype) links the same engine in-process with no socket. Every run is owned by one **run actor** goroutine that is the only writer of that run's event log; every concurrency slot is owned by one **scheduler loop** goroutine. Parts, built-in or external, talk to the engine through the same message set ([protocols.md](protocols.md)); built-ins just skip the JSON.

```mermaid
flowchart TB
  subgraph mill["mill (client)"]
    CLI["surface/cli"] --> WC["wire/client"]
    TUI["surface/tui"] --> WC
    CLI -- "--local" --> LOCAL["engine (in-process)"]
  end
  WC -- "JSONL / UDS" --> SRV
  subgraph milld
    SRV["daemon/server"] --> ENG["engine"]
    TRIG["trigger host"] --> ENG
    GW["gateway (opt)"] --> SRV
    GW --> TRIG
    ENG --> SCHED["scheduler loop"]
    ENG --> ACT["run actors"]
    ACT --> PH["parthost"]
    PH --> BI["builtin parts (in-proc)"]
    PH --> EXT["exec parts (stdio)"]
    PH --> LUA["lua parts"]
    PH --> MCP["mcp servers"]
    BI --> RUN["runners: local · pty · container · ssh"]
    ACT --> STORE["store: events · items · logs"]
    STORE --> CAS["artifacts CAS"]
    STORE -.write-behind.-> IDX["index.db"]
    ACT --> SEC["secrets broker + redactor"]
    ACT --> WS["workspace manager"]
  end
```

## 2. Package layout

Refines §27.4. Changes from the spec's layout, with reasons:

- `engine/` is new and holds the run actor, scheduler, slots, budgets, retries and cache. The spec spread these across `daemon/sched`; they must not depend on the daemon, because `--local` and the prototype use them.
- `store/`, `runner/`, `workspace/`, `secrets/`, `trigger/` move out of `daemon/` for the same reason. `daemon/` keeps only what exists because there's a daemon: server, boot wiring, reload, status file.
- The pipe-expression parser moves from `surface/pipe` to `pipeline/pipe`: it's a compiler front end, and `Compile` intents run it in the daemon.
- `errs/` is new: the §29 error object, codes, classes, explain pages. Every layer imports it.
- `core/` is new: plain data types shared by everything (ids, states, items, refs). No logic beyond validation.

```
cmd/
  mill/                 main: dispatch to surface/cli
  milld/                main: daemon/boot.Run
internal/
  core/                 ids, State enum + transition table, Item, FileRef, WorkspaceRef, Usage, Budget, selectors
  errs/                 Error{Code,Class,Message,Where,Suggestions,Help,Retryable,Cause}, codes.go, classes, explain/*.md (embed), exit-code map
  clock/                Clock interface (real, fake) — every timer goes through it
  paths/                config/data/state/runtime dirs, socket path, MILL_* overrides, safety checks
  mlog/                 slog to file (from rex rexlog)

  wire/jsonl/           envelope + codec (from rex), shared by daemon and parts protocol
  wire/protocol/        daemon intents/events payload structs, error codes, Hello/Snapshot
  wire/client/          dial, Hello, Do(ctx, intent) → reply, event stream
  partproto/            parts protocol message structs, Part/Session/Emitter interfaces, stdio transport, loopback transport, conformance suite

  pipeline/             Spec (decoded YAML with positions), Parse, Lint, InferEdges, Compile → ir
  pipeline/ir/          immutable compiled graph (Graph, Node, Edge, Port, ExprRef)
  pipeline/pipe/        pipe-expression lexer/parser → Spec
  pipeline/yamledit/    surgical YAML edits (node/edge commands, SavePipeline)
  pipeline/layout/      layered DAG layout for render/editor (sidecar read/write)
  expr/                 CEL env, ${{ }} template scanner, helpers, secret handle type, cost limits
  schema/               mill output shorthand → JSON Schema, validator wrapper, error mapping

  engine/               Engine, run actor, reducer (fold events → RunState), readiness, outputs
  engine/sched/         scheduler loop, ready queue, slot pools, priority/aging
  engine/budget/        ledger, limits, warnings
  engine/retry/         policy parsing (exp/linear/fixed), matchers, backoff with jitter
  engine/cache/         key derivation (canonical JSON), lookup/store

  store/                RunStore: run dirs, events.jsonl appender (group commit), item files, logs, run.json snapshot, live markers
  store/cas/            content-addressed artifact store (sha256), materialization, GC
  store/index/          SQLite schema, write-behind indexer, queries, rebuild
  store/lock/           daemon.lock, run.lock (from rex)

  parthost/             registry + resolution order, part pool, health pings, transports wiring, PartCall routing
  parts/builtin/        sh exec check lua map filter reduce sort uniq take skip batch merge split jq template parse
                        if switch loop assert handoff wait pipeline fail noop
                        agent agent.fanout agent.judge agent.review agent.loop agent.extract
                        git.worktree git.commit git.push git.diff gh.pr gh.issue gh.comment
                        http mcp file.* stdin stdout notify (n8n.call, zapier.call later)
  agents/               registry (merge by id), argv builder, headless output parsers, error normalization, usage extraction
  agents/detect/        terminal-agent state detection (regex/idle/hooks), auto_answer
  runner/               Runner interface, selection, per-runner caps
  runner/proc/          process groups and kill per OS (proc_unix.go, proc_windows.go Job Objects)
  runner/local/ runner/pty/ runner/container/ runner/ssh/ (runner/remote, runner/external later)
  workspace/            git worktrees, temp dirs, cleanup policies, snapshot/reset for retries, path safety
  secrets/              Broker, backends (env, keychain, op, vault, agefile), grants
  secrets/redact/       streaming value redactor, argv redaction (from rex)

  trigger/              Host: arm/disarm, concurrency policy, dedupe, persistent trigger state
  trigger/cron/ trigger/poll/ trigger/runevent/ trigger/fswatch/ trigger/webhook/ trigger/githook/ trigger/once/

  config/               layered loading (defaults, settings.lua, init.lua, .mill/mill.lua, env, flags), reload, --why
  config/settings/      settings registry (from rex types), settings.lua writer
  luaapi/               mill.* API (config phase, runtime phase), sandbox profiles, hooks dispatcher, LuaLS types generator
  plugins/              manifest, resolver, installer, lockfile, permissions, pack, import/export (M9+)

  daemon/server/        socket listener, conns, handlers (intent → engine/store calls)
  daemon/boot/          wiring: lock, config, recovery, engine, triggers, server, status file, signals
  daemon/status/        tiny status file for `mill prompt` / completion
  daemonctl/            spawn/find/reach milld (from rex)
  gateway/              HTTP + WebSocket front door (webhooks, remote API) (M7+)

  surface/cli/          subcommands; surface/cli/core output kit (from rex)
  surface/tui/          splash, board, run popup, editor, settings (out of scope here)
  surface/sound/        soundsets, mixer, silent fallback
  surface/mcpserver/    `mill mcp serve`
  testutil/             testenv, golden, fakeclock helpers, fake parts, echo agent harness
sdk/go/                 parts SDK (thin wrapper over partproto) — later ts/, python/
third_party/vt10x/      rex fork
examples/               example pipelines (integration tests)
```

## 3. Dependency direction

Lower layers never import higher ones. Enforced by `depguard` rules plus a `layering_test.go` that runs `go list -deps` and fails on violations (`make layering`).

| Layer | Packages | May import |
|---|---|---|
| 0 base | `core`, `errs`, `clock`, `paths`, `mlog`, `wire/jsonl` | stdlib only (+ each other where acyclic) |
| 1 language | `expr`, `schema`, `pipeline/**`, `partproto`, `wire/protocol` | layer 0, cel-go, yaml, jsonschema |
| 2 services | `store/**`, `secrets/**`, `workspace`, `runner/**`, `agents/**`, `config/**`, `luaapi` | layers 0–1 |
| 3 execution | `engine/**`, `parthost`, `parts/builtin/**`, `trigger/**`, `plugins` | layers 0–2 |
| 4 hosts | `daemon/**`, `gateway`, `surface/mcpserver`, `wire/client`, `daemonctl` | layers 0–3 |
| 5 surface | `surface/cli`, `surface/tui`, `surface/sound` | layers 0–1, `wire/client`, `daemonctl`, `config`; **plus `engine` + `parthost` + `store` only from `surface/cli/local`** (the `--local` entry) |

Rules that matter more than the table:
1. `engine` sees parts only through `partproto.Part`. It never imports `parts/builtin`. `parthost` registers built-ins.
2. `parts/builtin/*` never import `engine` or `store`. They get everything (runner, workspace, artifacts, secrets) through the `OpenRequest` capabilities object. This is what keeps them replaceable by an external part (spec §9: built-ins get no special internals).
3. `surface/tui` never reads run directories directly; it goes through the daemon. `surface/cli` may read run files read-only for `mill runs --store <dir>` and `mill logs` when the daemon is down (spec §17.12).
4. Only `daemon/boot` and `surface/cli/local` construct an `engine.Engine`.

## 4. Core interfaces

Go sketches. Names are the plan; signatures may change during M1 but the ownership rules may not.

### 4.1 Shared data (`core`)

```go
package core

type RunID string      // 32 lowercase hex chars, random
type NodeRunID string  // "<node>@<attempt>" or "<node>[<i>]@<attempt>", unique within a run

type State string // pending queued working retrying done failed skipped cached cancelled crashed

func (s State) Terminal() bool
func CanTransition(from, to State) bool // table in core/states.go; reducer rejects anything else

type Item struct {
	Data  json.RawMessage `json:"data"`
	Files []FileRef       `json:"files,omitempty"`
	WS    *WorkspaceRef   `json:"ws,omitempty"`
	Meta  ItemMeta        `json:"meta"`
}

type ItemMeta struct {
	ID      string    `json:"id"`      // "<node-run>/<port>/<index>", e.g. "plan@1/out/0"
	Run     RunID     `json:"run"`
	Node    string    `json:"node"`
	Port    string    `json:"port"`
	Index   int       `json:"index"`
	Parents []string  `json:"parents,omitempty"`
	TS      time.Time `json:"ts"`
}

type FileRef struct {
	Name   string `json:"name"`
	Path   string `json:"path,omitempty"` // local read-only view; filled at delivery, never stored
	SHA256 string `json:"sha256"`
	Size   int64  `json:"size"`
	MIME   string `json:"mime,omitempty"`
}

type WorkspaceRef struct {
	ID     string `json:"id"`
	Kind   string `json:"kind"` // git | dir | volume
	Path   string `json:"path"`
	Repo   string `json:"repo,omitempty"`
	Branch string `json:"branch,omitempty"`
	Base   string `json:"base,omitempty"` // commit sha at creation
}

type Usage struct {
	TokensIn, TokensOut, TokensCache int64
	CostMicroUSD                     int64 // integers only; floats never touch money
	Model                            string
}
```

### 4.2 Parts (`partproto`)

One interface for every node, trigger, sink and runner part. Built-ins implement it in Go; the stdio transport implements it by speaking JSONL to a process; the Lua host implements it with a Lua state.

```go
package partproto

type Part interface {
	Describe(ctx context.Context) (Manifest, error)
	Open(ctx context.Context, req OpenRequest, out Emitter) (Session, error)
}

// Session is one context (node-run, trigger arm, sink attach).
type Session interface {
	Item(ctx context.Context, port string, it core.Item) error // host → part input; may block (backpressure)
	EndOfInput(ctx context.Context) error
	Cancel(reason string)                                       // non-blocking; part must Fail(cancelled) within grace
	Resume(ctx context.Context, checkpoint json.RawMessage) error
	Close() error
}

// Emitter is part → host. Safe for concurrent use. Item blocks when the host applies backpressure.
type Emitter interface {
	Item(port string, it core.Item) error
	Log(level slog.Level, msg string, attrs ...slog.Attr)
	Progress(pct float64, msg string)
	Usage(u core.Usage)
	Artifact(name string, r io.Reader, mime string) (core.FileRef, error)
	Checkpoint(data json.RawMessage)
	Done(d DoneInfo)
	Fail(f Failure)
}

type OpenRequest struct {
	Context   ContextInfo        // run, node, node-run, attempt, item index, dry_run
	Capability string            // e.g. "node:sh"
	With      json.RawMessage    // resolved config (expressions evaluated; secret fields are handles)
	Workspace *core.WorkspaceRef
	Secrets   SecretGrant        // only declared + referenced names; Reveal(name) for in-proc parts
	Budget    core.BudgetRemaining
	Scratch   string             // per node-run dir, deleted at node-run end unless kept
	Host      HostServices       // in-proc only; nil fields over stdio
}

// HostServices is what built-ins get instead of importing engine/store.
// Over stdio an external part has the same powers through messages (Artifact, Log, …) or not at all.
type HostServices struct {
	Runner    Runner          // where to start processes for this node-run
	Workspace WorkspaceOps    // create worktree, snapshot, diff
	Agents    AgentRegistry   // read-only registry view
	Expr      Evaluator       // per-item `with` evaluation
	SubRuns   SubRunStarter   // for `pipeline` and `loop`
	Clock     clock.Clock
}

type Failure struct {
	Class     errs.Class
	Code      string        // "M6201" or plugin code "linear/E_TEAM_NOT_FOUND"
	Message   string
	Retryable bool
	RetryAfter time.Duration
	ExitCode  *int
	StderrTail string
}
```

Helpers so a built-in is a handful of lines:

```go
func Batch(fn func(ctx context.Context, c *NodeCtx, in []core.Item) ([]Out, error)) Part
func Each(fn func(ctx context.Context, c *NodeCtx, it core.Item) ([]Out, error)) Part
func Stream(fn func(ctx context.Context, c *NodeCtx, in <-chan core.Item, out Emitter) error) Part
```

### 4.3 Runners (`runner`)

```go
package runner

type Runner interface {
	ID() string
	Caps() Caps // pty, network control, volume mounts, remote paths
	Start(ctx context.Context, spec ProcSpec) (Proc, error)
}

type ProcSpec struct {
	Argv      []string          // never contains secret values (lint + runtime check)
	Dir       string
	Env       []string          // secrets arrive here, by name the node declared
	Stdin     io.Reader         // nil = /dev/null
	PTY       *PTYSize          // non-nil → terminal mode
	Workspace *core.WorkspaceRef
	Limits    Limits            // cpu, mem, pids (container); wall handled by engine
	Network   string            // "", "none", "default"
	Labels    map[string]string // run/node ids for ps, container labels
}

type Proc interface {
	PID() int
	Stdin() io.WriteCloser
	Stdout() io.Reader          // PTY: merged screen bytes
	Stderr() io.Reader          // nil in PTY mode
	Screen() Screen             // PTY only: vt10x mirror for detection and repaint
	Resize(cols, rows uint16) error
	Interrupt() error           // SIGTERM to group / CTRL_BREAK
	Kill() error                // SIGKILL to group / TerminateJobObject; idempotent
	Wait() (ExitStatus, error)
	Identity() ProcIdentity     // pgid + start time, recorded before Start returns for crash recovery
}
```

### 4.4 Triggers, secrets, sinks, artifacts

```go
package trigger

type Trigger interface {
	// Arm blocks until ctx ends, calling fire for each event. State is persisted by the host
	// (last fire time, seen keys) and handed back on re-arm.
	Arm(ctx context.Context, spec Spec, state State, fire func(Fire) error) error
}

type Fire struct {
	Event     json.RawMessage
	DedupeKey string
	At        time.Time
	Reply     chan<- WebhookReply // webhook "wait" mode only
}
```

```go
package secrets

type Backend interface {
	Name() string
	Get(ctx context.Context, name, ref string) (Value, error) // errs M9202 when missing
}
type Setter interface{ Set(ctx context.Context, name string, v []byte) error } // optional
type Value struct{ b []byte }                                                  // String() == "•••"; no JSON marshal
func (v Value) Reveal() []byte                                                 // only Broker and grants call it
```

```go
package engine

// Sink receives run events after they're durable. Delivery is async through a bounded queue;
// a full queue drops and counts (spec §2.10: optional never blocks core).
type Sink interface {
	Name() string
	Deliver(ctx context.Context, ev store.Event) error
}
```

```go
package cas

type Store interface {
	Put(ctx context.Context, r io.Reader, name, mime string) (core.FileRef, error) // stream-hash, dedupe, atomic
	Open(ctx context.Context, sha string) (io.ReadCloser, error)
	Materialize(ctx context.Context, ref core.FileRef, dir string) (string, error) // read-only hardlink or copy
	Stat(sha string) (int64, bool)
	Remove(sha string) error
	Walk(fn func(sha string, size int64, mtime time.Time) error) error
}
```

### 4.5 Run store and index

```go
package store

type RunStore interface {
	Create(ctx context.Context, h RunHeader) (Run, error) // allocates number, dir, run.lock, live marker
	Open(id core.RunID, mode OpenMode) (Run, error)       // ReadOnly for clients, Owner for the actor/recovery
	Live() ([]core.RunID, error)                          // from live/ markers
	Dir() string
}

type Run interface {
	Append(ev Event, durability Durability) (seq uint64, err error) // owner only; Sync waits for fsync
	Items(nr core.NodeRunID, port string) ItemFile                  // append (owner) / read+tail (any)
	Log(nr core.NodeRunID, stream string) io.WriteCloser            // redacting writer, owner only
	Events(fromSeq uint64) iter.Seq2[Event, error]
	Tail(ctx context.Context, fromSeq uint64) <-chan Event          // for Subscribe replay+follow
	WriteSnapshot(s RunSnapshot) error                              // run.json, atomic rename
	Finish() error                                                  // remove live marker, release run.lock
}

type Index interface {
	Upsert(ctx context.Context, batch []IndexOp) error
	Runs(ctx context.Context, q RunQuery) ([]RunRow, string, error)
	Resolve(ctx context.Context, sel core.Selector) ([]RunRow, error)
	NextNumber(ctx context.Context, pipeline string) (int, error) // BEGIN IMMEDIATE
	Stats(ctx context.Context, q StatsQuery) (StatsResult, error)
	Rebuild(ctx context.Context, rs RunStore, progress func(n, total int)) error
}
```

### 4.6 Engine and scheduler

```go
package engine

type Engine struct{ /* … */ }

func New(cfg Config) (*Engine, error) // Store, Index, PartHost, Secrets, Workspaces, Runners, Clock, Settings
func (e *Engine) Start(ctx context.Context, req StartRequest) (RunRef, error)
func (e *Engine) Cancel(id core.RunID, reason string) error
func (e *Engine) Delete(ctx context.Context, id core.RunID) error
func (e *Engine) Recover(ctx context.Context) (RecoveryReport, error)
func (e *Engine) Subscribe(f Filter) (<-chan Update, func()) // live updates for server/CLI
func (e *Engine) Feed(id core.RunID, items []core.Item, end bool) error // streaming stdin input
func (e *Engine) Shutdown(ctx context.Context) error // stop granting slots; live node-runs handled per execution.md §8

package sched

type Request struct {
	Run      core.RunID
	NodeRun  core.NodeRunID
	Priority int
	Order    int        // pipeline topological index
	Since    time.Time  // age
	Needs    SlotSet    // global node, agent, runner:<id>, run-agents:<run>, node-concurrency:<run>/<node>
	Grant    chan<- Grant
}

type Scheduler interface {
	Enqueue(r Request)
	Withdraw(run core.RunID, nr core.NodeRunID) // cancel before grant
	Release(g Grant)                             // exactly once per Grant (invariant 7)
	Resize(slot string, max int)                 // settings reload
	Snapshot() Stats                             // queue depth for metrics/TUI
}
```

## 5. Goroutines and ownership

| Goroutine | Count | Owns (only writer) | Talks to |
|---|---|---|---|
| `daemon/boot` main | 1 | process lifecycle, signal handling | everything at start/stop |
| server accept loop | 1 | listener | spawns conn goroutines |
| conn reader | 1 per client | that connection's subscriptions | engine, store (read), index (read) |
| conn writer | 1 per client | the socket write side (rex `clientConn`) | bounded queue; slow client dropped |
| **scheduler loop** | 1 | ready queue, all slot counters | run actors via `Grant` channels |
| **run actor** | 1 per live run | `RunState`, the run's `events.jsonl`, `run.json`, item file appends | scheduler, node-run workers, store, bus |
| node-run worker | 1 per working node-run | its part `Session`, its process | emits messages to its run actor's inbox |
| part I/O pumps | 2–3 per process (stdout reader, stdin writer, stderr pump) | pipe ends | node-run worker |
| part pool | 1 per long-lived part process | demux by `ctx` | node-run workers |
| indexer | 1 | `index.db` writes | fed by run actors through a bounded channel |
| event bus | 0 (fan-out under lock, non-blocking enqueue) | subscriber list | conn writers, sinks, Lua dispatcher |
| trigger arm | 1 per armed trigger | that trigger's runtime | trigger coordinator |
| trigger coordinator | 1 | concurrency policy state, dedupe windows, trigger state files | engine.Start/Cancel |
| Lua hook dispatcher | 1 | hook `LState` (rex dispatcher pattern) | engine via `mill.start` etc. |
| sink worker | 1 per sink | its queue | external systems |
| config watcher | 1 | reload debounce | config, parthost, triggers |
| status writer | 1 | status file | bus |
| pipeline watcher | 1 | pipeline catalog | bus (`PipelineChanged`), triggers (re-arm) |

Rules:
1. **One writer per run.** Only the run actor appends to `events.jsonl`. Node-run workers send messages (`partMsg`, `procExited`, `usage`) to the actor's inbox; the actor validates, redacts, persists, folds, then broadcasts. Broadcasting happens after the append returns, so clients never see an event that a crash could lose (for events written with `Sync` durability).
2. **One owner of slots.** The scheduler loop grants and releases. A run actor releases a grant only by sending `Release` when it has folded a terminal state for that node-run; teardown is the only path to a terminal state, so the release can't be skipped (invariant 7). A test asserts all pools are at zero after every engine test (`sched.AssertDrained`).
3. **No locks across components.** Components communicate by channels; the few mutex-protected structures (bus subscriber list, part registry snapshot, settings snapshot) are copy-on-write and never held while calling out.
4. **Blocking budget.** A run actor never blocks on I/O other than its own local disk appends. Slow things (process spawn, git worktree creation, secret resolution through `op`) run in node-run workers or short-lived helper goroutines that report back.
5. **Context tree.** `daemon ctx → engine ctx → run ctx → node-run ctx → process`. Cancel uses explicit teardown messages, not just ctx cancellation, so the grace period and drain happen (invariant 8). ctx cancellation is the last resort at shutdown.
6. **goleak clean.** Every component has a `Close`/`Shutdown` that returns only after its goroutines exit. Tests run with goleak via `testenv.RunLeakChecked`.

## 6. Lifecycle of a run

### 6.1 States

Run: `queued → working → done | failed | cancelled | crashed`. Node-run: see [engine.md](engine.md) §4 for the full table. Transitions are validated by `core.CanTransition` inside the reducer; an illegal transition is an `M0xxx` internal error and the event is not written.

### 6.2 From `mill run` to result (daemon mode)

```mermaid
sequenceDiagram
  autonumber
  participant C as mill (CLI)
  participant S as milld server
  participant E as engine
  participant A as run actor
  participant Q as scheduler loop
  participant W as node-run worker
  participant P as part (sh)
  participant R as runner (local)
  participant D as store
  C->>S: Hello
  S-->>C: Snapshot
  C->>S: StartRun{pipeline, inputs, wait} id=7
  S->>E: Start(req)
  E->>E: load pipeline @hash, compile (cached), resolve secrets (fail fast M9202)
  E->>D: Create run (number, dir, run.lock, live marker)
  E->>A: spawn actor(graph, header)
  A->>D: Append RunCreated (sync)
  E-->>S: RunRef{id, number}
  S-->>C: Ack{run_id, number} id=7
  S-->>C: RunAdded (broadcast)
  C->>S: Subscribe{run_id, replay:true}
  A->>A: readiness: source nodes ready
  A->>Q: Enqueue(plan@1, needs{node, agent, runner:local})
  Q-->>A: Grant
  A->>D: Append NodeRunStateChanged(queued→working) + NodeRunStarting (sync, before spawn)
  A->>W: start worker(grant, open request)
  W->>P: Open(with, workspace, secrets)
  P->>R: Start(argv, env, dir)
  R-->>P: Proc (pgid recorded)
  W->>A: procStarted{pid, pgid, start time}
  A->>D: Append NodeRunStarted
  P-->>W: Item(out) … Usage … Log
  W->>A: items/usage/log msgs
  A->>D: append items file → ItemsCommitted event (redacted)
  P-->>W: Done
  W->>A: finished{done, exit 0}
  A->>D: Append NodeRunFinished (sync)
  A->>Q: Release(grant)
  A->>A: next ready nodes… until no node can progress
  A->>A: evaluate outputs
  A->>D: Append RunFinished (sync), write run.json, remove live marker
  A-->>S: RunUpdated / RunFinished (bus)
  S-->>C: RunUpdated{state:done, outputs}
  C->>C: print outputs JSONL on stdout, exit 0
```

### 6.3 Trigger to run

```mermaid
sequenceDiagram
  participant T as cron trigger
  participant K as trigger coordinator
  participant E as engine
  participant F as trigger state file
  T->>K: fire{at, dedupe_key}
  K->>K: dedupe window check, concurrency policy (queue/skip/replace/parallel N)
  alt skip (one already running)
    K->>F: record skipped fire
  else replace
    K->>E: Cancel(running, "replaced")
    K->>E: Start(pipeline, event)
  else queue / parallel
    K->>E: Start(pipeline, event, trigger ref)
  end
  K->>F: persist last_fire, seen keys (fsync)
  E-->>K: run id → TriggerFired event on bus
```

`queue` policy holds pending fires in the coordinator (persisted), not in the engine, so a long queue never creates `queued` runs that eat run slots and so a restart keeps the queue.

### 6.4 Cancel and teardown

```mermaid
sequenceDiagram
  participant C as client
  participant A as run actor
  participant W as worker
  participant P as part/process
  participant Q as scheduler
  C->>A: CancelRun(reason)
  A->>A: Append RunCancelRequested; stop enqueuing; Withdraw queued requests
  loop each working node-run
    A->>W: teardown(reason)
    W->>P: Cancel (protocol) + Interrupt (SIGTERM group / CTRL_BREAK)
    alt exits within grace (default 10s)
      P-->>W: Fail{class: cancelled} or exit
    else
      W->>P: Kill (SIGKILL group / TerminateJobObject)
    end
    W->>W: drain stdout/stderr ≤ 2s or 1 MiB
    W->>A: finished{cancelled}
    A->>A: Append NodeRunFinished(cancelled) — never failed
    A->>Q: Release
  end
  A->>A: workspace cleanup per policy
  A->>A: Append RunFinished(cancelled, reason)
```

### 6.5 Daemon restart

```mermaid
sequenceDiagram
  participant B as boot
  participant L as daemon.lock
  participant S as store
  participant E as engine
  B->>L: acquire (fail M7xxx if held)
  B->>S: Live() → run ids
  loop each live run
    B->>S: try run.lock (skip if held by a --local process)
    B->>S: replay events → RunState (truncate torn tail)
    B->>E: Recover(run)
    E->>E: per node-run: queued/pending → requeue; working → kill orphan group if same identity, then resumable→Resume, idempotent→new attempt, else crashed (M4301)
  end
  B->>B: start server, arm triggers (missed cron per catch_up)
```

## 7. Process model details

| Process | Started by | Lifetime | Group / job |
|---|---|---|---|
| `milld` | `daemonctl.Start` (detached: setsid / new process group + hidden console), or by the user, or launchd/systemd unit (later) | until `Shutdown` or signal | own session |
| step process (sh, exec, check, headless agent) | runner/local | node-run | own process group (Unix `Setpgid`), own Job Object (Windows) |
| terminal agent | runner/pty | node-run | own session (creack/pty uses `Setsid`), Job Object on Windows |
| executable part (per-call) | parthost | one context | own process group |
| executable part (long-lived) | parthost | until idle timeout (default 5 min) or failed `Ping` | own process group |
| MCP server | parthost | long-lived, restarted on failure with backoff | own process group |
| container | runner/container via `docker`/`podman` CLI | node-run | `--rm`, labeled `mill.run=<id>` so recovery can find and kill it |

Only `milld` (or the `--local` process) ever spawns children. The `mill` client spawns only `milld` and, for `mill edit`, `$EDITOR`.

## 8. Configuration flow

1. `config.Load(layers)` evaluates Lua layers in order (spec §19.2) into a `config.Snapshot` (settings, agents, MCP servers, runners, secret backends, plugin specs, hooks as Lua function refs).
2. The snapshot is immutable. Reload builds a new one; on error the old one stays and the error is published (`config_error` on the bus, M8xxx).
3. Consumers subscribe and apply: scheduler resizes pools, parthost diffs MCP/runner definitions, trigger host re-arms only changed triggers. Running runs keep the snapshot they started with for agent registry and runner definitions (invariant 10 extended to config): the run header records the config snapshot hash.
4. The client loads the same layers for TUI settings, keymaps and themes, with runtime-only API calls (`mill.on`, `mill.command` bodies) recorded but not executed.
