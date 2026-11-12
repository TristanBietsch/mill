## 12. Runners

A runner decides *where* a node-run executes. Runners are parts too.

| Runner | Notes |
|---|---|
| `local` (default) | Child process on this machine, in its own process group. |
| `pty` | Like `local` with a pseudo-terminal, for agents that only run in a terminal. Supports view-only attach and screen repaint. |
| `container` | Docker or Podman. `image`, `mounts` (workspace mounted by default), `network: none | default`, resource limits. Good isolation for untrusted steps. |
| `ssh` | Run on another host. Workspace synced or the remote path used. |
| `remote` | Hand the node-run to another `milld` (a home server or VPS). Items and artifacts move over the gateway. |
| `external` | Delegate to an external session manager through an adapter part. For tools that already supervise agent sessions: mill asks them to start a session, follows its state, and reads its output. |

Runner selection: node `runner` → pipeline `defaults.runner` → the `default_runner` setting. `policy.allowed_runners` can restrict it (for example, force `container` for everything in pipelines from the registry).

Each runner has its own concurrency cap (`runners.<id>.max_concurrent`).


---

[← spec index](index.md)
