## 10. Built-in parts catalog

All built-ins use the public parts protocol (§9.3) and can be replaced.

### 10.1 Execution
| Part | What it does |
|---|---|
| `sh` | Run a shell command. stdin = input items (per `io`), stdout = output items. Non-zero exit fails the node unless `ok_codes` includes it. |
| `exec` | Like `sh` but with an argv list and no shell. Safer for templated arguments. |
| `check` | Run a command as a pass/fail test. Output: `passed`, `exit_code`, captured output as an artifact. Routes to `true`/`false` ports. |
| `lua` | Sandboxed Lua transform over items. |

### 10.2 Data
| Part | What it does |
|---|---|
| `map` | Apply an expression to each item. |
| `filter` | Keep items where an expression is true. |
| `reduce` | Fold items into one. |
| `sort`, `uniq`, `take`, `skip`, `batch` | Stream utilities. `batch` groups N items or by time window. |
| `merge` | Combine streams: `all` (wait for every input), `any` (first arrival), `zip`, `concat`. |
| `split` | Explode a list field into separate items. |
| `jq` | Run a jq program (built in, no external jq required). |
| `template` | Render text or files from a template and items. |
| `parse` | JSON, YAML, CSV, XML, HTML (CSS selectors), Markdown front-matter. |

### 10.3 Control flow
| Part | What it does |
|---|---|
| `if` | Route each item to `true` or `false`. |
| `switch` | Route by expression to `case.<name>` ports, with `default`. |
| `loop` | Re-run a subgraph (its `body`) until a condition holds or `max_iters` is reached. The last try's output leaves on `out` when `until` held, or on `exhausted` when it didn't. |
| `assert` | Check an expression on each item. `else: fail` (default) fails the node and shows the expression and its values in the error; `else: route` sends the item to the `else` port; `else: drop` discards it. |
| `handoff` | Pass work to a person or another tool through any part (`gh.comment`, `mcp:slack/send_message`, `linear.issue.create`, an external session manager…), record where it went as the node's output, and end that branch. mill never waits for the answer. |
| `wait` | Sleep, or wait until a given time. |
| `pipeline` | Run another pipeline as a sub-pipeline. Its `outputs` become items. |
| `fail`, `noop` | Explicit failure; placeholder. |

A loop names its body; the body's first node reads from the loop's `body` port, and everything downstream reads from the loop:

```yaml
retry:
  uses: loop
  in: found.true
  with:
    body: [patch, test]
    until: ${{ nodes.test.item.passed }}
    max_iters: 3
patch: { uses: agent, in: retry.body, with: { prompt: "Fix it. Last try: ${{ loop.prev }}" } }
test:  { uses: check, in: patch, with: { sh: make check } }
pr:    { uses: gh.pr, in: retry }                      # after a passing try
give-up: { uses: handoff, in: retry.exhausted, with: { to: gh.issue } }
```

### 10.4 Agents (see §11)
| Part | What it does |
|---|---|
| `agent` | Run one agent, unattended. |
| `agent.fanout` | Same task × N agents, models or seeds, each in its own workspace. |
| `agent.judge` | Compare candidate items against a rubric and pick one (or rank them). |
| `agent.review` | Independent review by an agent that never saw the producer's session. Verdict plus comments. |
| `agent.loop` | Run an agent until a check passes, feeding each failure back in. |
| `agent.extract` | Cheap headless pass that turns free text into JSON matching a schema. |

### 10.5 Code and git
| Part | What it does |
|---|---|
| `git.worktree` | Create a branch and worktree for the run or for each item. |
| `git.commit`, `git.push` | Commit and push the workspace. Uses the user's existing signing setup; never bypasses it. |
| `git.diff` | Diff a workspace against its base, as an artifact and as structured data. |
| `gh.pr`, `gh.issue`, `gh.comment` | GitHub operations (through the `gh` CLI or the API with a token). |
| `gl.mr` | GitLab equivalent (part, not built in at v1). |

### 10.6 I/O and network
| Part | What it does |
|---|---|
| `http` | Any HTTP request. Response becomes the item. Pagination helpers. |
| `openapi` | Generated nodes from an OpenAPI spec. |
| `mcp` | Call a tool on an MCP server. |
| `file.read`, `file.write`, `file.append`, `file.glob` | Local files inside allowed roots. |
| `stdin`, `stdout` | Pipeline-level pipe endpoints (§14). |
| `notify` | Desktop notification, terminal bell or sound. |
| `n8n.call`, `zapier.call` | Call an n8n webhook or a Zapier Catch Hook, optionally waiting for a callback (§23). |


---

[← spec index](index.md)
