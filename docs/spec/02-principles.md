## 2. Principles

These decide arguments. When a design choice is unclear, the principle higher on this list wins.

| # | Principle | What it means in practice |
|---|---|---|
| 1 | **Hands-off.** | A run never waits on a person: no approvals, no questions, no takeovers. Decisions are made by checks, judges and policies inside the pipeline. Work that needs a human is handed off and that branch ends. If the answer should continue the work, it arrives later as a trigger and starts a new run. |
| 2 | **Simple surface, deep engine.** | The screen stays small however much happens underneath. Scheduling, retries, budgets, plugins and recovery live in `milld` and reach the screen only as a state, a few words, or a square in the run history. |
| 3 | **One glance.** | The board answers "is anything wrong, what's running, what happened" in one look: a summary header, three sections, one line per pipeline. Detail is one key away, never in the way. |
| 4 | **Client renders, daemon owns.** | Two binaries: `mill` (CLI and TUI) and `milld` (daemon). `milld` owns child processes, triggers, schedules and disk. You can close the TUI and runs keep going. Many clients can connect at once. Everything is scriptable. |
| 5 | **Files are the truth.** | Pipelines are plain text in git. Run history is an append-only event log. Every database is a cache that can be rebuilt from files. |
| 6 | **Pipes all the way down.** | A step is a filter: items in, items out. Any executable that reads stdin and writes stdout is a valid step with zero integration work. mill itself behaves the same way at the shell. |
| 7 | **Everything is a part.** | Steps, triggers, runners, agents, secret stores and sinks are plugins behind one protocol. The core is a scheduler plus a part host. Built-ins get no special internals. |
| 8 | **No vendor lock-in.** | Agents are described declaratively: command, arguments, how they take a prompt, how state and output are detected. No vendor HTTP APIs or SDKs are required. |
| 9 | **Headless first.** | Every TUI action has a CLI equivalent. The TUI is one client among many. CI, cron and SSH sessions get the same power. |
| 10 | **Optional never blocks core.** | Summaries, notifications, sounds, telemetry and the hosted relay can fail without affecting a run. Headless and SSH environments degrade silently. |
| 11 | **Faithful replay.** | Raw output of every step is captured as-is, minus registered secret values (§21). Views (states, summaries, items) are derived from it, so any run can be inspected or replayed exactly. |
| 12 | **Additive evolution.** | Wire messages, parts protocol and file format are versioned. New optional fields and new message types don't bump the version, and readers ignore unknown fields. Changing a meaning requires a version bump. |
| 13 | **Errors are interface.** | Every error says what happened, where, and the exact next command, with a stable code (`mill explain M3012`). Same shape in the CLI, TUI, JSON and protocol (§29). |
| 14 | **Safe by default.** | Steps that change code run in isolated workspaces. Trust is explicit. Dangerous paths are refused. Secrets never touch logs. Listeners bind to localhost. |
| 15 | **Local-first, cloud-optional.** | Everything works offline on one machine. Hosted pieces only add reach (public webhooks, always-on runners). |
| 16 | **Boring formats.** | JSONL on the wire, YAML for pipelines, Lua for config, UTF-8 text, exit codes. No custom binary formats. Easy for people and LLMs to read and write. |


---

[← spec index](index.md)
