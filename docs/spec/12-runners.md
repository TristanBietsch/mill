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

### 12.1 Environment

Every step process gets a clean, explicit environment, never `milld`'s whole environment (it may hold tokens from the shell that started it):

1. **Base:** `milld`'s environment filtered by the `env.passthrough` allowlist. Default: `PATH HOME USER LOGNAME SHELL LANG LC_* TERM TZ TMPDIR SSH_AUTH_SOCK XDG_* GIT_* EDITOR`, plus on Windows `SYSTEMROOT COMSPEC PATHEXT USERPROFILE APPDATA LOCALAPPDATA TEMP TMP`.
2. **Pipeline `env`,** then **node `env`** (§7.2, §7.3). These may name secret handles (`GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}`), which is the only way a secret reaches an `sh` or `exec` process (§21).
3. **mill's own variables:** `MILL_RUN_ID`, `MILL_RUN_NUMBER`, `MILL_PIPELINE`, `MILL_NODE_ID`, `MILL_NODE_RUN_ID`, `MILL_ATTEMPT`, `MILL_ITEM_INDEX`, `MILL_WORKSPACE`, `MILL_ARTIFACTS`, `MILL_SCRATCH`, `MILL_OUTPUT` (agents, §11.3), and `MILL_DRY_RUN=1` in a dry run.

**Processes.** Each node-run runs in its own process group on macOS and Linux and its own Job Object on Windows, so cancel and teardown kill everything it started. After a daemon restart, leftover processes are found by their recorded identity and killed.


---

[← spec index](index.md)
