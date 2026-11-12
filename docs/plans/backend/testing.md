# Testing

Test tiers, fakes, fixtures, fuzzing, golden files, benchmarks and the CI gate. Most of the scaffolding is copied from rex (`testutil/testenv`, `golden`, `bench/`, `tools/coverage`, `Makefile`), then extended for an engine that runs other people's processes.

The test strategy of each layer is in [layers.md](layers.md) §4; the layer checks (`internal/archtest`, depguard, forbidigo) in [layers.md](layers.md) §8; benchmark budgets in [performance.md](performance.md). Citations: [references.md](references.md); sources table in §10.

## 1. Principles

1. **Hermetic by default.** Every package's `TestMain` calls `testenv.RunLeakChecked(m)` (from rex): throwaway `HOME`, `XDG_*`, `MILL_*`, `PATH` trimmed to the test helpers + system dirs, goleak after the run. No test can read the developer's config, dial their daemon or call a real agent.
2. **Fake time everywhere it matters.** Every timer (retries, budgets, cron, debounce, grace periods, coalescing) goes through `clock.Clock`. Engine and trigger tests use a fake clock (or `testing/synctest` bubbles where goroutines sleep) so a "5 minute backoff" test runs in microseconds and deterministically.
3. **Synthetic data only** (spec §22). Fixtures use invented repos, issues and tokens; a test (`fixtures_lint_test.go`) greps fixtures for token-shaped strings and personal paths.
4. **Race always.** `go test -race` is the default in `make check`; there's no non-race CI job for Linux/macOS.
5. **Every invariant has a test** (spec §5.4). The table in §8 maps each one.
6. **Fakes over mocks** [SEG ch. 13]: real binaries (echo agent, fake CLIs, `mill-test-part`) and in-memory fakes with real behavior; no interaction-verifying mocks of our own interfaces.
7. **Test sizes follow layers** [SEG ch. 11]: L0–L2 are mostly small (T0) tests; L3–L5 add medium component tests (T1, T2); L6–L7 are covered by larger tests (T3, T4).

## 2. Tiers

| Tier | Scope | Runs | Example |
|---|---|---|---|
| T0 unit | pure functions: CEL helpers, template scanner, edge inference, retry parser, JCS, redactor, reducer, selectors | `go test ./...` | `TestInferPrimaryUpstream` |
| T1 component | one subsystem with fakes: engine with fake parts + fake runner + fake clock + temp store; scheduler with synthetic requests; store with crash injection | `go test ./...` | `TestEngine_RetryRateLimitedHonorsRetryAfter` |
| T2 integration | real processes and files: `local`/`pty` runners with the echo agent and helper binaries, real `git` worktrees in temp repos, daemon server over a real UDS, MCP fake server over stdio | `go test ./...` (skips when `git` missing) | `TestDaemon_StartSubscribeFinish` |
| T3 CLI scripts | `testscript` over `mill` and `milld` built in-process | `go test ./cmd/mill/...` | `testdata/script/run_pipe_jq.txtar` |
| T4 examples | every pipeline in `examples/` run end to end with `--local` against the echo agent and recorded HTTP | `make test-e2e` (CI on Linux and macOS) | `examples/issue-to-pr` with fixture issue |
| T5 stress / fuzz | long fuzz, `-count=50`, crash-point exploration | nightly workflows | `FuzzEventLogReplay` |

## 3. The echo agent

A real binary, so process-group, PTY, stdin and exit-code behavior is exercised for real on all three OSes. Source: `internal/testutil/echoagent/cmd/mill-echo-agent`, built once per test binary into a temp dir by `testutil.EchoAgentPath(t)` (cached with `sync.Once`) and put on the hermetic `PATH`. Registry entry `echo` (headless + terminal variants) points at it.

Behavior is driven by directives in the prompt (so pipelines in `examples/` can script it without extra plumbing):

| Directive | Effect |
|---|---|
| `@echo:say <text>` | prints `<text>` as the final message |
| `@echo:json <json>` | headless JSON result `{"result": <json>, "usage": {...}}` |
| `@echo:output <json>` | writes `<json>` to `$MILL_OUTPUT` |
| `@echo:bad-output` | writes JSON that fails the schema (tests the retry-once path) |
| `@echo:usage <in> <out>` | reports that usage |
| `@echo:fail <exit> <text>` | prints text to stderr, exits |
| `@echo:rate-limit <n> <secs>` | fails with `429 rate limit, retry after <secs>s` for the first n attempts (state in `$MILL_SCRATCH/../echo-state`) |
| `@echo:sleep <dur>` | sleeps (respects SIGTERM unless `@echo:ignore-term`) |
| `@echo:ignore-term` | ignores SIGTERM/CTRL_BREAK (tests the kill path) |
| `@echo:spawn-child` | spawns a grandchild that sleeps (tests group kill) |
| `@echo:edit <path> <content>` | writes a file in `$MILL_WORKSPACE` (tests diff, worktrees, retry reset) |
| `@echo:ask <question>` | terminal mode: prints the question and the ready prompt again (tests M6304) |
| `@echo:trust-dialog` | terminal mode: shows a folder-trust dialog (tests `auto_answer`) |
| `@echo:print-env <NAME>` | prints an env var (tests secret delivery and redaction) |
| `@echo:flood <bytes>` | writes that much output fast (tests drain limits, log caps, benchmarks) |

