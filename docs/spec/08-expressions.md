## 8. Expressions and templating

- `${{ … }}` in any string is a **CEL** expression. A string that's entirely one expression keeps its type (object, list, number). Otherwise the result is interpolated as text.
- Available variables:

| Variable | Meaning |
|---|---|
| `inputs` | Run inputs |
| `event` | The trigger payload |
| `item` | The current item's `data` (in `each` mode, or inside `with` for per-item evaluation) |
| `items` | All input items' `data` (in `batch` mode) |
| `nodes.<id>.item` / `nodes.<id>.items` | Output of an upstream node: one item, or all items. In `each` mode, `nodes.<id>.item` follows lineage (`meta.parents`) to *this* item's ancestor from that node when there is one; otherwise it's the node's first item. |
| `nodes.<id>.state`, `.exit_code`, `.files`, `.ws`, `.diff` | Node-run metadata |
| `run` | `id`, `number`, `started_at`, `pipeline`, `budget.remaining` |
| `loop` | Inside a loop's body: `iter` (1-based), `max`, `prev` (the last try's output) |
| `env`, `secrets` | Environment and declared secrets. A secret is an opaque handle: it can be passed whole to a field the part marks as a secret, or to an `env` entry, but it can't be turned into a string or interpolated into a larger one (`"Bearer ${{ secrets.X }}"` is a lint error). Its value only reaches parts. |
| `now` | Current time |

- Helper functions on top of standard CEL: `json()`, `yaml()`, `text()`, `lines()`, `trim()`, `slug()`, `hash()`, `glob()`, `duration()`, `template()`, `default(a, b)`.
- **Lua** is available only inside the `lua` transform part, sandboxed (no `os`, `io` or network) unless the pipeline is trusted and the node sets `unsafe: true`.


---

[← spec index](index.md)
