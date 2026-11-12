# M0 Weekend Prototype Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single `mill run <file.mill.yaml>` (no daemon) that parses a pipeline, infers edges, runs `sh`, headless `agent` and `stdout` nodes in topological order with JSONL items between them, records the run on disk, and is used nightly for the `nightly-vuln-sweep-lite` dogfood pipeline.

**Architecture:** The engine is a library from day one (backend plan, decision 1): `cmd/mill` → `internal/surface/cli` → `internal/engine`, which compiles with `internal/pipeline` + `internal/expr` and executes parts from `internal/parts`. M0 runs nodes **sequentially**, `batch` mode only. Package names and paths follow [architecture.md §2](architecture.md#2-package-layout) so M1 extends rather than replaces them; M0's `parts.Part` interface is a deliberate in-process stand-in for `partproto` (M1).

**Tech Stack:** Go 1.27 (`toolchain go1.27.4`), `github.com/google/cel-go` v0.30.0, `go.yaml.in/yaml/v4` v4.0.0, `github.com/rogpeppe/go-internal/testscript` for CLI tests, `github.com/stretchr/testify`. Lint: golangci-lint v2.14.0 via `go run` (as rex). Vuln: `golang.org/x/vuln/cmd/govulncheck@v1.8.0` via `go run`.

**Spec:** [milestones.md § M0](milestones.md#m0--weekend-prototype); spec [§6](../../spec/06-data-model.md), [§7](../../spec/07-pipeline-format.md), [§8](../../spec/08-expressions.md), [§11](../../spec/11-agents.md), [§14](../../spec/14-piping.md), [§17.2](../../spec/17-cli.md#172-exit-codes), [§29](../../spec/29-errors.md); [engine.md §1–3](engine.md), [storage.md §1, §3](storage.md), [execution.md §5](execution.md).

## Global Constraints

- Module path `github.com/TristanBietsch/mill`; `go 1.27` with `toolchain go1.27.4`; `CGO_ENABLED=0` builds.
- Pipeline file subset accepted in M0: top-level `mill` (must be `1`), `name`, `description`, `inputs`, `nodes`, `outputs`; node keys `uses`, `with`, `in`, `io`. Any other key → `M3001` error naming the key and its line.
- Parts in M0: `sh`, `agent`, `stdout`. Anything else in `uses` → `M5101`.
- Agents in M0: `claude` and `echo` registry entries hard-coded in Go. `codex` moves to M1 (its headless output needs the JSONL parser M1 brings).
- Edge rule (ADR-028 / backend D9): every `nodes.X` reference is an order edge; data input = `in:` if given, else the **primary upstream** (the referenced node every other referenced node is upstream of), else `M3021 ambiguous input`. A node with no references and no `in` is a source; only the **first** source in file order receives `--stdin` items.
- Default `io` for `sh` in M0 is `plain` (one JSON value per line both ways); `text` = raw lines in, each output line becomes a JSON string item. `envelope` is M1.
- Step environment allowlist (ADR-031): `PATH HOME USER LOGNAME SHELL TMPDIR LANG LC_ALL LC_CTYPE TERM TZ` plus `MILL_RUN_ID`, `MILL_NODE_ID`. Nothing else is inherited.
- Run store root: `$MILL_DATA_DIR`, else `$XDG_DATA_HOME/mill`, else `~/.local/share/mill`; `--store <dir>` overrides. Layout `runs/<run-id>/{events.jsonl, run.json, inputs.json, items/<node>@1/out.jsonl, logs/<node>@1.out.log, logs/<node>@1.err.log}`. Dirs `0700`, files `0600`.
- Events: one JSON object per line `{"seq":N,"ts":"<RFC3339Nano UTC>","type":T,"nr":"<node>@1"?,"data":{…}}`; M0 types only `RunCreated` (data carries `"log_v":1`, `run`, `pipeline`, `bundle_hash`, `local:true`), `NodeRunStarted`, `NodeRunFinished` (`state`, `exit_code`, `duration_ms`, `items_out`), `RunFinished` (`state`, `reason`?, `duration_ms`).
- Run id: 32 lowercase hex chars from `crypto/rand`.
- Errors follow §29 shape: `error[MXXXX]: <message>` then `   ┌─ <file>:<line>:<col>` when known, then `  help: <next command>`. Codes used in M0: `M1201` file not found, `M1301` unknown flag, `M1306` unknown command, `M1309` required input missing (new, free), `M2101` YAML parse, `M3001` unknown key, `M3002` required key missing, `M3012` unknown node in expression, `M3013` unknown input in expression, `M3020` cycle, `M3021` ambiguous input, `M3101` expression error, `M5101` unknown part, `M5201` part failed, `M6101` agent CLI not on PATH, `M6300` agent failed.
- Exit codes (§17.2): `0` done, `1` run failed or cancelled, `5` invalid arguments or compile/lint failure.
- stdout carries only results; progress, warnings and errors go to stderr (§14.1).
- Output contract: without `--json`, stdout gets `stdout`-node lines during the run, then one line: the `outputs` object (`{"greeting":"…"}`) if the pipeline declares outputs. With `--json`, the final line is the run record `{"run":"<id>","pipeline":"<name>","state":"done","outputs":{…},"dir":"<run dir>"}`. (This corrects milestones.md exit criterion 1: `mill run hello.mill.yaml | jq .greeting`, `mill run hello.mill.yaml --json | jq .outputs.greeting`.)
- Conventional Commits, signed commits never bypassed, one branch `feat/M0-weekend-prototype` in worktree `../mill-M0`, one PR with an independent review (CONTRIBUTING).

## Review Focus

1. **Non-JSON line from an `io: plain` step** (e.g. a tool prints a banner): the node fails with `M5201` naming the node, the 1-based line number and the first 80 bytes of the line, and `help:` pointing at `io: text`. Never silently dropped. Test in Task 6.
2. **Last output line without a trailing newline** (common with `printf`): still becomes an item. Test in Task 6.
3. **Agent CLI missing or failing** (`claude` not on PATH under cron's minimal PATH, or not signed in): `M6101 claude not found on PATH` with `help:` naming the PATH line for crontab; a non-zero agent exit → `M6300` with the last 20 stderr lines. Run dir records `failed`; exit 1. Test in Task 7.
4. **Ctrl-C / SIGTERM mid-run** (or cron killing the job): the child's whole process group is killed, `NodeRunFinished{state:"cancelled"}` and `RunFinished{state:"cancelled"}` are written, exit 1, no orphan `sleep` left. Test in Task 9.
5. **`~` in paths from inputs** (`repo: ~/code/app`): `sh`'s `dir` expands a leading `~/` to `$HOME/`; a missing dir fails with `M5201 dir "…" does not exist`, not a shell error. Test in Task 6.

---

## File Structure

| Path | Responsibility |
|---|---|
| `go.mod`, `Makefile`, `.golangci.yml`, `.github/workflows/ci.yml` | Module, gate (`make check`), lint config and CI copied from rex and trimmed (header comment names the rex commit) |
| `cmd/mill/main.go` | `os.Exit(cli.Main(os.Args[1:], os.Stdin, os.Stdout, os.Stderr))` |
| `internal/core/{ids,item,state}.go` | `RunID`, `Item`, `ItemMeta`, `FileRef`, `WorkspaceRef`, `State` (architecture §4.1) |
| `internal/errs/errs.go` | `Error` type, `Format`, exit-code mapping |
| `internal/paths/paths.go` | `DataDir()` |
| `internal/expr/{scan,env,template}.go` | `${{ }}` scanner, CEL env + helpers, compiled templates, reference extraction |
| `internal/pipeline/{spec,parse,compile}.go` | YAML → `Spec` with positions → `Graph` (validation, edge inference, topo order) |
| `internal/store/rundir.go` | Run directory writer: events, items, logs, `run.json` |
| `internal/parts/{part,registry}.go` | `Part` interface, `Call`, `Result`, `Lookup` |
| `internal/parts/builtin/sh/sh.go`, `.../stdout/stdout.go`, `.../agent/agent.go` | The three M0 parts |
| `internal/agents/{entry,builtin,build,output}.go` | Agent registry (claude, echo), argv building, headless output parsing |
| `internal/testutil/echoagent/main.go` | `mill-echo-agent` test helper binary |
| `internal/engine/engine.go` | `Run`: sequential executor, vars, outputs, cancellation |
| `internal/surface/cli/{main,run}.go` + `testdata/script/*.txtar` | `mill run`, flags, output modes, exit codes |
| `examples/{hello,pipe-demo,nightly-vuln-sweep-lite}.mill.yaml`, `examples/README.md` | Example pipelines, crontab recipe |
| `docs/plans/backend/m0-findings.md` | What dogfooding taught (exit criterion 4) |

---

### Task 1: Module and gate

**Files:**
- Create: `go.mod`, `Makefile`, `.golangci.yml`, `.github/workflows/ci.yml`, `cmd/mill/main.go`, `internal/surface/cli/main.go`, `internal/surface/cli/main_test.go`

**Interfaces:**
- Produces: `func Main(args []string, stdin io.Reader, stdout, stderr io.Writer) int` in package `cli`; `var Version = "0.0.0-dev"` (set by `-ldflags`).

- [ ] **Step 1: Write the failing test** `TestMainVersion`: `Main([]string{"--version"}, …)` returns `0` and stdout is `mill 0.0.0-dev\n`; `TestMainUnknownCommand`: `Main([]string{"nope"}, …)` returns `5` and stderr starts with `error[M1306]: unknown command "nope"`.
- [ ] **Step 2: Run** `go test ./internal/surface/cli/` → FAIL (package missing).
- [ ] **Step 3: Implement** `go.mod` (module path, `go 1.27`, `toolchain go1.27.4`), `cmd/mill/main.go`, and `Main` dispatching on `args[0]` (`--version`, `run` stub returning 5 with "not implemented").
- [ ] **Step 4: Gate.** Copy rex's `Makefile` targets `test test-race vet lint vuln check` (drop `lint-windows`, `cover-check`, `fuzz`, bench and every `third_party/vt10x` line; `check: vet test-race lint vuln`), rex's `.golangci.yml`, and rex's CI workflow reduced to `make check` on `ubuntu-latest` and `macos-latest`. Header comment in each: `# copied from rex <commit-sha>, trimmed for mill M0`.
- [ ] **Step 5: Run** `make check` → all targets pass.
- [ ] **Step 6: Commit** `chore: Go module, make check gate and CI`.

### Task 2: Core types and errors

**Files:**
- Create: `internal/core/ids.go`, `internal/core/item.go`, `internal/core/state.go`, `internal/core/core_test.go`, `internal/errs/errs.go`, `internal/errs/errs_test.go`, `internal/paths/paths.go`, `internal/paths/paths_test.go`

**Interfaces:**
- Produces:
  - `core.RunID string`; `func NewRunID() RunID` (32 lowercase hex).
  - `core.Item`, `ItemMeta`, `FileRef`, `WorkspaceRef` exactly as [architecture.md §4.1](architecture.md#41-shared-data-core).
  - `core.State` with consts `StatePending, StateWorking, StateDone, StateFailed, StateCancelled`; `func (State) Terminal() bool` (done, failed, cancelled true).
  - `errs.Error{Code, Message, File string; Line, Col int; Source string /* the offending source line, optional */; Help string}`; `func (e *Error) Error() string` (single line `MXXXX message`); `func Format(err error) string` (§29 multi-line shape; non-`*Error` → `error: <msg>`); `func ExitCode(err error) int` (`*Error` with code prefix `M1`,`M2`,`M3` → 5; otherwise 1).
  - `paths.DataDir() (string, error)` honoring `MILL_DATA_DIR`, `XDG_DATA_HOME`, then `~/.local/share/mill`.

- [ ] **Step 1: Write failing tests.**
  - `TestNewRunID`: 1000 ids, each matches `^[0-9a-f]{32}$`, all distinct.
  - `TestStateTerminal`: done/failed/cancelled true; pending/working false.
  - `TestItemJSON`: an `Item{Data: json.RawMessage(`{"a":1}`), Meta: ItemMeta{ID:"x@1/out/0", Node:"x", Port:"out"}}` marshals without `files`/`ws` keys.
  - `TestFormatWithPosition`: `Format(&Error{Code:"M3012", Message:`unknown node "plann" in expression`, File:"p.mill.yaml", Line:41, Col:13, Source:"      show: ${{ nodes.plann.item }}", Help:"did you mean nodes.plan"})` contains, in order, `error[M3012]: unknown node "plann" in expression`, `┌─ p.mill.yaml:41:13`, `41 │       show: ${{ nodes.plann.item }}`, `help: did you mean nodes.plan`.
  - `TestExitCode`: `M3020` → 5, `M5201` → 1, `errors.New("x")` → 1.
  - `TestDataDir`: with `MILL_DATA_DIR=/a` → `/a`; unset and `XDG_DATA_HOME=/b` → `/b/mill`.
- [ ] **Step 2: Run** `go test ./internal/core/ ./internal/errs/ ./internal/paths/` → FAIL.
- [ ] **Step 3: Implement** the types and functions above.
- [ ] **Step 4: Run** the same command → PASS.
- [ ] **Step 5: Commit** `feat(core): ids, items, states; errs: §29 error shape`.

### Task 3: Expressions

**Files:**
- Create: `internal/expr/scan.go`, `internal/expr/env.go`, `internal/expr/template.go`, `internal/expr/expr_test.go`

**Interfaces:**
- Consumes: `errs.Error`.
- Produces:
  - `type Segment struct{ Lit, Expr string; IsExpr bool; Offset int }`; `func Scan(s string) ([]Segment, error)` — CEL-aware: a `}}` inside a CEL string literal doesn't close; unterminated `${{` → `M3101`.
  - `type Env struct{ /* cel.Env */ }`; `func NewEnv() (*Env, error)` declaring `inputs` (map), `item` (dyn), `items` (list), `nodes` (map), `now` (timestamp) and helpers `json(dyn) string`, `default(dyn, dyn) dyn`, `trim(string) string`, `lines(string) list(string)`.
  - `type Template struct{…}`; `func (e *Env) Compile(s string) (*Template, error)` (parse + check; errors are `M3101` with the offset); `func (t *Template) Eval(v Vars) (any, error)`; `func (t *Template) Refs() Refs`; `func (t *Template) IsLiteral() bool`.
  - `type Vars struct{ Inputs map[string]any; Item any; Items []any; Nodes map[string]NodeVars; Now time.Time }`; `type NodeVars struct{ Item any; Items []any; State string; ExitCode int }`.
  - `type Refs struct{ Nodes, Inputs []string }` (sorted, unique), collected by walking the parsed AST for `Select(Ident("nodes"), X)` and `Select(Ident("inputs"), X)`.
  - Eval rule: a string that is exactly one `${{ }}` returns the native Go value (`map[string]any`, `[]any`, `string`, `int64`, `float64`, `bool`, `nil`); otherwise segments are concatenated, non-strings rendered as compact JSON.

- [ ] **Step 1: Write failing tests.**
  - `TestScan`: `"a ${{ x }} b"` → 3 segments; `` "${{ "}}" + y }}" `` → 1 expr segment `"}}" + y`; `"${{ x"` → error code `M3101`.
  - `TestEvalTyped`: `${{ inputs.n + 1 }}` with `Inputs{"n": int64(2)}` → `int64(3)`; `${{ nodes.a.item }}` with `Nodes{"a":{Item: map[string]any{"k":"v"}}}` → `map[string]any{"k":"v"}`.
  - `TestEvalInterpolated`: `"n=${{ inputs.n }} m=${{ inputs.m }}"` with `n=2, m={"x":1}` → `n=2 m={"x":1}`.
  - `TestHelpers`: `${{ json(items) }}` with `Items []any{1,"a"}` → `[1,"a"]`; `${{ default(inputs.missing, "d") }}` → `"d"` when the key is absent; `${{ lines("a\nb") }}` → `[]any{"a","b"}`.
  - `TestRefs`: `"${{ nodes.plan.item.x + nodes.review.items[0] }} ${{ inputs.repo }}"` → `Nodes: [plan review]`, `Inputs: [repo]`.
  - `TestCompileError`: `${{ 1 + }}` → `*errs.Error` with `Code == "M3101"`.
- [ ] **Step 2: Run** `go test ./internal/expr/` → FAIL.
- [ ] **Step 3: Implement.** `default(a,b)` needs absent-key tolerance: declare it as a macro or check `has()` semantics; cel-go's `optional` library is acceptable. Cost limit: `cel.CostLimit(1_000_000)` on every program.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(expr): ${{ }} scanner, CEL env, templates, refs`.

### Task 4: Pipeline parse, validate, infer edges

**Files:**
- Create: `internal/pipeline/spec.go`, `internal/pipeline/parse.go`, `internal/pipeline/compile.go`, `internal/pipeline/pipeline_test.go`, `internal/pipeline/testdata/*.mill.yaml`

**Interfaces:**
- Consumes: `expr.Env`, `expr.Template`, `errs.Error`; `parts.Known(id string) bool` (Task 6 provides; until then a stub list `sh`, `agent`, `stdout` in `compile.go` replaced in Task 6).
- Produces:
  - `type Pos struct{ Line, Col int }`
  - `type Spec struct{ Mill int; Name, Description string; Inputs []InputDecl; Nodes []NodeSpec; Outputs []OutputSpec; Path string }`
  - `type InputDecl struct{ Name, Type, Description string; Default any; HasDefault bool; Pos Pos }`
  - `type NodeSpec struct{ ID, Uses, IO string; With map[string]any; In []string; Pos Pos; WithPos map[string]Pos }` (`In` accepts a string or list of node ids)
  - `type OutputSpec struct{ Name, Expr string; Pos Pos }`
  - `func Parse(path string, src []byte) (*Spec, error)` (yaml/v4 Node API; keeps file order of `nodes`, `inputs`, `outputs`)
  - `type Graph struct{ Name, BundleHash string; Inputs []InputDecl; Nodes []*Node /* topological, ties by file order */; Outputs []Output }`
  - `type Node struct{ ID, Uses, IO string; With any /* tree: map[string]any / []any / *expr.Template / literal */; In []string; After []string; Source bool; Pos Pos }`
  - `type Output struct{ Name string; Expr *expr.Template }`
  - `func Compile(s *Spec, env *expr.Env) (*Graph, error)` — `BundleHash = "sha256:" + hex(sha256("mill-bundle-v1\n" + path + "\n" + hex(sha256(src))))`.

- [ ] **Step 1: Write failing tests** (YAML fixtures in `testdata/`):
  - `TestParseOrder`: nodes `b`, `a` in file order → `Spec.Nodes[0].ID == "b"`.
  - `TestParseErrors`: bad indentation → `M2101` with line; `mill: 2` → `M3001`-style error `unsupported mill version 2` (code `M3001`); node key `retry:` → `M3001 unknown key "retry"` at its line; missing `uses` → `M3002`.
  - `TestPrimaryUpstream`: nodes `plan` (source), `review` refs `plan`, `pr` refs `plan` and `review` → `pr.In == [review]`, `pr.After` contains `plan` and `review`.
  - `TestExplicitIn`: `in: [a, b]` overrides inference; `In == [a b]`.
  - `TestAmbiguous`: `c` refs independent sources `a` and `b`, no `in` → `M3021` with help `in: [a, b]`.
  - `TestCycle`: `a` refs `b`, `b` refs `a` → `M3020` naming `a → b → a`.
  - `TestUnknownRefs`: `${{ nodes.plann.item }}` → `M3012` with `Line`, `Col` of the expression and help `did you mean nodes.plan` (Levenshtein ≤ 2); `${{ inputs.rep }}` → `M3013`.
  - `TestUnknownPart`: `uses: http` → `M5101 unknown part "http" (M0 has sh, agent, stdout)`.
  - `TestSources`: two nodes with no refs and no `in` → both `Source`, topo order = file order.
- [ ] **Step 2: Run** `go test ./internal/pipeline/` → FAIL.
- [ ] **Step 3: Implement** `Parse` and `Compile` (compile every string leaf of `with` and each output with `env.Compile`; union of `Refs().Nodes` gives order edges; data input per Global Constraints; Kahn's algorithm with file-order tie-break).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(pipeline): parse M0 subset, primary-upstream edges, topo order`.

### Task 5: Run store

**Files:**
- Create: `internal/store/rundir.go`, `internal/store/rundir_test.go`

**Interfaces:**
- Consumes: `core.RunID`, `core.Item`, `core.State`.
- Produces:
  - `type RunDir struct{ Path string; … }`
  - `func Create(root string, id core.RunID) (*RunDir, error)` — creates `root/runs/<id>/` (`0700`) plus `items/`, `logs/`.
  - `func (d *RunDir) Event(typ, nr string, data any) error` — appends one line with monotonically increasing `seq` from 1 and `ts` in UTC RFC3339Nano; `fsync` after `RunCreated` and `RunFinished`.
  - `func (d *RunDir) WriteItems(nr, port string, items []core.Item) error` → `items/<nr>/<port>.jsonl`
  - `func (d *RunDir) Logs(nr string) (stdout, stderr io.WriteCloser, err error)` → `logs/<nr>.out.log`, `logs/<nr>.err.log`
  - `func (d *RunDir) WriteJSON(name string, v any) error` (atomic temp + rename; used for `inputs.json`, `run.json`)
  - `func (d *RunDir) Close() error`

- [ ] **Step 1: Write failing tests.**
  - `TestEventsSeq`: three `Event` calls → `events.jsonl` has 3 lines, `seq` 1,2,3, each line valid JSON with `type`; `nr` omitted when empty.
  - `TestPerms`: run dir mode `0700`, `events.jsonl` mode `0600`.
  - `TestWriteItems`: two items → `items/a@1/out.jsonl` has 2 lines that unmarshal to the same `core.Item`s.
  - `TestWriteJSONAtomic`: `WriteJSON("run.json", …)` twice; file holds the second value; no `*.tmp` left.
- [ ] **Step 2: Run** `go test ./internal/store/` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(store): run directory with event log, items and logs`.

### Task 6: Parts interface, `sh` and `stdout`

**Files:**
- Create: `internal/parts/part.go`, `internal/parts/registry.go`, `internal/parts/builtin/sh/sh.go`, `internal/parts/builtin/sh/sh_test.go`, `internal/parts/builtin/stdout/stdout.go`, `internal/parts/builtin/stdout/stdout_test.go`, `internal/runner/proc/proc_unix.go`
- Modify: `internal/pipeline/compile.go` (replace the stub with `parts.Known`)

**Interfaces:**
- Consumes: `core.Item`, `errs.Error`.
- Produces:
  - `type Call struct{ RunID core.RunID; Node, NR, IO string; With map[string]any /* evaluated */; Input []json.RawMessage /* item data */; Stdout, Stderr io.Writer /* log sinks */; Out io.Writer /* mill's stdout, stdout part only */; Env []string }`
  - `type Result struct{ Items []json.RawMessage; ExitCode int }`
  - `type Part interface{ Run(ctx context.Context, c Call) (Result, error) }`
  - `func Lookup(id string) (Part, bool)`; `func Known(id string) bool`; `func Register(id string, p Part)` (built-ins register in `init`; tests register fakes); `func StepEnv(runID core.RunID, node string, extra map[string]string) []string` (allowlist from Global Constraints).
  - `proc.Start(ctx, cmd *exec.Cmd)` sets `Setpgid`; on `ctx.Done()` sends `SIGTERM` to `-pgid`, waits 5 s, then `SIGKILL`.
  - `sh` `with`: `run` (string, required), `dir` (string, optional, leading `~/` expanded), `ok_codes` ([]int, default `[0]`). Runs `/bin/sh -c <run>` in `dir`.

- [ ] **Step 1: Write failing tests** (real `/bin/sh`; `//go:build unix`):
  - `TestShPlain`: input `[{"n":1},{"n":2}]`, `run: "jq -c '{m: (.n*10)}'"` → items `{"m":10}`, `{"m":20}` (skip if `jq` missing).
  - `TestShText`: `io: text`, input `["a","b"]`, `run: "tr a-z A-Z"` → items `"A"`, `"B"`.
  - `TestShNoTrailingNewline`: `run: "printf '{\"x\":1}'"` → one item `{"x":1}`.
  - `TestShNonJSON`: `run: "echo hello"` with `io: plain` → `*errs.Error` code `M5201`, message contains `line 1` and `hello`, help contains `io: text`.
  - `TestShExitCode`: `run: "exit 3"` → `M5201` message contains `exit 3`; with `ok_codes: [0,3]` → no error, `ExitCode == 3`.
  - `TestShDirTilde`: `dir: "~/"` runs in `$HOME` (`run: "pwd"`, `io: text` → item equals `os.UserHomeDir()`); `dir: "~/no-such-dir-xyz"` → `M5201 dir "…" does not exist`.
  - `TestShEnvAllowlist`: parent env has `SECRET_X=1`; `run: "env"` output has no `SECRET_X` and has `MILL_NODE_ID=`.
  - `TestShCancel`: `run: "sleep 307 & sleep 307"`, cancel ctx after 100 ms → returns within 6 s, and `pgrep -f 'sleep 307'` finds nothing.
  - `TestStdout`: two items → `Out` gets `{"a":1}\n{"b":2}\n`; `Result.Items` passes the input through unchanged.
- [ ] **Step 2: Run** `go test ./internal/parts/... ./internal/runner/...` → FAIL.
- [ ] **Step 3: Implement** the interface, registry, `sh`, `stdout`, `proc`. Output parsing: `bufio.Reader` with no line-length cap (lines up to 64 MiB), blank lines skipped.
- [ ] **Step 4: Run** → PASS. Also `go test ./internal/pipeline/` → still PASS.
- [ ] **Step 5: Commit** `feat(parts): sh and stdout parts, process-group cancel`.

### Task 7: Agents and the `agent` part

**Files:**
- Create: `internal/agents/entry.go`, `internal/agents/builtin.go`, `internal/agents/build.go`, `internal/agents/output.go`, `internal/agents/agents_test.go`, `internal/testutil/echoagent/main.go`, `internal/parts/builtin/agent/agent.go`, `internal/parts/builtin/agent/agent_test.go`

**Interfaces:**
- Consumes: `parts.Part`, `parts.Call`, `parts.Result`, `parts.StepEnv`, `proc.Start`.
- Produces:
  - `type Entry struct{ ID string; Command []string; Models map[string][]string; HeadlessArgs []string; Output string /* "stdout-json" | "stdout-text" */; ResultPath, UsageIn, UsageOut string /* dotted JSON paths */ }`
  - Built-ins: `claude` = `Command: ["claude"]`, `HeadlessArgs: ["-p", "--output-format", "json"]`, `Models: {"haiku": ["--model","haiku"], "sonnet": ["--model","sonnet"], "opus": ["--model","opus"]}`, `Output: "stdout-json"`, `ResultPath: "result"`, `UsageIn: "usage.input_tokens"`, `UsageOut: "usage.output_tokens"`, prompt as the last argv element. `echo` = `Command: ["mill-echo-agent"]`, `Output: "stdout-text"`, prompt on stdin.
  - `func Get(id string) (Entry, bool)`; `func Build(e Entry, model, prompt string) (argv []string, stdin string, err error)` (unknown model → error listing known models).
  - `func ParseOutput(e Entry, stdout []byte) (text string, usage core.Usage, err error)`.
  - `agent` part `with`: `agent` (string `id` or `id/model`, required), `model` (string, optional, overrides), `prompt` (string, required), `dir` (optional, `~/` expanded). Output: one item, a JSON string of the agent's final text; if the text is a JSON object or array, the item is that value instead.
  - `mill-echo-agent`: reads stdin, writes `echo: <first 200 bytes of stdin>` to stdout; `ECHO_AGENT_EXIT=<n>` makes it write `boom` to stderr and exit n; `ECHO_AGENT_JSON=1` makes it print `{"ok":true}`.

- [ ] **Step 1: Write failing tests.**
  - `TestBuildClaude`: `Build(claude, "haiku", "hi")` → `["claude","-p","--output-format","json","--model","haiku","hi"]`, stdin `""`.
  - `TestParseClaude`: stdout `{"type":"result","result":"done","usage":{"input_tokens":12,"output_tokens":3}}` → text `done`, usage in 12 / out 3.
  - `TestAgentEcho` (builds `mill-echo-agent` into `t.TempDir()` with `go build`, prepends it to PATH): prompt `hello` → one item `"echo: hello"`.
  - `TestAgentEchoJSON`: `ECHO_AGENT_JSON=1` → item `{"ok":true}`.
  - `TestAgentMissing`: `agent: "claude"` with PATH set to an empty temp dir → `M6101 claude not found on PATH`, help mentions `PATH=` and `crontab`.
  - `TestAgentFails`: `ECHO_AGENT_EXIT=2` → `M6300`, message contains `exit 2` and `boom`.
  - `TestAgentModelSyntax`: `agent: "claude/haiku"` and `agent: "claude", model: "haiku"` build identical argv.
- [ ] **Step 2: Run** `go test ./internal/agents/ ./internal/parts/builtin/agent/` → FAIL.
- [ ] **Step 3: Implement**; register `agent` in `parts`. Env for the agent: `StepEnv` (claude reads auth from `$HOME`).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(agents): headless claude and echo agents, agent part`.

### Task 8: Engine

**Files:**
- Create: `internal/engine/engine.go`, `internal/engine/engine_test.go`

**Interfaces:**
- Consumes: `pipeline.Graph`, `pipeline.Node`, `expr.Vars`, `parts.Lookup`, `store.RunDir`, `core.*`.
- Produces:
  - `type Options struct{ StoreRoot string; Inputs map[string]any; Stdin []json.RawMessage; Out io.Writer /* results */; Progress io.Writer /* stderr */ }`
  - `type Result struct{ RunID core.RunID; State core.State; Outputs map[string]any; Dir string; Err error }`
  - `func Run(ctx context.Context, g *pipeline.Graph, opt Options) Result`
  - Behaviour: resolve inputs (declared defaults; a declared input without default and not provided → `M1309 input "<name>" is required` before any run dir is created); write `inputs.json`, `RunCreated`; for each node in `g.Nodes`: input items = concatenation of `items` of its `In` nodes, or `opt.Stdin` for the first source; evaluate `With` with `Vars{Inputs, Items: input, Item: input[0] or nil, Nodes: finished nodes}`; `NodeRunStarted`; `Part.Run`; write items with `ItemMeta{ID:"<node>@1/out/<i>", Run, Node, Port:"out", Index:i, TS}`; `NodeRunFinished`. First failure stops the run (`state failed`); later nodes get no events. Outputs evaluated after all nodes are done. `RunFinished` and `run.json` (`{run, pipeline, state, outputs, started_at, finished_at}`) always written, also on failure and cancel. Progress lines on `opt.Progress`: `≋ <pipeline> <short-id>`, then per node `  ✓ <node> <n> items <dur>` / `  ✗ <node> <code>`.

- [ ] **Step 1: Write failing tests** (real `sh` nodes; fake parts via `parts.Register` where a test needs one):
  - `TestRunLinear`: `a` sh `echo '{"n":1}'` → `b` sh `jq -c '{n: (.n+1)}'` → outputs `{ n: ${{ nodes.b.item.n }} }` → `State == done`, `Outputs["n"] == int64(2)`; `events.jsonl` types in order `RunCreated, NodeRunStarted, NodeRunFinished, NodeRunStarted, NodeRunFinished, RunFinished`.
  - `TestRunStdinFirstSourceOnly`: two sources; `Stdin` two items → first source sees 2 items, second sees 0.
  - `TestRunFailureStops`: `a` exits 1, `b` downstream → `State == failed`, `Err` code `M5201`, no `NodeRunStarted` for `b`, `RunFinished.state == "failed"`.
  - `TestRunMissingInput`: required input absent → `Err` code `M1309`, no run dir created.
  - `TestRunCancel`: node `sleep 308`; cancel after 100 ms → `State == cancelled` within 6 s, `RunFinished.state == "cancelled"`.
  - `TestRunWithTemplating`: `with.run: "echo ${{ json(inputs.msg) }}"` and input `msg = "hi"` → item `"hi"`.
- [ ] **Step 2: Run** `go test ./internal/engine/` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `go test -race ./...` → PASS.
- [ ] **Step 5: Commit** `feat(engine): sequential run with items, events, outputs, cancel`.

### Task 9: `mill run` CLI

**Files:**
- Create: `internal/surface/cli/run.go`, `internal/surface/cli/cli_test.go`, `internal/surface/cli/testdata/script/{run_hello,run_stdin,run_errors,run_json,run_signal}.txtar`
- Modify: `internal/surface/cli/main.go`

**Interfaces:**
- Consumes: `pipeline.Parse`, `pipeline.Compile`, `expr.NewEnv`, `engine.Run`, `errs.Format`, `errs.ExitCode`, `paths.DataDir`.
- Produces: `mill run <file> [--input k=v]... [--stdin] [--text] [--json] [--store dir]`. `--input` values are parsed as JSON when they parse (`n=2` → `2`, `x={"a":1}`), else as strings. `--stdin` reads JSONL (`--text`: each line a JSON string). SIGINT/SIGTERM cancel the context. Exit code from `errs.ExitCode` or the run state (done 0, failed/cancelled 1).

- [ ] **Step 1: Write failing testscript tests** (`TestScript` runs `testscript.Run` with `mill` mapped to `cli.Main`, `jq` from PATH):
  - `run_hello`: two `sh` nodes with `outputs: {greeting: ...}`; `exec mill run hello.mill.yaml` → stdout `{"greeting":"hello, mill"}`; `exec mill run hello.mill.yaml --json` → stdout JSON has `.state == "done"`, `.outputs.greeting`, `.dir` exists; stderr contains `✓`.
  - `run_stdin`: `stdin items.jsonl` + `exec mill run up.mill.yaml --stdin` → upper-cased items printed by a `stdout` node, one per line.
  - `run_errors`: missing file → `error[M1201]`, exit 5 (invalid argument); unknown flag `--nope` → `M1301`, exit 5; YAML with `nodes.plann` → stderr contains `error[M3012]` and `┌─`, exit 5; failing node → `error[M5201]`, exit 1.
  - `run_json`: `--input n=2` reaches `${{ inputs.n + 1 }}` as `3`.
  - `run_signal`: `exec mill run sleep.mill.yaml &`, `sleep 1`, `kill -INT` the process (testscript `kill`), `wait` → exit 1, stderr contains `cancelled`.
- [ ] **Step 2: Run** `go test ./internal/surface/cli/` → FAIL.
- [ ] **Step 3: Implement** `run.go`.
- [ ] **Step 4: Run** `make check` → PASS.
- [ ] **Step 5: Commit** `feat(cli): mill run with inputs, stdin, json output, exit codes`.

### Task 10: Examples and the dogfood pipeline

**Files:**
- Create: `examples/hello.mill.yaml`, `examples/pipe-demo.mill.yaml`, `examples/nightly-vuln-sweep-lite.mill.yaml`, `examples/README.md`, `internal/surface/cli/testdata/script/examples.txtar`

**Interfaces:**
- Consumes: everything above. No new Go API.

- [ ] **Step 1: Write the failing test** `examples.txtar`: runs `examples/hello.mill.yaml` (stdout `.greeting == "hello, mill"`), `examples/pipe-demo.mill.yaml` with `--stdin` over three JSON lines (three items out, transformed by `jq` in an `io: plain` node, filtered by a second, printed by `stdout`), and `examples/nightly-vuln-sweep-lite.mill.yaml --input agent=echo --input repo=$WORK/repo --input notes=$WORK/notes` against a tiny module in `$WORK/repo` (`go.mod` + one file, no deps) → exit 0 and `$WORK/notes/vuln-*.md` exists and contains `echo:`.
- [ ] **Step 2: Run** `go test ./internal/surface/cli/ -run TestScript/examples` → FAIL (files missing).
- [ ] **Step 3: Write the pipelines.** `nightly-vuln-sweep-lite`:
  - inputs: `repo` (default `~/Documents/personal/dev/rex`), `agent` (default `claude/haiku`), `notes` (default `~/notes`).
  - `scan`: `sh`, `dir: ${{ inputs.repo }}`, `run: go run golang.org/x/vuln/cmd/govulncheck@v1.8.0 -format json ./... | jq -c 'select(.finding)'`, `io: plain` (each finding is an item; none → zero items).
  - `triage`: `agent`, `agent: ${{ inputs.agent }}`, prompt: `"Triage these govulncheck findings for ${{ inputs.repo }}. Say what is really reachable, what to fix first, and the exact upgrade command for each. If the list is empty, reply 'No findings.' Findings (JSON): ${{ json(items) }}"`.
  - `write`: `sh`, `io: text`, `run: n=${{ json(inputs.notes) }}; case "$n" in "~/"*) n="$HOME/${n#\~/}";; esac; mkdir -p "$n" && f="$n/vuln-$(date +%F).md" && cat > "$f" && echo "$f"` (the shell expands a leading `~/` because the value arrives quoted; `json()` quotes it safely).
  - outputs: `report: ${{ nodes.write.item }}`.
  - `examples/README.md`: how to run each; the crontab line `30 3 * * * PATH=/opt/homebrew/bin:/usr/local/go/bin:/usr/bin:/bin $HOME/go/bin/mill run $HOME/Documents/personal/dev/mill/examples/nightly-vuln-sweep-lite.mill.yaml >> $HOME/.local/state/mill/vuln-sweep.log 2>&1`, and why PATH is set (cron's PATH lacks Homebrew; Review Focus 3).
- [ ] **Step 4: Run** `make check` → PASS. Then run for real once: `go run ./cmd/mill run examples/nightly-vuln-sweep-lite.mill.yaml` → exit 0, report path printed, the file reads as a sensible triage.
- [ ] **Step 5: Commit** `feat(examples): hello, pipe-demo, nightly-vuln-sweep-lite`.

### Task 11: Dogfood and findings

**Files:**
- Create: `docs/plans/backend/m0-findings.md`
- Modify: `docs/agents/HANDOFF.md` (state: code exists, M0 done, next M1), `CHANGELOG.md` (`### Added`: `mill run` prototype), `docs/plans/backend/milestones.md` (M0 exit criterion 1 wording per Global Constraints)

**Interfaces:** none.

- [ ] **Step 1:** Install the crontab line from `examples/README.md` (the user does this; print it for them) and let it run at least 3 nights.
- [ ] **Step 2:** Write `m0-findings.md` with sections: file format (what was awkward to write), item model (`plain` vs `text` default; was `io: plain` right for `sh`?), edge inference (did primary-upstream ever surprise?), ergonomics of `${{ }}` in shell strings (quoting, `json()`), what the vuln sweep needed that M0 lacked (e.g. `env`, `when` to skip the agent on zero findings, `~` expansion), cron behaviour (3 nightly results, failures and their messages), and a verdict on decisions D9 (primary upstream) and D10 (stream mode deferred).
- [ ] **Step 3:** Update HANDOFF, CHANGELOG, milestones.md; run the docs link check.
- [ ] **Step 4: Commit** `docs: M0 findings and handoff update`; push the branch, open the PR, get the independent review, merge.
