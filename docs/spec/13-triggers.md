## 13. Triggers

Triggers are parts with the `trigger` capability. A pipeline can have several.

| Trigger | Config highlights |
|---|---|
| `manual` | Always available: TUI, `mill run`, MCP, API. |
| `cron` | Five-field cron or `@every 10m`, `tz`, `jitter`. Missed runs while the machine slept: `catch_up: none | last | all`. |
| `webhook` | `path`, verification (`token`, `hmac`, `github-hmac`, `stripe`), `filter` expression, response mode (`ack` immediately or `wait` for outputs). |
| `fswatch` | Paths and globs, events, `debounce`. |
| `git` | Local hooks (`post-commit`, `post-merge`, `pre-push`), installed with `mill hooks install`. |
| `poll` | Run any node part on an interval and fire for each *new* item (dedupe by a key expression). Covers RSS, GitHub notifications, IMAP, APIs without webhooks. |
| `stdin` | The pipeline is used as a filter (§14). |
| `run` | Another run's event: `done` or `failed` of pipeline X. Chains pipelines without coupling their files. |
| `mcp` | Called as a tool by an MCP client (§23). |
| `schedule.once` | Run at a specific time, then disable. |

**Concurrency policy** (per pipeline, `on.concurrency`): `queue` (default), `skip` (drop new events while one is running), `replace` (cancel the running one), or `parallel: N`. Each event can carry a `dedupe` key; duplicate keys within `dedupe_window` are dropped.


---

[← spec index](index.md)
