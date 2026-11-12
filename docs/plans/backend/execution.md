# Execution

Where node-runs execute and how processes are started, watched and killed: runners, process groups per OS, PTYs and terminal-only agents, workspaces, the agent registry, and the structured-output pipeline. Refines spec §11, §12, §6.4.

## 1. Runners

| Runner | Milestone | How | Caps |
|---|---|---|---|
| `local` | M0 | `os/exec` with a new process group (Unix) / Job Object (Windows); stdin/stdout/stderr pipes | default |
| `pty` | M6 | creack/pty (Unix), ConPTY via `charmbracelet/x/xpty` (Windows); vt10x screen mirror | `pty`, view-only attach, repaint |
| `container` | M9 | `docker` or `podman` CLI (`runtime.container = auto` picks whichever is on PATH, podman first): `run --rm --init -i --name mill-<run8>-<nr> --label mill.run=<id> --label mill.node_run=<nr> -v <ws>:/work -w /work --network none|bridge --memory --cpus --pids-limit <image> <argv>` | network control, mounts, limits |
| `ssh` | M9 | system `ssh` binary (respects `~/.ssh/config`, agents, ProxyJump); `ssh -T -o BatchMode=yes host -- <argv>`; workspace either a remote path (`remote_path`) or synced with `rsync` before/after | remote |
| `remote` | M10 | hand the node-run to another milld through the gateway | |
| `external` | M10 | adapter part (e.g. `mill/rex`) | |

Selection: node `runner` → pipeline `defaults.runner` → `default_runner` setting; `policy.allowed_runners` filters (lint M3210, runtime refusal M9302). Runner definitions (`mill.runner "gpu-box" { kind = "ssh", … }`) come from config and are frozen per run by the config hash.

Which parts use a runner: process-launching built-ins (`sh`, `exec`, `check`, every `agent*`, `git.*` and `gh.*` for their CLI calls) ask `HostServices.Runner` for a process. Pure parts (`map`, `filter`, `if`, `http`, …) run in-process in milld regardless of the node's runner. External executable parts are launched through the runner the host picks for the **part process** (`local`, or `container` when `plugins.sandbox = "container"`).

## 2. Process groups and kill semantics

`internal/runner/proc` hides the OS differences behind `Start/Interrupt/Kill/Wait/Identity`.

