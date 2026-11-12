# Pipeline format (agents)

Contract summary. Reference: **[spec §7](../../spec/07-pipeline-format.md#7-pipeline-file-format)**, expressions in **[spec §8](../../spec/08-expressions.md#8-expressions-and-templating)**. User doc (when shipped): [../../pipeline-format.md](../../pipeline-format.md). Code: [../modules/pipeline.md](../modules/pipeline.md). Status: planned.

## Where pipelines live

`.mill/pipelines/*.mill.yaml` (project) and `~/.config/mill/pipelines/*.mill.yaml` (user). Id = `name`, unique per scope; project shadows user. Pipe expressions can be saved to a file with `--save`.

## Top-level keys

| Key | Required | Meaning |
|---|---|---|
| `mill` | yes | Format version, `1` |
| `name` | yes | Lowercase kebab-case id |
| `description`, `tags` | no | One line; free labels |
| `on` | no | Triggers; none = manual or piped only |
| `inputs` | no | Named inputs: type, default, description, `from` |
| `outputs` | no | Named expressions → the run's result |
| `env` | no | Environment for every node |
| `secrets` | no | Secret names this pipeline may use |
| `budget` | no | `max_tokens`, `max_cost_usd`, `max_wall`, `max_nodes`, `max_agents` |
| `policy` | no | `allow_destructive`, allowed runners, network, `fail_fast` |
| `defaults` | no | Default `runner`, `retry`, `timeout`, `workspace`, `agent` |
| `nodes` | yes | Map of node id → node |
| `edges` | no | Explicit edges; usually inferred |

## Node keys

`uses`, `with`, `in`, `mode` (`batch`/`each`/`stream`), `concurrency`, `when`, `needs`, `runner`, `workspace` (`inherit`/`git`/`temp`/`none`), `retry`, `timeout`, `cache`, `on_error` (`fail`/`continue`/`route`), `output` (schema), `io` (`envelope`/`plain`/`text`). See [§7.3](../../spec/07-pipeline-format.md#73-node-keys).

`uses` forms: built-in id (`sh`, `agent`, `assert`), `mcp:<server>/<tool>`, `part:<id>@<version>`, a plugin part id (`linear.issue.create`).

## Edges

Inferred from `${{ nodes.a.… }}` and `in: a`. Write `edges:` only to name a non-default port (`check.false -> fix`) or order unrelated nodes (prefer `needs:`). Chains: `- a -> b -> c`. See [§7.4](../../spec/07-pipeline-format.md#74-edge-inference).

Known issue: a node referencing several nodes gets all their items as input under the literal rule. The backend plan proposes a "primary upstream" rule and lint error `M3021` (backend S7, D9). Not accepted yet.

## Expressions

`${{ … }}` is CEL. A string that is one whole expression keeps its type. Variables: `inputs`, `event`, `item`, `items`, `nodes.<id>.item(s)/state/exit_code/files/ws/diff`, `run`, `env`, `secrets`, `now`. Helpers: [§8](../../spec/08-expressions.md#8-expressions-and-templating).

## Validation (`mill lint`)

Schema; unknown parts or versions; unbound references; cycles outside `loop`; type mismatches between `output` and consumers; undeclared secrets; policy violations (unlisted destructive parts); unreachable nodes; disallowed runners. Diagnostics with file:line:column, text or `--json`. See [§7.7](../../spec/07-pipeline-format.md#77-validation-mill-lint).

## Versioning

New optional keys do not bump `mill:`. Changing a key's meaning does. Readers ignore unknown keys ([ADR-017](../DECISIONS.md#adr-017-additive-evolution)).

## Companion files

- Layout sidecar `<name>.mill.layout.json`: canvas positions only ([../data/schemas.md](../data/schemas.md#layout-sidecar)).
- Prompt files referenced with `@path` (`@prompts/plan.md`).

## Example

Full worked pipeline: [§7.5](../../spec/07-pipeline-format.md#75-full-example) (`issue-to-pr`).
