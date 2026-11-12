## 23. Integrations and interop

### 23.1 MCP, both directions

- **mill as an MCP client:** servers declared with `mill.mcp` (or shipped in a plugin) are started by the part host. Every tool appears in the palette as `mcp:<server>/<tool>`, with a form built from its input schema.
- **mill as an MCP server:** `mill mcp serve` exposes `list_pipelines`, `describe_pipeline`, `run_pipeline`, `get_run`, `rerun`, `lint_pipeline` and `save_pipeline`. Any agent can build, check and launch pipelines itself.

### 23.2 HTTP and OpenAPI

`http` covers any REST API. `mill plugin add openapi:<url-or-file>` generates a plugin whose nodes are the spec's operations, with typed inputs and auth wired to declared secrets.

### 23.3 n8n and Zapier

| Direction | How |
|---|---|
| mill → n8n/Zapier | `n8n.call` / `zapier.call` POST to an n8n Webhook node or a Zapier Catch Hook. With `wait: true`, mill passes a one-time callback URL and the remote result returns as an item. |
| n8n/Zapier → mill | `webhook` triggers. Locally `http://127.0.0.1:7337/hook/<pipeline>/<trigger>`. Public callers need `gateway.public_url` (a tunnel or the relay). |
| Import n8n | `mill import n8n workflow.json` maps common nodes (Webhook, Schedule, HTTP Request, IF, Switch, Set, Code, Merge, Wait, Execute Workflow) to mill parts. Anything it can't map becomes an `n8n.call` stub that runs it on n8n, and the import prints a list of stubs. |
| Export | `mill export <p> --to n8n` for pipelines using only mappable parts. |
| Later | An n8n community node package and a Zapier app, both thin wrappers over the gateway API. |

### 23.4 Other exports

- `--to mermaid`: a diagram for READMEs and PR descriptions.
- `--to gha`: a GitHub Actions workflow for pipelines that only use `sh`, `exec`, `check` and data parts. Lets a pipeline built locally run in CI unchanged.

### 23.5 External session managers

Tools that already supervise agent sessions can be used through the `external` runner (§12): an adapter part translates "start session, report state, give output, send input" into that tool's own protocol. mill doesn't need to know the tool exists beyond the adapter.

### 23.6 Alongside rex, or anything else

mill is designed as rex's companion. The two split the work cleanly:

| | rex | mill |
|---|---|---|
| What it runs | Agent sessions you work with directly | Pipelines that run unattended |
| When a person is needed | The session shows up in NEEDS YOU and waits | Never waits: the work is handed off and that branch ends |
| Main screen | Kanban board of sessions | Table of pipelines |
| Typical use | "Work on this with me" | "Do this every time X happens" |

They share the same conventions (client and daemon over a local socket, JSONL messages, selectors, exit codes, a settings registry, soundsets, vim-style keys), so they feel like one toolset.

**Connection points.** All of these come from the official `mill/rex` plugin. Nothing in mill's core knows about rex.

| Connection | What it does |
|---|---|
| **Handoff to rex** | A `rex.session` handoff target starts a rex session with a prompt built from the run (task, plan, diff, failing test) in the run's workspace. The mill branch ends there, and the person picks the work up in rex. |
| **rex as a runner** | An `external` runner adapter starts agent nodes as rex sessions, grouped in a rex fleet named `mill/<pipeline>/<run>`, and follows their state over rex's socket. mill-run agents then appear on the rex board. If a session asks for input, mill fails the node as usual (`M6304`) and leaves the session in rex for someone to take over. |
| **Triggers from rex** | A `rex.event` trigger starts a mill run when a rex session changes state (for example, run a review-and-PR pipeline when a session is marked done). rex's own Lua hooks can also call `mill run`. |
| **Status side by side** | `mill prompt` and rex's status line sit next to each other in the shell prompt and the tmux status bar. |

**Without rex.** Every connection point has a generic equivalent, so nothing is lost:

| With rex | Without rex |
|---|---|
| Handoff to a rex session | Handoff to a GitHub issue or comment, a Slack or email message, a Linear ticket, or any plugin |
| rex as a runner | `local`, `pty`, `container` or `ssh` runners, or an `external` adapter for any other session manager |
| `rex.event` trigger | Webhooks, cron, file watches, git hooks, polling, or other pipelines' runs |


---

[← spec index](index.md)
