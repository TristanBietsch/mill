# Conventions

Planned conventions, from the spec and rex's observed practice. Once code exists, replace each section with what the code actually does (rex's rule: observed, not aspirational).

## Module layout

- Go module `github.com/TristanBietsch/mill`, one module, Go 1.27, `CGO_ENABLED=0`, MIT.
- Binaries under `cmd/<name>/main.go` only; `main` stays thin and calls into `internal/`.
- Libraries under `internal/<package>/`. One package per directory. Tests in the same package as `*_test.go`; black-box tests as `package x_test`.
- File names `snake_case.go`. OS-specific files `_unix.go`, `_windows.go`.

## Go style

- `gofmt` and `goimports`. golangci-lint with rex's linter set as the base, plus `depguard` for layering and `forbidigo` for stray `fmt.Print*`.
- Doc comments on every exported symbol, as full sentences starting with the name.
- `context.Context` first on anything that blocks or does I/O. No goroutine without an owner that stops it; `goleak` checks every package.
- Every timer, sleep and `time.Now` goes through the injected clock.
- No `panic` for expected failures. Panics are for programmer bugs only and surface as `M0001`.
- Paths from the paths package only. Never resolve a path relative to the working directory by accident, never fall back to `/tmp`.

## Errors

Spec: [§29](../spec/29-errors.md#29-errors-and-diagnostics).

- Libraries return `error` and wrap with `%w` where context helps.
- Anything that can reach a user becomes mill's error type: code, class, message, where, suggestions, help, retryable, cause.
- **Codes:** `M` + four digits. Area by thousands: `M0` internal, `M1` CLI and selectors, `M2` files and pipelines, `M3` lint and expressions, `M4` runs and scheduling, `M5` parts and plugins, `M6` agents, `M7` daemon and protocol, `M8` Lua config, `M9` security, trust, secrets. Never reuse or renumber; a changed meaning gets a new code.
- **Headline:** plain words, names the thing, lowercase start, no trailing period, under 100 characters. No blame, no internal jargon (goroutine, nil, EOF).
- **Help line:** a command the user can copy and run.
- **Suggestions:** `did you mean` from edit distance over what exists.
- **No stack traces** unless `--debug` or `MILL_DEBUG=1`; always in the daemon log.
- **Redact** every error before printing, logging or sending.
- Plugin errors are namespaced (`linear/E_TEAM_NOT_FOUND`) and wrapped as the cause of `M5201`.
- Each code: an explain page and golden tests for human and JSON output.

## Exit codes

Spec: [§17.2](../spec/17-cli.md#172-exit-codes), [§29.6](../spec/29-errors.md#296-error-classes). Derived from the error class, never chosen ad hoc.

| Code | Meaning | Class |
|---|---|---|
| 0 | Success | |
| 1 | Generic error or the run failed | `transient`, `rate_limited`, `auth`, `cancelled`, `agent`, `internal` |
| 2 | Not found | `not_found` |
| 3 | Ambiguous selector | `ambiguous` |
| 4 | Daemon unreachable | `unreachable` |
| 5 | Invalid arguments or lint failure | `user` |
| 6 | Refused or conflict | `refused`, `conflict` |
| 7 | Timed out | `timeout` |
| 8 | Reserved | |
| 9 | Budget exceeded | `budget` (a run `cancelled` with reason `budget`, backend plan S9) |

## Output

Spec: [§17.4](../spec/17-cli.md#174-conventions), [§17.6](../spec/17-cli.md#176-output-modes).

- Data on stdout, diagnostics and progress on stderr. Always.
- Every listing takes `--json`: one JSON document. Streaming commands with `--json` print JSONL, one object per line, flushed.
- With `--json`, a failure prints the error object on **stdout** and exits non-zero.
- Pretty mode only on a terminal. Plain when piped, `--plain` or `NO_COLOR`. `MILL_ASCII=1` replaces glyphs with words.
- Table columns keep their order; new columns only on the right. Relative times on screen, RFC 3339 in plain and JSON.
- Destructive commands confirm in a terminal and need `--yes` otherwise (exit 6). `--dry-run` on every command that changes state.

## Naming

| Thing | Form | Example |
|---|---|---|
| Pipeline name | lowercase kebab-case | `issue-to-pr` |
| Node id | `[a-z0-9][a-z0-9_-]*` | `hand-back` |
| Part id | lowercase, dot-namespaced by plugin | `linear.issue.create`, `agent.loop` |
| Plugin name | `owner/name` | `acme/linear` |
| JSON and wire fields | `snake_case` | `run_id`, `node_run_id` |
| Intent and event types | `PascalCase` | `StartRun`, `NodeRunUpdated` |
| States, classes | lowercase | `working`, `rate_limited` |
| Settings | `snake_case`, dot-grouped | `max_concurrent_agents`, `sound.volume` |
| Env vars | `MILL_` + upper snake | `MILL_OUTPUT`, `MILL_SOCKET` |
| Error codes | `M` + 4 digits | `M6304` |
| Commit subjects | Conventional Commits | `feat(pipeline): infer edges from in:` |

## Logging

- `log/slog` to a file (`milld.log`, `logs/tui.log`), mode 0600.
- Logs carry ids, counts and durations. Never secret values, never prompts or item payloads by default.

## Tests

- TDD: the failing test first.
- Table-driven where cases vary by input.
- Hermetic: temp `HOME`, `XDG_*`, `MILL_*`; no test reads real config, dials a real daemon or calls a real agent (use the `echo` agent).
- Fake clock for anything with time.
- Golden files for CLI output, errors and TUI views; update with a flag, review the diff.
- Synthetic data only.

## Documentation voice

- Plain, short sentences. Concrete: names, numbers, commands. Tables where a list has more than two attributes.
- No filler, no marketing, no "simply" or "just".
- User docs describe shipped behavior. Agent docs are navigation and link instead of copying tables.
- Examples use synthetic data.
