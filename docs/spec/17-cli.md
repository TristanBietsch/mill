## 17. CLI reference

### 17.1 Commands

```
mill                                   # TUI
mill pipe '<expr>' [--save name] [--dry-run] [--stdin] [--each]

mill new [<name>] [--from template|file.json|describe:"plain English"] [--trigger spec] [--agent id/model]
         [--steps a,b,c] [--budget small|medium|large] [--allow part,...] [--on-failure target] [--dry-run]   # no flags: the wizard
mill ls [--filter expr] [--group by] [--sort by] [--json]
mill show <pipeline> [--outline|--yaml|--graph]
mill render <pipeline|run> [--format text|svg|mermaid] [-w cols] [-h rows]
mill lint [file...] [--json] [--strict] [--fix] [--code Mxxxx --node n]   # --fix: safe fixes only
mill edit <pipeline>                   # opens $EDITOR
mill estimate <pipeline> [--json]      # cost and time per node and per run

mill node add <pipeline> <id> --uses <part> [--after n[.port] | --before n | --branch n.port] [--with k=v]...
mill node set <pipeline>/<node> <key>=<value>...   # value @file reads a file
mill node unset <pipeline>/<node> <key>...
mill node rm <pipeline>/<node> [--heal] | mill node mv <pipeline>/<old> <new>   # mv updates every reference
mill node cp <pipeline>/<node> [<new>] [--after n[.port]]
mill node run <pipeline>/<node> [--sample | --from-run <run>] [--json]          # one node, against its input
mill node pin <pipeline>/<node> --from-run <run> | --file items.jsonl          # pin a sample
mill edge add <pipeline> <from>[.port] <to> [--needs] | mill edge rm <pipeline> <from>[.port] <to>
mill refs <pipeline>/<node>            # every field, output and edge that uses a node
mill pipeline set <pipeline> <key>=<value>...      # budget.max_cost_usd=15, policy.allow_destructive+=gh.pr
mill wrap <pipeline> loop <node>... [--until expr] [--max n] | mill unwrap <pipeline>/<node>
mill extract <pipeline> <node>... --as <name>      # selection → sub-pipeline
mill pattern ls | add <pipeline> <pattern> --after <n>[.port] | save <pipeline> <node>... --as <name>
mill layout <pipeline> pin <node> <col>,<row> | reset [<node>...]

mill watch [<run>|--all] [--json]      # line-per-event live view, no screen control
mill prompt [--format f]               # tiny status segment for PS1 / tmux / starship

mill run <pipeline> [--input k=v]... [--stdin] [--each] [--text]
                    [--wait|--detach] [--budget-tokens n] [--json]
                    [--local] [--dry-run]
mill runs [--pipeline p] [--state s] [--since 24h] [--store dir] [--json]
mill status [--json]                   # one-line summary; exit 1 if anything needs input
mill logs <run>[/<node>] [-f] [-n lines] [--raw]
mill items <run>/<node> [--port p] [--json]
mill wait <run> [--until state] [--timeout d]
mill attach <run>/<node> [--no-replay]      # view-only
mill cancel <run> | mill rm <run> | mill prune [--older 30d] [--dry-run] [--yes]
mill rerun <run> [--from node] | mill replay <run> --node n

mill trigger ls | enable | disable | fire <pipeline> [trigger]
mill trigger add <pipeline> <kind> [--with k=v]... | mill trigger rm <pipeline> <kind>
mill hooks install | uninstall         # git hooks for git triggers
mill events [--json]                   # stream daemon events

mill plugin add | ls | info | update | rm | search | sync [--frozen]
mill plugin new | dev | test | pack | publish
mill part ls | info <part>              # capabilities from all plugins
mill import <src> [--scope s] [--map-agent a=b] [--dry-run]
mill export <pipeline> [--dir|--gist|--to n8n|mermaid|gha] [--parametrize]
mill trust <file|dir> | mill lua-types
mill agent ls | test <id> [--model m]  # check an agent CLI works
mill mcp ls                            # configured MCP servers and their tools
mill mcp serve                         # run mill as an MCP server (stdio)

mill secrets set | ls | rm | test
mill cache ls | clear
mill config list | get | set | reset | edit
mill daemon start | stop | status | restart | logs [-f]
mill import n8n <file> | mill export <pipeline> --to n8n|mermaid|gha
mill stats [--since 30d] [--json] | mill digest [--date d]
mill setup | doctor | update | uninstall
mill completion bash|zsh|fish
```

### 17.2 Exit codes

Exit codes follow the error class (§29.6), so scripts can branch on them without parsing messages.

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic error, or the run failed; `mill status` when anything needs input; `mill doctor` when a check fails |
| 2 | Selector not found |
| 3 | Ambiguous selector |
| 4 | Daemon unreachable |
| 5 | Invalid arguments or lint failure |
| 6 | Operation refused (policy, trust, confirmation required in a non-interactive shell) |
| 7 | Timed out (`mill wait`, `--timeout`) |
| 8 | Reserved |
| 9 | Budget exceeded |

