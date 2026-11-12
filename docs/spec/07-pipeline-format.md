## 7. Pipeline file format

### 7.1 Where pipelines live

- **Project pipelines:** `.mill/pipelines/*.mill.yaml` in a repo. Versioned with the code they act on. A project you haven't trusted (a fresh clone) has its pipelines listed but disarmed: no triggers, no manual runs, until `mill trust .` (§22).
- **User pipelines:** `~/.config/mill/pipelines/*.mill.yaml`.
- **Ad-hoc:** pipe expressions (§14), optionally saved to a file with `--save`.

A pipeline's id is its `name`, unique within its scope. Project scope shadows user scope.

### 7.2 Top-level keys

| Key | Required | Meaning |
|---|---|---|
| `mill` | yes | File format version. `1`. |
| `name` | yes | Pipeline id: lowercase, kebab-case. |
| `description` | no | One line. Shown in the library and `mill ls`. |
| `tags` | no | Free labels for filtering. |
| `on` | no | One or more triggers (§13). No triggers means manual or piped runs only. |
| `inputs` | no | Named inputs with a type, a default and a description. Prompted for in the TUI, passed with `--input` on the CLI, or bound from a trigger. |
| `outputs` | no | Named expressions that become the run's result. Written to stdout when the pipeline is used as a filter, and returned to callers (sub-pipelines, MCP, webhooks). |
| `env` | no | Environment variables for every node, added to the allowlisted base environment (§12.1). |
| `secrets` | no | The secret names this pipeline may use (§21). Anything not declared can't be resolved. |
| `budget` | no | Run limits (§15.5). |
| `policy` | no | Safety rules: which destructive capabilities the pipeline may use (`allow_destructive`), allowed runners, network rules, `fail_fast` (default `true`: cancel in-flight siblings when a node fails, §15.8), `priority` (default 0, §15.2). |
| `defaults` | no | Default `runner`, `retry`, `timeout`, `workspace` and `agent` settings for nodes. Without a `retry`, the default retry policy applies (§15.3). |
| `nodes` | yes | Map of node id → node definition. |
| `edges` | no | Explicit edges. Usually inferred (§7.4). |

### 7.3 Node keys

| Key | Meaning |
|---|---|
| `uses` | The part and capability: `sh`, `agent`, `http`, `assert`, `mcp:<server>/<tool>`, `part:<id>@<version>`. |
| `with` | Configuration for the part. Validated against the part's JSON schema. |
| `in` | Where input items come from: an expression or a list of node ports. Defaults to the inferred upstream. |
| `mode` | `batch`, `each` or `stream` (§6.2). |
| `concurrency` | Max parallel node-runs in `each` mode. |
| `when` | Expression. If false, the node-run is `skipped`. |
| `needs` | Extra ordering dependencies with no data flow. |
| `runner` | Where to execute (§12). |
| `workspace` | `inherit` (default), `git`, `temp`, or `none`. |
| `retry` | `max`, `backoff`, `on` (exit codes or error classes), `keep_workspace`. Default: the policy in §15.3. |
| `timeout` | Wall-clock limit per node-run. |
| `cache` | `true`, or a key expression. Reuse output when the key matches. |
| `on_error` | `fail` (default), `continue`, or `route` (send to the `error` port). |
| `discard` | Output ports whose items are dropped on purpose (`[false]`). Without it, an unconnected branch port is a lint warning (`M3030`). |
| `output` | The schema the node's output items must match: the shorthand below, or a full JSON Schema object. |
| `env` | Environment variables for this node, added after the pipeline's `env` (§12.1). The only way a secret reaches an `sh` or `exec` process. |
| `destructive` | `true` marks this node as destructive: `--dry-run` skips it, and it needs `policy.allow_destructive` like a destructive part (§22). For `sh` and `exec` steps that push, post or delete. |
| `io` | For exec-style nodes: `envelope`, `plain` or `text` (§6.1). |

**Output shorthand.** `output` is usually written in a short form that mill turns into JSON Schema:

| Shorthand | Means |
|---|---|
| `string`, `number`, `integer`, `bool` | That JSON type |
| `[T]` | A list of `T` |
| `{ field: T, … }` | An object with those fields, all required |
| `T?` | Optional (the field may be missing) |
| `{ enum: [a, b, c] }` | One of these values |

An object with `$schema` or `type` at the top is read as full JSON Schema instead. For agents, this schema is what mill asks the agent to write (§11.3).

### 7.4 Edge inference

A node's **input** (the item stream that `item`, `items` and the part's stdin see) and its **order** (what it waits for) are separate things.

1. **Every reference orders.** If node `b` uses `${{ nodes.a.… }}`, `b` waits for `a` and can read `a`'s outputs through `nodes.a`. That alone doesn't send `a`'s items to `b`.
2. **The input** comes from, in this order:
   - `in:` if given: a node, `node.port`, a list of them (their items are concatenated), or an expression;
   - else explicit `edges:` into the node;
   - else the **primary upstream**: of the nodes `b` references, the one that every other referenced node is upstream of. In §7.5, `pr` references `review` and `plan`; `plan` is upstream of `review`, so `pr` reads `review`'s items.
