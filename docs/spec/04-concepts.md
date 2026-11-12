## 4. Concepts and glossary

| Term | Meaning |
|---|---|
| **board** | The TUI's one screen: every pipeline, one line each, in Failed, Running and Ready sections. |
| **pipeline** | A named, versioned graph defined in a `*.mill.yaml` file. |
| **node** | One step in a pipeline. It `uses` a part and configures it `with` settings. |
| **part** | One capability the engine uses: a node type, trigger, runner, secrets backend, sink or artifact store. |
| **plugin** | An installable, shareable package containing parts, MCP server and agent definitions, pipelines, prompts and config defaults (§9). |
| **bundle** | A plugin that mostly contains pipelines; what `mill export` produces and `mill import` consumes. |
| **port** | A named input or output on a node. The default ports are `in` and `out`. Branching parts add more (`true`/`false`, `case.*`, `default`, `else`, `exhausted`, `error`); a loop's body reads from its `body` port. |
| **edge** | A connection from one port to another. It carries a stream of items. |
| **item** | One unit of data on an edge: a JSON value, plus optional files, a workspace reference and lineage metadata (§6). |
| **artifact** | A file produced by a node. Stored by content hash and referenced from items. |
| **workspace** | An isolated working directory (git worktree, temp dir or container volume) owned by a run or a node. |
| **run** | One execution of a pipeline. Has an id, inputs, a state, node-runs, artifacts and a budget ledger. |
| **node-run** | One execution of one node within a run. Nodes in `each` mode produce one per item. |
| **trigger** | Something that starts runs: manual, cron, webhook, file watch, git, poll, stdin, MCP call, another run's event. |
| **runner** | Where a node-run executes: local process, PTY, container, SSH host, remote `milld`, or an external session manager. |
| **assert** | A node that checks a condition on items and fails, routes or drops them when it's false. The automated replacement for an approval. |
| **handoff** | A node that passes work to a person or another tool (an issue comment, a message, a ticket) and ends that branch. mill never waits for the answer. |
| **agent** | An agent CLI described in the agent registry (§11). |
| **budget** | Limits for a run: tokens, money, wall time, node count, concurrent agents. |
| **pattern** | A small pre-wired group of nodes the editor inserts in one step, such as plan → check → handoff (§18.7.6). Patterns are YAML files. |
| **sample** | Items pinned as a node's input, used for expression previews and single-node runs while editing (§18.7.12). |
| **selector** | How a run is referenced on the CLI: full id, `pipeline#number`, or a short hex id. |
| **pipe expression** | A one-line `a \| b \| c` pipeline written in the shell (§14). |

### States

Run and node-run states share one vocabulary:

| State | Run | Node-run | Meaning |
|---|---|---|---|
| `pending` | | ✓ | Not ready yet (waiting on inputs) |
| `queued` | ✓ | ✓ | Ready, waiting for capacity |
| `working` | ✓ | ✓ | Executing |
| `retrying` | | ✓ | Failed with a retryable error (rate limit, network, timeout) and waiting out its backoff before the next attempt |
| `done` | ✓ | ✓ | Finished successfully |
| `failed` | ✓ | ✓ | Finished with an error |
| `skipped` | | ✓ | On a branch that wasn't taken, or `when` was false |
| `cached` | | ✓ | Output reused from the cache; not executed |
| `cancelled` | ✓ | ✓ | Stopped by a person, a policy or a budget |
| `crashed` | ✓ | ✓ | The daemon died while it was live, and it couldn't be resumed or safely re-run (§15.7) |

`done`, `failed`, `skipped`, `cached`, `cancelled` and `crashed` are terminal; nothing moves a node-run out of a terminal state, only a *new* node-run (a retry) can follow it.


---

[← spec index](index.md)