### 17.3 Selectors

| Form | Example | Rule |
|---|---|---|
| Full id | `0b9e…` | Exact match |
| Pipeline and number | `issue-to-pr#42` | Run 42 of that pipeline |
| Latest | `issue-to-pr#last` | Most recent run |
| Short id | `0b9e` | Four or more hex digits, prefix match |
| Node | `issue-to-pr#42/build` | A node within a run; with `[n]` for a specific item in `each` mode |

### 17.4 Conventions

- Every listing command accepts `--json`. Human tables are for people; JSON is the stable interface.
- Destructive commands ask for confirmation in a terminal and require `--yes` otherwise (exit 6 without it).
- `--dry-run` is available on every command that changes state.

### 17.5 Working without the TUI

`mill` with no arguments opens the TUI. Every subcommand is a plain CLI tool that does one thing, prints, and exits. The CLI isn't a fallback: it's a complete product on its own, for SSH sessions, CI, cron, tmux panes, scripts, agents, and people who'd rather type. Anything you can do in the TUI, you can do here.

Each TUI screen has a CLI equivalent:

| TUI | CLI |
|---|---|
| Board | `mill ls` (`--filter`, `--group`, `--sort`, `--view`), `mill status` |
| Run popup | `mill render <run>`, `mill watch <run>`, `mill items <run>/<node>` |
| Log (in the run popup) | `mill logs <run> [-f] [--node n]` |
| Agent screen | `mill attach <run>/<node>` (view-only) |
| Editor | `mill show <p> --outline`, `mill node …`, `mill edge …`, `mill pipeline set`, `mill wrap`, `mill extract`, `mill pattern`, `mill lint --fix`, `mill refs`, `mill estimate`, `mill edit` (the full map is §18.7.16) |
| Wizard | `mill new` (prompts in the terminal, or flags) |
| `λ` prompt | `mill run <p> --input k=v`, `mill pipe '…'` |
| Plugins | `mill plugin ls`, `mill part ls`, `mill agent ls`, `mill mcp ls` |
| Settings | `mill config` |
| Stats | `mill stats` |

### 17.6 Output modes

| Mode | When | What you get |
|---|---|---|
| **Pretty** | stdout is a terminal | Aligned tables, color, state glyphs (`✓ ✗ ◈ ◌ ⠋`) |
| **Plain** | not a terminal, `--plain`, or `NO_COLOR` set | Same columns, no color. Glyphs replaced by words if `MILL_ASCII=1`. |
| **JSON** | `--json` | One JSON document. The stable interface for scripts. |
| **JSONL** | `--json` on streaming commands (`watch`, `events`, `logs -f`, `run --each`) | One object per line, flushed immediately |

Table rules: column order is stable and new columns are only ever added on the right, so `cut` and `awk` scripts keep working. `--fields id,state,cost` picks columns. Times are relative in pretty mode (`12m ago`) and RFC 3339 in plain and JSON modes.

### 17.7 What the commands print

All examples use synthetic data.

**`mill ls`**
```
NAME            TRIGGER             LAST RUN         NEXT       RUNS
issue-to-pr     webhook /github     ✓ done   41m     -          42
nightly-sweep   cron 0 3 * * *      ✗ failed 9h      in 14h     118
morning-brief   cron 30 7 * * 1-5   ✓ done   4h      tomorrow   63
best-of-3       manual              - never          -          0
```

**`mill runs`**
```
RUN                 STATE         NODES   TOKENS   COST    STARTED    DURATION
nightly-sweep#118   ✗ failed      3/5     95K      $0.31   9h ago     6m
issue-to-pr#42      ↻ retrying    4/9     812K     $4.10   12m ago    12m
inbox-triage#913    ⠋ working     2/4     3K       $0.01   12s ago    12s
morning-brief#63    ✓ done        5/5     40K      $0.08   4h ago     48s
```

**`mill status`**: one line, fast, made for prompts and scripts. Exits 1 if any pipeline's latest run failed.
```
≋ 1 failed · 1 retrying · 1 working · 14 done today · $6.20 today
```

**`mill show issue-to-pr --outline`**
```
issue-to-pr · 9 nodes · webhook /github
⌁ webhook /github
└─ ⑂ ws          git.worktree
   └─ ◆ plan         agent claude/opus
      └─ ⊢ scope        assert risk != high, ≤ 8 files
         ├─ ↻ build        agent.loop claude/sonnet · until make check
         │  └─ ◇ review       agent.review codex
         │     └─ ⇡ pr           gh.pr draft (when approved)
         │        └─ ◎ ping         mcp:slack/send_message
         └─ ↪ hand-back    handoff gh.comment (else)
```
Joins get a `◀ joins: a, b` note on the merging node.