| | macOS | Linux | Windows |
|---|---|---|---|
| Group | `SysProcAttr{Setpgid: true}` → pgid = pid. PTY children: `Setsid` + controlling tty (creack/pty) | same; optionally `Pdeathsig: SIGKILL` on the direct child (helps when milld is SIGKILLed; only covers the direct child, see note) | Job Object per node-run with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`; `CREATE_NEW_PROCESS_GROUP` so `CTRL_BREAK` can be targeted |
| Interrupt (start of grace) | `kill(-pgid, SIGTERM)`; PTY agents first get the registry's `exit_keys` (e.g. `/exit\r`), then SIGTERM | same | `GenerateConsoleCtrlEvent(CTRL_BREAK_EVENT, pid)` for console apps; ConPTY: write `exit_keys`, then close the pseudo console |
| Kill (end of grace) | `kill(-pgid, SIGKILL)` | same | `TerminateJobObject(job, 1)` (replaces rex's `taskkill /T /F`, which its lint config already flags as temporary) |
| Escapees | processes that call `setsid`/daemonize leave the group and survive. Documented; caught only by the container runner. | same, plus optional cgroup v2: if `$XDG_RUNTIME_DIR/systemd` delegation exists, run each node-run in a transient scope (`systemd-run --user --scope`) so `systemctl kill` reaches escapees. Off by default (M11 hardening). | Job Objects catch all descendants unless they break away (blocked by not setting `JOB_OBJECT_LIMIT_BREAKAWAY_OK`) |
| milld dies | children keep running (own group); stdout pipes break, so writes get `SIGPIPE`/`EPIPE`; recovery kills the recorded group if the identity matches ([engine.md](engine.md) §10) | same; `Pdeathsig` kills the direct child sooner | job handle closes → every process in every job dies immediately |
| Identity for recovery | pid + pgid + process start time (`sysctl kern.proc.pid.<pid>` `p_starttime`) | pid + pgid + `/proc/<pid>/stat` field 22 (start time in ticks) | not needed |

Windows assignment race: `os/exec` can't start a process suspended and hand back its thread handle, so the child is assigned to the job right after `Start`. A child that spawns grandchildren in its first microseconds could escape. Acceptable for v0; fix later by creating the process with `windows.CreateProcess` + `CREATE_SUSPENDED`, assign, `ResumeThread`.

Exit status mapping: exit 0 → done; non-zero → the part decides the class (`sh`/`exec`: a failed node with `exit:N` matchable by retry rules, class `user` unless `ok_codes` includes it; agents: through the registry's `errors` rules, spec §29.7); killed by our teardown → cancelled/timeout (ignored exit code); killed by someone else (signal we didn't send) → `transient`, retryable for idempotent parts.

## 3. Environment and arguments

Every process gets a clean, explicit environment:

- Base: the daemon's environment filtered by `env.passthrough` (default: `PATH HOME USER LOGNAME SHELL LANG LC_* TERM TZ TMPDIR SSH_AUTH_SOCK XDG_* GIT_* EDITOR`, plus Windows essentials `SYSTEMROOT COMSPEC PATHEXT USERPROFILE APPDATA LOCALAPPDATA TEMP TMP`). Rationale: milld may have been started from a shell with unrelated tokens in it; those shouldn't leak into every step by accident.
- Pipeline `env`, then node `env` (templated; may reference `secrets.*` handles, which is the **only** way a secret reaches an exec process).
- mill variables: `MILL_RUN_ID`, `MILL_RUN_NUMBER`, `MILL_PIPELINE`, `MILL_NODE_ID`, `MILL_NODE_RUN_ID`, `MILL_ATTEMPT`, `MILL_ITEM_INDEX`, `MILL_WORKSPACE`, `MILL_ARTIFACTS`, `MILL_SCRATCH`, `MILL_OUTPUT` (agents), `MILL_DRY_RUN=1` when dry.
- Argv never contains a secret value: lint rejects `secrets.*` in argv/`run` positions (M9203); at runtime the redactor scans the final argv and refuses to start if a registered secret appears (M9203, defense in depth).

`sh` part: Unix runs `/bin/sh -c <run>`; `shell: bash` (or any argv prefix) overrides. Windows: looks for `sh.exe` from Git for Windows (`%ProgramFiles%\Git\bin\sh.exe`, then PATH), else fails M6101-style with "install Git for Windows or use `exec`/`shell: pwsh`". Open decision D8.

## 4. PTY runner and terminal-only agents

Copied structure from rex's `internal/daemon/pty` (console abstraction, screen mirror, repaint, transcript pause), with a node-run-shaped supervisor:

1. **Start** the agent in a PTY sized `pty.cols × pty.rows` (default 200×50; wide so agents don't wrap tables). One mutex covers "append raw bytes to log + write to vt10x screen + fan out to attached viewers" so attach snapshots never lose or double a chunk (rex invariant).
2. **Prompt delivery** per registry `prompt.via`: `argv` (position), `stdin` (written then closed — not for PTY), `file` (`MILL_PROMPT_FILE`), or `paste` once `ready_regex` matches on the screen and output has been quiet for `ready_settle_ms` (bracketed paste unless `plain_paste`).
3. **Auto-answers.** On each screen change, `auto_answer` rules (`shown` regex over the screen text, `move` keys until `selected` regex matches, `confirm` keys) fire at most once per rule per node-run. Folder-trust rules fire only when the workspace path is inside `trusted_dirs` (rex `CheckAutoTrustDirs`, reused).
4. **State detection** (`agents/detect`), per registry `detect.kind`:

| kind | working | done | asked for input → M6304 |
|---|---|---|---|
| `exit` | process alive | exit code 0 | n/a |
| `hooks` | hook file says `working` | hook says `done`/`stop`, or exit 0 | hook says `waiting`/`needs_input` |
| `heuristic` | `busy_regex` on screen, or output within `idle_ms` | `done_regex` matches, or `MILL_OUTPUT` written and `ready_regex` back, or exit 0 | `ready_regex` back (or idle ≥ `idle_ms` with no busy marker) after work started **and** no `MILL_OUTPUT` and no `done_regex` |

5. **M6304** quotes the last non-empty screen lines (≤ 12, sanitized, redacted) as the question, then tears down (exit keys → SIGTERM → SIGKILL). It never waits.
6. **Attach** (`mill attach run/node`, TUI `enter`): `Subscribe{node_run_id, repaint: true}` → first `NodeOutput` is the serialized screen (rex `Repaint`), then live bytes. View-only: no input intent exists.
7. **Transcript.** Raw bytes in `logs/<nr>.out.log`; final screen in `transcripts/<nr>.screen`; sanitized text for classification only.

Terminal mode is the fallback. Every built-in agent entry uses headless mode where the CLI has one (Claude Code `-p`, Codex `exec`, Gemini `-p`, Aider `--message`, OpenCode `run`, Ollama `run` with stdin).

## 5. Agent registry

Built-in entries live in `internal/agents/builtin.lua` (embedded), translated from rex's `builtin.yaml`, and are merged by `id` with `mill.agent` declarations (scalars replace, `models` merge by model id; spec §11.1).

```go
package agents

