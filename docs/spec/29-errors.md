## 29. Errors and diagnostics

Errors are part of the interface, not an afterthought. Each one says what happened, where, and exactly what to type next. It uses a stable code you can look up, and it looks the same in the CLI, the TUI, JSON output and the logs.

### 29.1 Rules for every error

1. **Say what happened, in plain words.** `pipeline "issue-to-pr" not found`, never `error occurred` or `invalid input`.
2. **Name the thing.** The pipeline, node, run, file, line, part or setting involved.
3. **Give the next command.** A `help:` line with something you can copy and run (`mill daemon restart`, `mill secrets set LINEAR_API_KEY`, `mill rerun issue-to-pr#40 --from build`).
4. **Suggest when we can guess.** Typos in pipeline names, node ids, settings, part ids and flags get `did you mean`, based on edit distance and what exists.
5. **One short headline.** Lowercase start, no trailing period, under 100 characters. Detail goes on the lines below it.
6. **No stack traces by default.** They only appear with `--debug` or `MILL_DEBUG=1`, and they're always in the daemon log.
7. **No blame, no jargon.** Never "you failed to…". Internal terms (goroutine, nil, EOF) never reach the headline.
8. **Never leak secrets.** Every error passes through the redactor before it's printed, logged or sent to a client.
9. **Stable and documented.** Every error has a permanent code and a page you can read with `mill explain <code>`.

### 29.2 What an error looks like