3. **No primary upstream is a lint error.** When the referenced nodes run in parallel (no single one is downstream of the others), lint fails with `M3021 ambiguous input` and offers `in: [a, b]` or a `merge` node.
4. **With an explicit `in:`**, references to nodes that aren't already upstream add an ordering edge only, never items.
5. `needs:` adds ordering edges only. A node that references nothing and has no `in` is a source: the first one in file order gets the run's input items (from `--stdin` or the trigger), the others none.

Write `edges:` explicitly only to:
- name a non-default port (`check.false -> fix`), or
- order nodes that share no data (prefer `needs:` for that).

The compact edge syntax allows chains: `- a -> b -> c`. `mill show <p> --graph` prints both kinds of edge, so inference is never invisible.

### 7.5 Full example

```yaml
mill: 1
name: issue-to-pr
description: Turn a labeled GitHub issue into a tested, reviewed draft pull request.
tags: [code, github]

on:
  webhook:
    path: /github
    verify: github-hmac
    secret: GITHUB_WEBHOOK_SECRET
    filter: ${{ event.action == "labeled" && event.label.name == "agent" }}

inputs:
  issue: { type: object, from: ${{ event.issue }} }
  repo:  { type: string, default: ~/code/app }

secrets: [GITHUB_TOKEN, GITHUB_WEBHOOK_SECRET, SLACK_TOKEN]

budget:
  max_tokens: 2_000_000
  max_cost_usd: 15
  max_wall: 1h
  max_agents: 3

policy:
  allow_destructive: [git.push, gh.pr, gh.comment]   # anything destructive not listed fails lint

defaults:
  agent: { id: claude, model: sonnet }
  retry: { max: 2, on: [class:transient, class:rate_limited] }

nodes:
  ws:
    uses: git.worktree
    with:
      repo: ${{ inputs.repo }}
      branch: mill/issue-${{ inputs.issue.number }}

  plan:
    uses: agent
    in: ws                             # items carry the workspace on to build (§6.4)
    with:
      model: opus
      effort: high
      prompt: |
        Issue #${{ inputs.issue.number }}: ${{ inputs.issue.title }}
        ${{ inputs.issue.body }}
        Write an implementation plan. Do not edit code.
    output:
      plan: string
      files: [string]
      risk: { enum: [low, medium, high] }

  scope:
    uses: assert
    with:
      that: ${{ nodes.plan.item.risk != "high" && size(nodes.plan.item.files) <= 8 }}
      else: route                      # out-of-scope plans go to the `else` port

  hand-back:
    uses: handoff
    in: scope.else
    with:
      to: gh.comment
      issue: ${{ inputs.issue.number }}
      body: |
        mill stopped before writing code: the plan is high risk or touches more than 8 files.
        ${{ nodes.plan.item.plan }}

  build:
    uses: agent.loop
    in: scope
    with:
      prompt: "Implement this plan:\n${{ nodes.plan.item.plan }}"
      until: { sh: "make check" }
      max_iters: 3

  review:
    uses: agent.review
    with:
      agent: { id: codex }
      diff: ${{ nodes.build.item.diff }}

  pr:
    uses: gh.pr
    when: ${{ nodes.review.item.verdict == "approve" }}
    with:
      draft: true
      title: "fix: ${{ inputs.issue.title }} (#${{ inputs.issue.number }})"
      body: ${{ nodes.plan.item.plan }}

  ping:
    uses: mcp:slack/send_message
    with:
      channel: "#dev"
      text: "Draft PR ready: ${{ nodes.pr.item.url }}"

outputs:
  pr_url: ${{ nodes.pr.item.url }}
```

Nothing in this pipeline waits. An out-of-scope plan becomes a comment on the issue and that branch ends. A person deals with it on GitHub, and relabeling the issue starts a fresh run. The PR is opened as a draft, so merging stays where it belongs: with people, outside mill.

### 7.6 Layout sidecar

Canvas positions live in `<name>.mill.layout.json` next to the pipeline. If it's missing, auto-layout is used. It's safe to delete or gitignore.

It only holds the nodes you placed by hand, as layout slots (column, row), never cell coordinates, so it survives zoom levels and terminal sizes:

```json
{ "version": 1, "pins": { "judge": [4, 1], "no-winner": [6, 2] } }
```

Everything else the editor remembers (zoom, the minimap, the last selection) is a view preference in `tui-state.json`, not part of the pipeline (§18.7.1).

### 7.7 Validation (`mill lint`)

Checks: schema; unknown parts or versions; unbound references; ambiguous inputs (`M3021`); cycles outside `loop`; type mismatches between a node's declared `output` and a consumer's expectations; secrets used but not declared; policy violations (for example a destructive part like `git.push` that `policy.allow_destructive` doesn't list); unreachable nodes (`M3024`); runners not allowed by policy. Exits non-zero with file:line:column diagnostics, in text or `--json`. The full list of checks, with their codes and quick fixes, is in §18.7.9; `mill lint --fix` applies the safe fixes.


---

[← spec index](index.md)