**`mill render issue-to-pr#42`**: the same box-and-arrow canvas the TUI draws, printed once as text with live states, at a fixed size (`-w`, `-h`). Deterministic, so it can go in a PR comment, a README, or a CI log. `--format svg` and `--format mermaid` use the same layout engine.

**`mill watch issue-to-pr#42`**: a live view that only appends lines and never redraws the screen, so it works in CI logs, `tee`, tmux panes and slow SSH connections.
```
12:04:01  issue-to-pr#42  ws        ✓ done        0.8s
12:09:40  issue-to-pr#42  plan      ✓ done        5m38s   210K tk   $0.94
12:09:40  issue-to-pr#42  scope     ✓ done        risk medium · 3 files
12:11:02  issue-to-pr#42  build     ↻ retrying    rate limited · next try in 40s (2 of 3)
```
`mill watch --all` follows every run. `--json` emits the same as JSONL.

### 17.8 Building and editing pipelines from the CLI

Pipelines are YAML files, so `$EDITOR` always works. The `node` and `edge` commands add safe, scriptable edits on top:

```sh
mill new triage --describe "label new GitHub issues and post a summary to Slack"
mill node add triage fetch  --uses gh.issue --with state=open --with since=1d
mill node add triage label  --uses agent --after fetch \
    --with model=haiku --with prompt=@prompts/label.md
mill node set triage/label mode=each concurrency=4
mill node add triage post   --uses mcp:slack/send_message --after label --with channel="#triage"
mill lint triage && mill show triage --outline
```

- Edits go through the daemon (`SavePipeline` with the file's current hash), so a CLI edit and an open TUI never overwrite each other. A stale edit fails with `conflict` (exit 6) and the change is not applied.
- The YAML is edited in place, keeping comments, key order and formatting. You can keep hand-editing the same file.
- `@path` reads a value from a file, which keeps long prompts in their own Markdown files.
- Every edit runs `lint` and refuses to save a broken pipeline unless `--force` is given.
- Every editor action has one of these commands, built on the same edit package (§18.7.16).

### 17.9 Running and inspecting from the CLI

```sh
mill run issue-to-pr --input issue=@issue.json --detach   # prints: issue-to-pr#43
mill watch issue-to-pr#43                                  # follow along
mill logs issue-to-pr#43/build -f                          # one node's output
mill attach issue-to-pr#43/build                           # watch the agent's screen, view-only (Ctrl+] to leave)
mill items issue-to-pr#43/plan --json | jq .plan           # what a node produced
mill wait issue-to-pr#43 --until done --timeout 1h && mill items issue-to-pr#43/pr --json
mill rerun issue-to-pr#40 --from build                     # after fixing whatever made it fail
```

There are no approve or reply commands. Runs never wait on a person (§2).

### 17.10 No daemon: `--local`

`mill run <p> --local` runs a single pipeline in-process: no daemon, no socket, nothing left running afterward. The run is written to the normal run store, so `mill runs` and `mill logs` see it later.

- Supported: every node part, runner, budget, cache and the full stdin/stdout piping.
- Not supported: triggers (it's a one-shot run) and watching from another terminal.
- `--dry-run` runs everything except destructive parts, which report what they would have done instead. It's the way to test a pipeline change in CI before it can push, post or open anything.

This is the mode for CI jobs, containers and cron entries on machines that shouldn't run a daemon.

### 17.11 Shell and terminal integration

- **Prompt segment:** `mill prompt` prints `≋✗1` when one pipeline's latest run failed, and nothing otherwise. It reads a small status file `milld` rewrites on every change, so it never touches the socket and costs about a millisecond. Ready-made snippets for bash, zsh, fish, starship and tmux.
- **Completion:** `mill completion bash|zsh|fish` completes commands, flags, pipeline names, node ids, run selectors and part ids (dynamic values come from the status file and the index).
- **Notifications:** `milld` sends a desktop notification when a run fails (the `notify` setting), and `mill.on("run_updated", …)` hooks can send failures anywhere else.
- **Composability:** everything prints clean data on stdout and diagnostics on stderr, and exits with the codes in §17.2. Typical one-liners:

```sh
mill runs --state failed --since 24h --json | jq -r '.[].id' | xargs -n1 mill rerun
mill ls --json | jq -r '.[] | select(.last.state=="failed") | .name' | xargs -n1 mill trigger disable
watch -n 5 mill status
```

### 17.12 CI recipe

```sh
# check a pipeline change without side effects
mill lint && mill run docs-drift --local --dry-run --json > plan.json

# run it for real, no daemon
mill run docs-drift --local --budget-tokens 300k --json > result.json
case $? in
  0) echo "ok" ;;
  9) echo "budget exceeded" ; exit 1 ;;
  *) exit 1 ;;
esac
```

Secrets come from the CI environment through the `env` secrets backend. The run store can be uploaded as a build artifact and opened later with `mill runs --store ./mill-runs`.


---

[← spec index](index.md)
