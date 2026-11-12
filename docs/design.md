# Design overview

One page on what mill is and how it looks and feels. The spec is the reference: [spec/index.md](spec/index.md), and for the TUI [§18](spec/18-tui.md#18-tui-reference). The HTML prototype is [`mockups/v2/index.html`](../mockups/v2/index.html).

## What mill is

A terminal-native pipeline engine for agents, code and automations. You describe a pipeline as a graph of steps (agent calls, shell commands, HTTP requests, MCP tools, checks, branches, loops) in a `*.mill.yaml` file. A local daemon runs it by hand, on a schedule, or when something happens. Every step is a filter: items in, items out, so mill pipes into `jq`, `grep` and other pipelines ([§1](spec/01-summary.md#1-summary), [§6](spec/06-data-model.md#6-data-model-items-artifacts-workspaces)).

It sits next to [rex](https://github.com/TristanBietsch/rex): rex holds the agent sessions that need a person, mill runs the work that needs no one. mill doesn't depend on rex.

## Hands-off

The first principle ([§2](spec/02-principles.md#2-principles)): **a run never waits on a person.** No approvals, questions or takeovers. Decisions inside a run come from checks, judges, reviews, budgets and policies. Work that needs a human goes through a `handoff` node (an issue comment, a message, a ticket) and that branch ends. If the answer should continue the work, it arrives later as a trigger and starts a new run.

What that means for design:

- Budgets and policies are set before the run, in the pipeline (the wizard's "limits" step, the editor's pipeline panel), because nobody is there to stop it later.
- Lint refuses parts that wait for a person (`M3206`), and an agent pipeline with an automatic trigger and no budget doesn't lint (`M3201`).
- Failures are designed to be read later: every error has a code, a cause and the next command ([§29](spec/29-errors.md#29-errors-and-diagnostics)).

## Simple surface, deep engine

Scheduling, retries, budgets, plugins and crash recovery live in `milld`. They reach the screen only as a state, a few words, or a square in the run history. The design rules come from rex ([§18.1](spec/18-tui.md#181-design-rules)): one screen, five keys in the help bar, color means state and only state, columns appear only when a row needs them, rows never wrap.

## The three screens that matter

| Screen | What it is | Spec |
|---|---|---|
| **Board** | The TUI's one screen: every pipeline on one line, in FAILED, RUNNING and READY, with a summary header (`1 failed · 2 running · 14 ok today · $6.20 today`). Each row shows state, name, one useful line of text, the last runs as squares, and age. | [§18.2](spec/18-tui.md#182-the-board), [§18.3](spec/18-tui.md#183-row-columns) |
| **Run popup** | `enter` on a pipeline: one row per node with an inline timeline, branches and failed attempts as indented sub-rows, `L` for the log in the same popup, `enter` on an agent for its screen (view-only). | [§18.6](spec/18-tui.md#186-run) |
| **Editor** | `e` on a pipeline: the graph and an inspector. Add parts with `a`, connect ports with `c`, branch and loop, fix what live lint finds with `ga`, fill prompts and expressions with completion and preview, and dry-run the buffer on the graph. Every action is one YAML edit and has a CLI command. | [§18.7](spec/18-tui.md#187-editor) |

Around them: the boot splash ([§18.13](spec/18-tui.md#1813-boot-splash)), the new-pipeline wizard ([§18.8](spec/18-tui.md#188-wizards)), plugins ([§18.9](spec/18-tui.md#189-plugins)), settings ([§18.10](spec/18-tui.md#1810-settings)), sound ([§18.11](spec/18-tui.md#1811-sound)) and the screensaver ([§18.12](spec/18-tui.md#1812-screensaver-long-runs-detach)).

## Headless first

The TUI is one client. Every screen has a CLI command, every listing takes `--json`, and `mill run --local` runs a pipeline with no daemon for CI and cron ([§17](spec/17-cli.md#17-cli-reference)).

## Where to go next

- The full TUI reference: [spec §18](spec/18-tui.md#18-tui-reference).
- The prototype: [`mockups/v2/index.html`](../mockups/v2/index.html) (the spec wins on behavior; [plans/editor.md](plans/editor.md) lists what it covers).
- How it's built: [tech.md](tech.md).
