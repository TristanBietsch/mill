## 28. Roadmap

| Phase | Scope | Done when |
|---|---|---|
| **v0: engine** | Go module and release pipeline; the error system (codes, classes, human and JSON formats, `mill explain`) from the first command; Lua config (`init.lua`, `mill.setup`, `mill.agent`, layered loading); file format and `mill lint`; `milld` with scheduler, run store and parts protocol (internal); parts `sh`, `exec`, `check`, data parts, `if`, `assert`, `handoff`, `agent`, `git.worktree`; CLI `run`, `runs`, `logs`, `wait`, `items`, `rerun`; mill as a stdin/stdout filter. | A pipeline runs end to end from the CLI and pipes cleanly into `jq`. |
| **v0.1: CLI and pipe** | Full headless CLI (§17.5–17.12): `--local`, `watch`, `render` (text), `show --outline`, the edit commands (`node`, `edge`, `pipeline set`, `wrap`, `extract`, `pattern`, `refs`, `lint --fix`, `estimate`; §18.7.16), `attach` (view-only), `--dry-run`, `status`/`prompt`, completion. Pipe expressions (`mill pipe`, `--save`), `--each`, `mill events`, exit codes complete. | A pipeline can be built, run, inspected and re-run end to end without the TUI, and the one-liners in §26 work. |
| **v0.2: see it** | The boot splash (§18.13), the board (sections, five-thing rows, `v` detailed view, conditional columns, narrow-terminal rules), `λ` prompt, run popup with inline timeline and log, view-only agent screens, settings, sounds, screensaver, `:stats`. | You can tell what's wrong, what's running and what happened from the board alone, and follow a run from trigger to result. |
| **v0.3: edit it** | The five-step wizard, the editor (§18.7: graph and inspector, palette with patterns, port-aware connect, branches and loops, live lint with quick fixes, expression completion and preview, the pipeline panel, dry run on the graph, single-node runs with pinned samples, undo, YAML pane, layout sidecar), `:plugins`. See `docs/plans/editor.md` for the phases. | best-of-3, issue-to-pr and nightly-sweep can each be built from an empty file without a text editor, every editor action has a tested CLI equivalent, and the editor never writes YAML that `mill lint` rejects without saying so. |
| **v0.4: triggers** | cron, webhook (gateway), fswatch, git hooks, poll, run-to-run. | The nightly sweep runs unattended for a week. |
| **v0.5: plugins** | Public parts protocol v1, Lua parts, Go/TS/Python SDKs, plugin format and manifest, `mill plugin` commands, `mill import` / `mill export` bundles, lockfile, permissions, MCP client and server, OpenAPI plugins, container and SSH runners. | A Linear plugin written by someone else installs pinned, and a pipeline exported on one machine imports and runs on another. |
| **v0.6: interop** | n8n import/export, `n8n.call`/`zapier.call` with callbacks, exports to mermaid and GitHub Actions, `external` runner adapters. | A real n8n workflow imports and runs. |
| **v1: hosted** | Relay, remote runners, run viewer, registry, shared history. | A GitHub webhook runs a pipeline on a VPS while the laptop is closed. |

**Weekend prototype:** skip the daemon. A single `mill run` that parses YAML, runs `sh` and headless `agent` nodes in topological order, and pipes JSONL between them. That's enough to test the file format, the item model and the pipe ergonomics before building `milld`.


---

[← spec index](index.md)
