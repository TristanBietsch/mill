# Error handling across layers

Status: accepted with the 2026-11-12 refinement. Spec: [§29](../../spec/29-errors.md#29-errors-and-diagnostics), [§17.2](../../spec/17-cli.md#172-exit-codes).

Where errors are created, wrapped, translated and rendered in each layer ([layers.md](layers.md)), so the §29 promise (one shape everywhere, a stable code, a next command) holds from a failing `fsync` to a red line in the TUI.

## 1. One type, created once

`errs.Error` (L0) is the §29.3 object as a Go type:

```go
package errs

type Error struct {
	Code        string   // "M3012"; plugin codes live in Cause ("linear/E_TEAM_NOT_FOUND")
	Class       Class    // user, not_found, ambiguous, conflict, refused, unreachable, transient,
	                     // rate_limited, auth, timeout, budget, cancelled, agent, internal
	Message     string   // headline: lowercase, no period, < 100 chars (spec §29.1 rule 5)
	Where       Where    // file/line/col, pipeline, node, run, part, setting
	Suggestions []string // did-you-mean
	Help        []string // next commands
	Retryable   bool
	RetryAfter  time.Duration
	Cause       error    // wrapped; rendered up to three levels
}

func New(code string, class Class, format string, args ...any) *Error // code must exist in codes.go
func Wrap(err error, code string, class Class, format string, args ...any) *Error
func As(err error) (*Error, bool)            // errors.As through any %w chain
func ExitCode(err error, state core.State)    // in surface/cli/core: class + run state + reason → §17.2
```

Rules [LG ch. 9]:

1. **Create where the code is known.** The layer that understands what went wrong creates the `*errs.Error` with its code. Lower layers return plain Go errors (wrapped with `%w`) only when they can't know the user-level meaning.
2. **Wrap, don't stringify.** `fmt.Errorf("open run %s: %w", id, err)` keeps the chain; `errors.Is`/`errors.As` find causes. No code ever matches on message text [BSRS ch. 6].
3. **Translate at boundaries only.** Each layer boundary that crosses a process, a vendor or a protocol has one translation function (§3). Everywhere else errors pass through unchanged.
4. **Unknown becomes internal.** A plain error that reaches a boundary without an `*errs.Error` in its chain becomes `M0001 internal error: this is a bug in mill` with the original as cause and `help: mill doctor --bundle`. A test per boundary asserts it.
5. **Redact before leaving the process.** Rendering (CLI, TUI, wire, logs, event payloads) passes the message, `Where` values and the cause chain through the run's or daemon's redactor (invariant 9, spec §29.1 rule 8).
6. **Class drives behavior, code drives docs.** Retries, exit codes and TUI color key off `Class`; `mill explain` keys off `Code` (spec §29.6).
7. **Fail noisily and early** [TAOUP ch. 1, Rule of Repair]: compile, lint and secret resolution run before any node-run, so most user errors surface before work starts [RI ch. 5, Fail Fast].

## 2. Codes by layer

A code range belongs to the layers that may create it. `make errors-check` enforces it by scanning `errs.New`/`errs.Wrap` call sites (with the package's layer from [layers.md](layers.md) §8.1).

| Range | Area (spec §29.4) | Created in | Typical creator |
|---|---|---|---|
| `M0xxx` | internal | any layer | boundary translators (rule 4); reducer on illegal transition |
| `M1xxx` | CLI, selectors, arguments | L7, L3 (`store/index` selector resolution), L6 (selector in the daemon) | `surface/cli`, `index.Resolve` |
| `M2xxx` | files and pipelines | L2 (parse), L3 (disk: M2002), L5 (`drafter`: M2212), L6 (`SavePipeline` conflict) | `pipeline.Parse`, `store` |
| `M3xxx` | lint and expressions | L2 | `pipeline.Lint`, `expr` |
| `M4xxx` | runs and scheduling | L4 (budgets, timeouts, recovery), L5 (`trigger` arm: M4401) | `engine`, `engine/budget` |
| `M5xxx` | parts and plugins | L5 | `parthost`, `plugins` |
| `M6xxx` | agents | L5 | `agents` (error rules), `parts/builtin/agent` |
| `M7xxx` | daemon and protocol | L0 (`paths`: M7002; `wire/jsonl`: M7101), L5 (`daemonctl`, `wire/client`), L6 | `daemonctl.Ensure`, `server` |
| `M8xxx` | Lua config | L5 | `config`, `luaapi` |
| `M9xxx` | security, trust, secrets | L2 (lint: M9201, M9204, M9207), L4 (`secrets`, `workspace` path rules), L5 (`parthost` trust, `plugins`) | `secrets.Broker`, `paths.CheckRoot` |

Codes this refinement reserves (explain pages and goldens land with the code that uses them):

| Code | Class | Headline | Where |
|---|---|---|---|
| `M2002` | `transient` | `not enough disk space in <dir> (<free> free)` · `help: mill prune --older 30d · mill doctor` | `store` on ENOSPC, `mill doctor` low-space check ([data-lifecycle.md](data-lifecycle.md) §6) |
| `M4108` | `transient` | `run queue is full (<n> queued runs)` · `help: wait, or raise max_queued_runs` | `engine.Start` when the queued-run cap is hit ([resilience.md](resilience.md) §4) |
| `M7007` | `refused` | `run <id> was written by a newer mill (log_v <n>)` · `help: upgrade mill, then mill daemon restart` | `core/event` decoder, surfaced by recovery and `GetRun` ([versioning.md](versioning.md) §3) |

## 3. Translation points

| # | Boundary | Layer | Input | Output | Function |
|---|---|---|---|---|---|
| T1 | OS → mill | L0 | `*fs.PathError`, `syscall.Errno` | `*errs.Error` with class (`ENOENT` → `not_found`, `EACCES` → `refused`, `ENOSPC` → `transient` M2002) | `errs.FromOS` |
| T2 | YAML/CEL → diagnostics | L2 | yaml/v4 and cel-go errors | positioned M2101/M31xx with source line and caret | `pipeline.diag`, `expr.diag` |
| T3 | Agent CLI → class | L5 | exit code + last 64 KiB of output | class, code, `retry_after`, help from registry `errors` rules; no match + non-zero → M6300 | `agents.Normalize` [DDD ch. 14, Anticorruption Layer] |
| T4 | Part → host | L5 | `Fail{class, code, message, retryable, stderr_tail}` or a protocol violation | M5201 with the part's own code as cause (`linear/E_TEAM_NOT_FOUND`); violations M5202 | `parthost.failure` |
| T5 | Node-run → event | L4 | the `*errs.Error` from T3/T4 or the engine itself (timeout M4201, budget M4101) | redacted §29.3 object inside `NodeRunFinished.error` and `RunFinished.error` | `engine.recordFailure` |
| T6 | Engine → wire | L6 | `*errs.Error` | `Error{code: <protocol code>, error: <§29.3 object>}`; protocol code from class (`not_found` → `not_found`, `user` → `bad_request`, `budget` → `budget_exceeded` …) | `server.replyError` |
| T7 | Wire → client | L5 | `Error` event | `*errs.Error` rebuilt from `data.error` (the client never invents codes for daemon errors) | `client.decodeError` |
| T8 | Error → person or script | L7 | `*errs.Error`, run state, reason | human text (stderr) or JSON (stdout with `--json`), exit code | `cli/core.Render`, `errs.ExitCode` |

A part's or agent's raw text never becomes a headline. It goes in `Cause` or `stderr_tail`, so the headline stays under 100 characters and in mill's words (spec §29.1 rules 5 and 7).

## 4. Errors inside concurrent code

- A node-run worker never returns an error to nobody: it sends `finished{err}` to its run actor's inbox, which records it (T5). Workers don't log-and-drop [CIG ch. 5, Error Propagation].
- Panics in a worker, part I/O pump or Lua call are recovered at the goroutine boundary, converted to M0001 with the stack in `milld.log` (never in the event payload), and the node-run fails `internal`. A panic in the run actor or scheduler loop is not recovered: the daemon exits and recovery replays (a corrupt in-memory state must not keep running) [RI ch. 5, Let It Crash].
- `context.Canceled` is never reported as a failure. Teardown records `cancelled` (or `timeout` for a node deadline, class `timeout`, M4201), per invariant 8.

## 5. Tests

| Test | Guards |
|---|---|
| Golden per M-code: human and `--json` | spec §29.11 |
| `errors-check`: code ↔ explain page ↔ golden ↔ allowed layer | §2 |
| Boundary tests T1–T8: a plain `errors.New` injected below each boundary comes out as M0001 with cause | rule 4 |
| `TestErrors_Redacted`: a fake secret inside a cause chain never appears in CLI, JSON, wire or log output | rule 5, invariant 9 |
| Agent rule tables per built-in registry entry, with recorded outputs | T3 |
| `FuzzPartMessages` (Fail after Done, double Fail) | T4 |

## 6. Sources

| Decision | Follows |
|---|---|
| Wrap with `%w`, inspect with `errors.As`, typed error | [LG ch. 9] |
| Never match on message text; small explicit translators | [BSRS ch. 6] |
| Fail early and noisily | [TAOUP ch. 1] [RI ch. 5, Fail Fast] |
| Vendor errors normalized at the edge | [DDD ch. 14, Anticorruption Layer] |
| Errors travel to the owner goroutine, never dropped | [CIG ch. 5] |
| Panic in the actor ends the daemon; recovery replays | [RI ch. 5, Let It Crash] [BSRS ch. 9] |
| Error codes and JSON shapes are contracts | [SEG ch. 1, Hyrum's Law] |