**CLI, human output** (stderr, color when it's a terminal):

```
error[M1201]: pipeline "isue-to-pr" not found
  looked in  .mill/pipelines/ and ~/.config/mill/pipelines/
  did you mean  issue-to-pr
  help: mill ls  ·  mill explain M1201
```

**Lint and Lua diagnostics** point at the exact spot, with the source line and a caret:

```
error[M3012]: unknown node "plann" in expression
   ┌─ .mill/pipelines/issue-to-pr.mill.yaml:41:13
   │
41 │       show: ${{ nodes.plann.item.plan }}
   │                   ^^^^^ no node named "plann"
   │
   = did you mean: nodes.plan.item.plan
   = help: mill lint --fix applies this suggestion
```

```
error[M8004]: unknown setting "max_agents" in mill.setup
   ┌─ ~/.config/mill/init.lua:7:3
   │
 7 │   max_agents = 6,
   │   ^^^^^^^^^^
   │
   = did you mean: max_concurrent_agents
   = note: the previous config is still running; nothing changed
```

**Run failures** summarize the cause instead of dumping logs:

```
✗ issue-to-pr#40 failed at build (loop 3 of 3)
  cause  check failed: `make check` exited 2
  last   --- FAIL: TestSender_429 expected 5 attempts, got 1
  log    mill logs issue-to-pr#40/build  (line 212)
  help   mill rerun issue-to-pr#40 --from build
```

**Cause chains** show up to three levels, innermost last. `--verbose` shows the full chain:

```
error[M5201]: part linear.issue.create failed
  caused by: Linear API returned 404: team "ENGG" not found   [linear/E_TEAM_NOT_FOUND]
  did you mean  ENG
  help: mill part info linear.issue.create  ·  list teams with  mill part call linear teams
```

**Warnings** use the same format with `warning[Mxxxx]` and never change the exit code, unless `--strict` is given.

### 29.3 JSON output

With `--json`, a failing command prints one JSON object on **stdout** (so scripts parse one stream) and still exits non-zero:

```json
{
  "error": {
    "code": "M3012",
    "class": "user",
    "message": "unknown node \"plann\" in expression",
    "where": { "file": ".mill/pipelines/issue-to-pr.mill.yaml", "line": 41, "col": 13, "node": "scope" },
    "suggestions": ["nodes.plan.item.plan"],
    "help": ["mill lint --fix"],
    "retryable": false,
    "cause": null,
    "docs": "mill explain M3012"
  }
}
```

The same object is what the daemon protocol carries in an `Error` event (`data.error`), what the MCP server returns, and what the gateway returns with the matching HTTP status. One shape everywhere.

### 29.4 Error codes

Codes are `M` plus four digits, grouped by area. A code is never reused or renumbered. If its meaning changes, it gets a new code.

| Range | Area | Examples |
|---|---|---|
| `M0xxx` | Internal: a bug in mill | `M0001` unexpected internal error |
| `M1xxx` | CLI, selectors, arguments | `M1201` pipeline not found · `M1203` ambiguous run selector · `M1301` unknown flag |
| `M2xxx` | Files and pipelines | `M2101` YAML parse error · `M2201` conflicting edit (file changed since you loaded it) · `M2203` lint errors, not saved |
| `M3xxx` | Lint and expressions | `M3012` unknown node in expression · `M3020` cycle outside a loop · `M3030` branch port not connected · `M3101` CEL type mismatch · `M3204` destructive part not allowed · `M3206` waits for a person. The full list is in §18.7.9. |
| `M4xxx` | Runs and scheduling | `M4101` budget exceeded · `M4201` node timed out · `M4301` crashed during daemon restart |
| `M5xxx` | Parts and plugins | `M5101` plugin not found · `M5102` version conflict · `M5201` part failed · `M5301` checksum mismatch |
| `M6xxx` | Agents | `M6101` agent CLI not installed · `M6201` rate limited · `M6202` auth expired · `M6301` output didn't match schema · `M6304` agent asked for input |
| `M7xxx` | Daemon and protocol | `M7001` daemon unreachable · `M7003` daemon older than client · `M7101` line too long |
| `M8xxx` | Lua config | `M8001` Lua syntax error · `M8004` unknown setting · `M8101` hook error |
| `M9xxx` | Security, trust, secrets | `M9101` pipeline not trusted · `M9201` secret not declared · `M9202` secret missing · `M9301` path refused |

The boot splash's codes (`M0002`, `M2001`, `M4401`, `M7002`, `M7004`, `M7006`) are listed with their fix hints in §18.13.

Plugins report their own codes namespaced by plugin (`linear/E_TEAM_NOT_FOUND`), and mill wraps them as the cause of an `M5201`.

### 29.5 `mill explain`

```
mill explain M6201
mill explain --list [--area agents]
```

Each code has a short page shipped inside the binary: what it means, the common causes, how to fix it, and an example. The same pages are generated into `docs/errors/`. In the TUI, `gx` on any error opens its page.

```
M6201  agent rate limited

The agent's provider refused the request because too many were sent.

Common causes
  · several agent nodes running at once against the same account
  · a fan-out wider than your plan allows

What mill does
  Retries automatically, waiting for the provider's Retry-After when given.

Fix
  · lower max_concurrent_agents, or set budget.max_agents on the pipeline
  · add `retry: { on: [class:rate_limited], max: 5 }` to the node
```

### 29.6 Error classes

Every error has a **class**. Retry policies, exit codes and the TUI all key off the class, not the message text.

| Class | Meaning | Retried by default | Exit code |
|---|---|---|---|
| `user` | Bad input: arguments, YAML, expressions, config | no | 5 |
| `not_found` | Selector, pipeline, node, part or file doesn't exist | no | 2 |
| `ambiguous` | Selector matches several things (the error lists them) | no | 3 |
| `conflict` | Something changed underneath you (stale edit, slug taken) | no | 6 |
| `refused` | Policy, trust, confirmation, or path rules said no | no | 6 |
| `unreachable` | The daemon or a remote runner can't be reached | no | 4 |
| `transient` | Network blips, 5xx responses, a part restarting | yes, with backoff | 1 |
| `rate_limited` | Provider throttling; carries `retry_after` when known | yes, honoring it | 1 |
| `auth` | Expired or invalid credentials | no | 1 |
| `timeout` | A node or `mill wait` ran out of time | node: per policy · `wait`: no | 7 |
| `budget` | Tokens, money, time or node limits hit | no | 9 |
| `cancelled` | Stopped by a person or a policy | no | 1 |
| `agent` | Agent-specific: context too long, refused the task, malformed output | per node policy | 1 |
| `internal` | A bug in mill | no | 1 |

### 29.7 Agent errors are normalized

Agent CLIs fail in very different ways. Each agent registry entry (§11.1) can declare `errors` patterns that map exit codes and output text to classes:

```lua
mill.agent "claude" {
  errors = {
    { match = "rate limit|429",        class = "rate_limited", retry_after = "retry after (%d+)s" },
    { match = "please run /login",     class = "auth",         help = "run `claude` once to sign in" },
    { match = "context window",        class = "agent",        code = "M6303" },
    { exit  = 130,                     class = "cancelled" },
  },
}
```

So a pipeline can say `retry: { on: [class:rate_limited] }` and have it work for every agent, and the run view shows "claude is rate limited, retrying in 40s" instead of a wall of terminal output.

### 29.8 Errors in the TUI

- **Never modal, never steal focus.** An error appears in the statusline as one red line with its code: `M5201 linear.issue.create: team "ENGG" not found  gx explain`. It fades after a few seconds, or when you press a key.
- **`:messages`** lists every error and warning from this session, newest first, the way vim does.
- **Lint uses a quickfix list.** `:lint` fills it; `]e` / `[e` jump between problems in the editor, `!` (or `:copen`) shows the list, `ga` applies a fix and `gA` every safe one, and each problem node gets a red `✗` or amber `!` in its title and border (§18.7.9).
- **Failed pipelines** show the cause as their row's text on the board. In the run popup the failed node's row is red; `enter` on it shows the full cause chain and the log lines around the failure. `gx` opens the explain page.
- **Config errors** show as one line under the board's header until fixed: `init.lua:7 unknown setting "max_agents" (running previous config)`.

### 29.9 Common failures, designed in advance

| Situation | What mill says and does |
|---|---|
| Daemon not running | Starts it automatically. If that fails: `M7001 daemon unreachable at <socket>`, the reason, and `help: mill daemon logs`. |
| Client and daemon versions differ | `M7003 daemon is older than this client (0.4.2 < 0.5.0)` · `help: mill daemon restart` (runs keep their state across the restart) |
| Ambiguous run selector | `M1203` lists every match with its short id, pipeline and start time. |
| Pipeline edited elsewhere | `M2201 issue-to-pr changed since you loaded it` · TUI offers to reload, diff or overwrite. |
| Agent asks a question mid-task | Fails the node with `M6304`, quoting the question · `help: add an auto_answer rule, or make the prompt answer it up front`. |
| Secret missing at run start | Fails before anything runs: `M9202 LINEAR_API_KEY isn't set` · `help: mill secrets set LINEAR_API_KEY`. |
| Agent CLI not installed | `M6101 codex not found on PATH` · `help: install it, or mill import --map-agent codex=claude`. |
| Plugin checksum changed | Refuses to start it: `M5301` with both hashes · `help: mill plugin update acme/linear` to accept the new version. |
| Budget hit mid-run | `M4101 issue-to-pr#42 stopped: token budget 2,000,000 reached at build` · `help: mill rerun issue-to-pr#42 --from build --budget-tokens 3m`. |
| Something inside mill broke | `M0001 internal error: this is a bug in mill` · `help: mill doctor --bundle` writes a redacted report to attach to an issue. |

### 29.10 `mill doctor`

`mill doctor` checks the whole setup and prints one line per check, with a fix command for each problem. It starts with the boot splash's steps (§18.13) in the same `[ STATUS ] step · finding` form, then goes deeper: daemon reachable and same version; config loads; every plugin present, pinned and unchanged; every agent in use is installed and signed in; declared secrets resolve; workspace root writable; disk space for the run store; git and container runtimes available if used; gateway port free.

`mill doctor --bundle` writes a single archive for bug reports: versions, OS, effective settings with secrets stripped, the daemon log tail, and (if you name one) a run's event log. Everything goes through the redactor, and the command prints what it included before writing.

### 29.11 Keeping errors good

- Every code has a golden test (CLI output, JSON output) and an explain page. CI fails if a code has no docs, a doc has no code, or a code is defined but never used.
- New errors go through review like UI changes: the headline, the help line and the suggestion are checked against the rules in §29.1.
- `mill explain --list --unused` and `--undocumented` exist for maintainers.


---

[← spec index](index.md)