type Entry struct {
	ID, Name    string
	Command     []string
	Models      []Model            // id, name, args, price{input,output} per 1M tokens (micro-USD ints after load)
	Effort      map[string][]string // low|medium|high|max → args
	Headless    *Headless          // args, output: stdout-json | stdout-jsonl | stdout-text | file; result_path, usage paths
	Terminal    *Terminal          // args, auto_answer[], exit_keys, ready_settle_ms, plain_paste
	Prompt      PromptSpec         // via: argv(position) | stdin | file | paste
	Detect      DetectSpec
	Usage       UsageSpec          // regex | json_path | hooks
	TrustPrompt *TrustPrompt
	Errors      []ErrorRule        // match/exit → class, code, retry_after regex, help
	Env         map[string]string
	Probe       ProbeSpec          // how `mill agent test` / boot agents.probe check installed + signed in
}
```

**Argv building** (`agents.Build(entry, model, effort, mode, prompt) ProcSpec`): `command + mode args + model args + effort args + prompt placement`. Prompts longer than `prompt.argv_max` (default 32 KiB; Windows command lines cap at 32,767 chars) automatically switch to `file` or `stdin` if the entry supports it, else fail M6102 before spawning.

**Usage extraction:** headless JSON via `json_path` (e.g. Claude's result object `usage.input_tokens`), JSONL streams by summing per-event deltas, text via regex; emitted as `Usage` events. Unknown → tokens 0 with `usage_unknown: true` (budgets then can't see that node; lint warns when a budget is set).

**Error normalization:** the last 64 KiB of stdout+stderr and the exit code are matched against `errors` rules in order; the first match sets class/code/retry_after/help. No match + non-zero exit → class `agent`, code M6300.

**`echo` agent** for tests: an entry whose command is the test helper binary `mill-echo-agent` (built by tests, see [testing.md](testing.md) §3), never shipped in release builds' default registry except as `echo` with `enabled_by_default: false`.

## 6. Structured output pipeline

For every agent node-run with a declared `output` (spec §11.3), the worker runs this state machine:

1. **Prepare.** Convert the node's `output` shorthand to JSON Schema (`schema.FromShorthand`): `string`, `number`, `integer`, `bool`, `[T]`, `{k: T}`, `{enum: [...]}`, `T?` optional, or a full JSON Schema object if it has `$schema`/`type`. (The spec shows the shorthand but never defines it; this is the definition.) Set `MILL_OUTPUT=<scratch>/output.json`. Append to the prompt: *"When you're finished, write your result as JSON matching this schema to $MILL_OUTPUT: <schema compact JSON>"*, unless the entry has native structured output (`headless.schema_arg`, e.g. a `--json-schema` flag), in which case pass the schema there instead.
2. **Collect.** Prefer, in order: native structured result (headless JSON field per `result_path`), then `MILL_OUTPUT` if it exists and parses, then a JSON object fenced in the final message.
3. **Extract.** If nothing parses: run `agent.extract` with the transcript tail (last 32 KiB) and the schema, using `agents.extract` setting (default: the same agent with its cheapest priced model; `off` disables). Counts toward the run budget.
4. **Validate.** Validate against the schema. On failure, **retry once** (same workspace, new attempt counted separately from `retry.max`) with the validation errors appended to the prompt. Second failure → `Fail{class: agent, code: M6301}` with the first three validation errors.
5. **Emit.** The validated object becomes the item's `data`; the transcript is an artifact (`transcript.jsonl` or `.log`); `files` from `MILL_ARTIFACTS`; `ws` carried.

Without `output`, the item is `{text: <final message>, exit_code}` plus usage.

## 7. Workspaces

`internal/workspace` owns creation, snapshots for retries, diff, cleanup and path safety.

**Git worktrees** (all through the `git` CLI; no go-git):

| Step | Command |
|---|---|
| validate repo | `git -C <repo> rev-parse --show-toplevel` (repo must be inside an allowed root; not `/`, not `$HOME` itself) |
| resolve base | `git -C <repo> rev-parse --verify <base>^{commit}` → sha recorded in `WorkspaceCreated` |
| branch name | `with.branch` or `mill/<pipeline>/<number>-<node>[-<i>]`; must pass `git check-ref-format --branch`; collision → `-2`, `-3` |
| create | `git -C <repo> worktree add --no-track -b <branch> <path> <base-sha>` where `<path> = workspace_root/<run-short>/<node>[-<i>]` |
| per-attempt snapshot | `git -C <path> stash create` (tree with uncommitted changes) or `HEAD` if clean; stored as `refs/mill/snap/<run-short>/<nr>` |
| reset for retry | `git -C <path> reset --hard <snap> && git -C <path> clean -fdx -e .mill-keep` |
| diff | `git -C <path> diff --binary <base-sha>` (+ untracked via `git add -N` in a temp index) → artifact `diff.patch` + structured `{files[], insertions, deletions}` |
| cleanup | `git -C <repo> worktree remove --force <path>`; delete the branch only if it was never pushed (`git branch -r --contains` empty) and the run didn't hand it off; remove `refs/mill/snap/<run-short>/*` |

Signing and hooks: mill never passes `--no-verify` or `-c commit.gpgsign=false`; `git.commit` uses the user's config as is (spec §10.5). A signing failure fails the node with the git error and a help line.

Concurrency: `git worktree add` on one repo takes `.git/worktrees` locks; the workspace manager serializes worktree create/remove per repo with an in-process mutex keyed by the repo's git common dir.

**Temp dirs:** `workspace_root/<run-short>/tmp-<node>`; removed per policy.

**Path safety** (`paths.CheckRoot` and `os.Root`), applied to `workspace_root`, `trusted_dirs`, file-part roots, `repo` inputs and artifact materialization:
1. Must be absolute after `~` expansion; reject anything containing a `..` element before cleaning.
2. `filepath.Clean` must be idempotent on it.
3. Reject the filesystem root (and any Windows volume root `C:\`), the home directory itself, and system dirs (`/etc`, `/usr`, `/bin`, `/System`, `C:\Windows`).
4. `filepath.EvalSymlinks`, then re-run 1–3 on the result (a symlink to `/` is still `/`).
5. For access *inside* an allowed root (file parts, artifact copies), open the root with `os.OpenRoot` and do every open/create through it: the stdlib then refuses `..` and symlink escapes at each path step, which closes the check-then-use race.
6. Refusals are M9301 with the path and the rule.

**`workspace: none`** (writing in the main checkout) requires a confirmation recorded per pipeline hash on first run (spec §22); in non-interactive contexts it fails M9101-style `refused` unless the pipeline is trusted and `policy.allow_main_checkout: true`.

## 8. Daemon shutdown and long runs

- `Shutdown{drain: false}` (default for `mill daemon stop`): stop granting slots, then exit within 5 s without tearing down node-runs or writing terminal events. Children lose their pipes and die or are killed on restart by identity; recovery applies spec §15.7 (idempotent → re-run, resumable → resume, else `crashed`). This is "crash, don't guess" applied uniformly: a shutdown is treated like a crash, never like a cancel.
- `Shutdown{drain: true}` (`mill daemon restart --drain`, upgrades): stop granting, wait for working node-runs up to `shutdown.drain_timeout` (10 m), then as above. Queued work survives in the logs and resumes after restart.
- **Sleep (macOS):** with `keep_awake_while_running` (default on for macOS), milld runs `/usr/bin/caffeinate -i -w <milld pid>` while any run is live and stops it when none is, so a closed-lid-on-power laptop doesn't suspend mid-run. Same mechanism rex uses for its screensaver (`-d` there, `-i` here).

## 9. Dry run

`--dry-run` (spec §17.10, §22): parts with `flags.destructive` don't execute; instead their `Open` gets `context.dry_run: true` and built-ins report what they would do (`would open draft PR "fix: …" on acme/app`) as their output item with `dry_run: true`. Non-destructive parts run normally. `sh`/`exec` are not flagged destructive by default but run in workspaces; a node can set `destructive: true` to make dry runs skip it (then its output is `{dry_run: true, would_run: "<argv redacted>"}`). Open decision D17.
