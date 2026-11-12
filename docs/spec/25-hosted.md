## 25. Optional hosted layer

Everything works without it. It only adds reach.

| Piece | What it solves |
|---|---|
| **Relay** | A stable public webhook URL that forwards to your `milld` over an outbound connection, and queues events while you're offline. |
| **Remote runners** | `milld` on a VPS or home server for jobs that shouldn't depend on a laptop being awake. Same binaries; `mill remote add`. |
| **Run viewer** | A read-only web page per run (graph, states, items, logs) to share with people who don't live in a terminal. Secrets are redacted. |
| **Plugin registry** | Publish, search and install signed plugins and pipeline bundles (`mill plugin add acme/linear`, `mill import acme/issue-to-pr`). Private registries for teams. |
| **Shared history** | Teammates see the same pipelines, runs and failures in the run viewer, and get the same failure notifications. |

The business shape this allows: open-source core (mill, milld, built-in parts, SDKs) plus a paid relay, hosted runners and team features.


---

[← spec index](index.md)
