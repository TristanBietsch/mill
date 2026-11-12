## 6. Data model: items, artifacts, workspaces

### 6.1 Items

Edges carry **streams of items**. A stream may hold zero, one or many items. Item order is preserved along an edge.

The full item envelope (as parts see it):

| Field | Type | Meaning |
|---|---|---|
| `data` | any JSON | The payload. This is what expressions see as `item`. |
| `files` | list | Artifact references: `name`, `path` (local view), `sha256`, `size`, `mime`. |
| `ws` | object or null | Workspace reference: `kind` (`git`, `dir`, `volume`), `path`, `repo`, `branch`, `base`. |
| `meta` | object | Lineage: `run`, `node`, `index`, `parents` (ids of the items it came from), `ts`. |

**Plain filters don't see the envelope.** A node using `io: plain` gets only `data` (one JSON value per line), or raw text lines with `io: text`. mill wraps their output back into envelopes and carries `files`, `ws` and lineage forward automatically. That's what makes `jq`, `grep`, `sed` and your own scripts work as steps unchanged.

### 6.2 Node modes

| Mode | Behavior | Typical use |
|---|---|---|
| `batch` (default) | One invocation receives all upstream items and emits any number. | Summaries, merges, sorting, "do this once" steps. |
| `each` | One node-run per item, in parallel up to `concurrency`. | Agent per issue, HTTP call per record. |
| `stream` | One long-lived invocation. Items flow in and out as they arrive, with backpressure. Arrives in v0.4 with the gateway, where its replay, caching and crash recovery are specified; v0 has `batch` and `each`. | Log tailers, chat bots, high-volume filters. |

### 6.3 Artifacts

Files a node produces (a diff, a report, an image, a build output) are stored once by SHA-256 in one global content-addressed store (`~/.local/share/mill/artifacts/`, §20). Each run records references to the artifacts it produced or used, so identical files are stored once and reruns and the cache are cheap. A mark-and-sweep GC removes artifacts no kept run references (`retention`, §19.6). Items reference artifacts by hash. Downstream nodes get a read-only local path. Artifacts larger than a configured limit can live on a pluggable artifact backend (local disk by default; S3-compatible as a part).

### 6.4 Workspaces

A workspace is where code-changing work happens.

| Kind | Created by | Notes |
|---|---|---|
| `git` | `git.worktree` part, or `workspace: git` on a node | A new branch and worktree from a base ref. Isolated per run or per item. |
| `dir` | `workspace: temp` | Scratch directory, removed at run end unless kept. |
| `volume` | Container runner | Mounted into the container. |

Workspaces travel on items (`ws`), so a later node can "continue in the same place." Cleanup policy: `on_success`, `always`, `never`, or `keep: 7d`.


---

[← spec index](index.md)