Other helpers built the same way: `mill-test-part` (a configurable executable part speaking the parts protocol: crash on Open, ignore Cancel, never Pong, emit huge items), `mill-test-mcp` (stdio MCP server with fixed tools via the official SDK).

## 4. Recorded HTTP and external CLIs

- **HTTP:** `go-vcr` v4 cassettes under `testdata/cassettes/<test>.yaml` for the `http` part, `gh.*` in API mode, Vault backend, n8n/zapier calls, MCP over streamable HTTP. A `BeforeSaveHook` strips `Authorization`, cookies and any registered fake secret. Mode `replay` in CI (unmatched request fails the test); `-record` flag locally.
- **`gh`, `git`, `docker`, `op`, `ssh`:** `git` is real (it's required). The others get fake executables in `testdata/bin/` (small Go programs from `internal/testutil/fakecli`) that record argv/env to a JSONL file and answer from fixtures, so tests assert exact invocations ("`gh pr create --draft --title …`, token from env not argv").
- **Agent CLIs:** never called in tests. `mill agent test claude` has a separate opt-in smoke workflow (`MILL_LIVE_AGENTS=1`, manual dispatch) that never runs on PRs.

## 5. CLI tests with testscript

`cmd/mill/main_test.go`:

```go
func TestMain(m *testing.M) {
	testscript.Main(m, map[string]func(){
		"mill":  mill.Main,
		"milld": milld.Main,
		"mill-echo-agent": echoagent.Main,
	})
}
func TestScripts(t *testing.T) {
	testscript.Run(t, testscript.Params{Dir: "testdata/script", Setup: testenv.ScriptSetup, UpdateScripts: *update})
}
```

Example `testdata/script/run_local_pipe.txtar`:

```
# a two-step pipeline pipes JSONL into jq-like filtering
exec mill run hello --local --input name=ada --json
stdout '"greeting":"hello ada"'
! stderr 'error'

# exit codes follow the run
! exec mill run broken --local
stderr 'error\[M3012\]: unknown node "plann"'
exit 5

-- .mill/pipelines/hello.mill.yaml --
mill: 1
name: hello
inputs: { name: { type: string } }
nodes:
  greet: { uses: sh, with: { run: "echo '{\"greeting\":\"hello ${{ inputs.name }}\"}'" } }
outputs: { greeting: ${{ nodes.greet.item.greeting }} }
-- .mill/pipelines/broken.mill.yaml --
…
```

Daemon scripts start `milld` in the background with a socket under `$WORK` (`exec milld --socket $WORK/s &`), then use `mill` against it, then `mill daemon stop`.

## 6. Fuzzing targets

| Target | Package | Property |
|---|---|---|
| `FuzzCodecRead` | `wire/jsonl` | copied from rex: never panics, sticky too-long, resumes after malformed |
| `FuzzPipelineParse` | `pipeline` | never panics; positions within file bounds; Parse(Print(spec)) == spec for valid inputs |
| `FuzzTemplateScan` | `expr` | `${{ }}` scanning never panics; reconstructing literal + expr segments equals input |
| `FuzzPipeExpr` | `pipeline/pipe` | never panics; `--save` YAML recompiles to the same IR |
| `FuzzCELHelpers` | `expr` | helpers never panic, respect cost limits |
| `FuzzRetrySpec`, `FuzzDuration`, `FuzzCronSpec` | `engine/retry`, `trigger/cron` | parse/format round trip |
| `FuzzSelector` | `core` | copied pattern from rex `selector_fuzz_test.go` |
| `FuzzPartMessages` | `partproto` | decoding arbitrary part output never panics and never crashes the host's context state machine (sequence fuzz: Item after Done, double Fail, …) |
| `FuzzEventLogReplay` | `core/fold` + `store` | any byte prefix of a valid log (torn writes) replays to a state the reducer accepts; truncated tail detected; invariant 6 holds |
| `FuzzRedactorChunks` | `redact` | for random secrets and random chunk boundaries, no secret (or encoding) survives in the output |
| `FuzzJCS` | `engine/cache` | canonical form is idempotent and stable across map orders |
| `FuzzPathSafety` | `paths` | no accepted path escapes its root (checked against `os.Root` behavior) |
| `FuzzYAMLEdit` | `pipeline/yamledit` | applying an op then re-parsing yields the intended spec; untouched lines unchanged |

`make fuzz` runs all for `FUZZTIME` (rex Makefile loop). Nightly `fuzz.yml` runs 5 min per target and uploads new corpus entries as an artifact.

## 7. Golden files, benchmarks, budgets

**Golden** (`testutil/golden`, `-update`, txtar archives per rex's CLI kit):
- every error code: human text and `--json` (spec §29.11; `make errors-check` fails if a code lacks an explain page or a golden, or a page has no code);
- every CLI command in each output mode (pretty with `NO_COLOR`, plain, json);
- lint diagnostics with carets;
- `mill show --outline`, `mill render` (text, mermaid), `mill watch` lines;
- compiled IR dumps for `examples/`;
- event logs of deterministic example runs (fake clock, fixed ids) — any change to event shapes shows up in review.

**Benchmarks** (copy rex's `bench/` layout: `budgets.txt`, `check_budgets.sh`, `BASELINE.md`, `make bench`, `make bench-check` fails on >10 % regression): the budget table B1–B16 moved to [performance.md](performance.md) §2 (B1–B12 unchanged, B13–B16 new).

## 8. Invariants → tests

| Invariant (spec §5.4) | Test |
|---|---|
| 1 one JSON object per line, `v` checked | codec tests + `FuzzCodecRead`; parts protocol conformance |
| 2 no broadcast before Hello | `TestServer_NoEventsBeforeHello` (copied from rex pattern) |
| 3 binary is base64 | `TestNodeOutput_Base64` |
| 4 logs append-only | store opens with `O_APPEND`; `TestStore_NoRewrite` checks the file never shrinks except torn-tail recovery (which copies the tail aside first) |
| 5 raw output stored as-is (minus secrets) | `TestLogs_ByteFaithful` with binary + ANSI + invalid UTF-8 input |
| 6 terminal never changes | reducer property test over random event sequences; `FuzzEventLogReplay` |
| 7 slots released on every path | `sched.AssertDrained` in every engine test cleanup; chaos test kills parts at random points |
| 8 one teardown path, no transient failed | `TestCancel_NeverFailed` with `@echo:ignore-term` and `@echo:spawn-child`; asserts grandchild dead, state sequence `working → cancelled` |
| 9 secrets never persisted | `TestSecrets_NotOnDisk`: runs a pipeline that prints/encodes a fake secret every way, then greps the whole data dir and the client event stream |
| 10 reload never changes a live run | `TestReload_RunKeepsBundleHash` edits the YAML mid-run |

**Crash-point exploration** (T5): run each example under the engine with a store wrapper that panics on the Nth append (N = 1…total), then recover and finish; assert no duplicate execution of non-idempotent parts (the fake CLIs' invocation logs), no lost terminal states, slots drained.

## 9. CI gate

`make check` = `vet` + `test-race` + `lint` + `lint-windows` + `cover-check` + `vuln` + `errors-check` + `layering` (`go test ./internal/archtest/`, which also runs inside `test-race`; depguard and forbidigo run inside `lint`). Workflows copied from rex and adjusted:

| Job | OS | Runs |
|---|---|---|
| check | ubuntu-latest | `make build check` with `CGO_ENABLED=0` |
| check-macos | macos-latest | `make build vet test-race lint vuln` |
| windows | windows-latest | `go vet ./...`, `go test ./...`, smoke: `mill run examples/hello --local`, daemon start → run → stop |
| e2e | ubuntu, macos | `make test-e2e` |
| fuzz (nightly) | ubuntu | `make fuzz FUZZTIME=5m` |
| stress (nightly) | ubuntu | `go test -race -count=50 ./internal/engine/... ./internal/store/...` + crash-point exploration |
| bench (on label `perf`) | ubuntu | `make bench bench-check` |

Coverage floors start at what M1 measures, per package, in `tools/coverage/baseline.txt`, never below the layer floor (D39): L0 85 %, L1 95 %, L2 85 %, L3 85 %, L4 85 %, L5 75 %, L6 75 %, L7 70 %. Floors only rise (ratchet) [BEA ch. 2].

## 10. Sources

| Decision | Section | Follows |
|---|---|---|
| Hermetic tests, no network, synthetic data | §1 | [SEG ch. 11] [SEG ch. 14] |
| Fake time everywhere | §1 | [SEG ch. 12] [CIG ch. 5] |
| Test sizes by layer | §1–2 | [SEG ch. 11] |
| Real fake binaries over mocks | §3–4 | [SEG ch. 13] |
| Recorded HTTP for integrations | §4 | [SEG ch. 14] |
| Fuzzing every parser, decoder and the redactor | §6 | [BSRS ch. 13] [LG ch. 15] |
| Goldens for every visible output | §7 | [SEG ch. 1, Hyrum's Law] |
| Every invariant has a test | §8 | [BSRS ch. 13] |
| Crash-point exploration | §8 | [RI ch. 17] [DBI ch. 5] |
| CI gate with layering and coverage ratchets | §9 | [SEG ch. 23] [BEA ch. 4] |
