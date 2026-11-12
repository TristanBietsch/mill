# ≋ MILL: Machine Intelligence Logistics Layer

*A terminal-native, pipeable, fully modular pipeline engine for agents, code and automations.*

```
mill — Machine Intelligence Logistics Layer
```

> **Why "mill":** a mill is where raw material goes in and finished product comes out, which is what a pipeline does. The name also carries two histories: the water mill (fluid power, channels, sluices) and the "mill" Babbage named as the processor of his Analytical Engine (1837).
>
> **Mark:** `≋` (U+224B, TRIPLE TILDE): water running through a mill race. It appears in the TUI header, the startup splash, `mill status` and the shell prompt segment. ASCII fallback: `~`.
>
> **Companion:** built to be used alongside [rex](https://github.com/TristanBietsch/rex), the terminal board for agent sessions you work with directly. rex handles the work that needs a person; mill runs the work that needs no one. mill doesn't depend on rex, though, and works with anything (§23.6).
>
> **Status:** specification draft, **v2.3** (v2 plus the decisions of 2026-11-11 and 2026-11-12, below). v2 keeps everything the engine does and redesigns the surface around rex's design rules: one board, one line per pipeline, everything readable at a glance (§18.1). No code yet. When this spec and the eventual source disagree, the source wins, and this document gets fixed.

---

## What changed in v2

The engine is the same; the surface is much smaller. The goal is a mill as simple and refined on the surface as ray: everything in one glance, while the backend does a lot.

| v1 | v2 |
|---|---|
| Five tabs (pipelines, create, runs, log, plugins) | **One board.** Everything else opens over it and `esc` returns to it (§18.1). |
| A table with seven columns | **Five things per row** (state, name, text, runs, age), `v` for the rest, extra columns only while a row needs them (§18.3) |
| Rows sorted in one list | **Three sections:** Failed, Running, Ready, each with a count; long Ready folds (§18.2) |
| Text column showed the state again | **Text says the one useful thing:** where it failed, what it's doing, or the last result and next run |
| Run view as a box graph + a separate log tab | **Run popup:** one row per node with an inline timeline; `L` shows the log in the same popup (§18.6) |
| Canvas with a permanent parts drawer | **Editor:** graph + inspector only; parts on demand with `a`; port-aware connect, branches and loops, live lint with quick fixes, dry run painted on the graph (§18.7) |
| Seven-step wizard | **Five steps:** describe, trigger, agent, limits, review (§18.8). v2.2 replaced them with four |
| Help bars with up to nine keys | **Five keys:** `enter open · n new · e edit · R run · ? more` |
| Bottom prompt unused | **A bottom prompt runs things:** a pipeline with inputs, or a pipe expression (§18.2). v2.2 replaced it with the `:` command line (`:run`, `:pipe`) |
| Plugins as a tab | `:plugins` overlay; `:stats` added (§18.9, §18.12) |
| A blank screen until the board loaded | **Boot splash:** a clone of rex's bootloader with mill's own steps, `[  OK  ]`/`[ WARN ]`/`[ SKIP ]`/`[ FAIL ]` rows, and a cause and fix when boot fails (§18.13) |

Everything else (engine, parts and plugins, Lua config, agents, triggers, piping, errors, security, sound, Go) carries over from v1 unchanged except where it referred to the old screens.

## v2.1 (2026-11-11): decisions applied

The open decisions in the backend and editor plans were decided, and the backend plan's spec issues S1–S31 were applied. The record is in `docs/agents/DECISIONS.md` (ADR-024 to ADR-044).

- **Name and distribution:** `mill` stays; repo and module `github.com/TristanBietsch/mill`, Homebrew tap `tristanbietsch/tap/mill`, MIT license, Go 1.27, Windows tier 2 in v0 (§27.1).
- **Edges:** every reference orders; the input is the primary upstream; no primary upstream is lint `M3021`. Unreachable moved to `M3024` (§7.4, §18.7.3, §18.7.9).
- **Trust:** a fresh clone's pipelines are disarmed until `mill trust .` (§22, §13, §18.3).
- **Parts:** built-ins win; `PATH` executables and MCP tools only fill gaps (§9.4). Registry v0.5 is a GitHub index with SHA-256 hashes; signatures later (§9.6, §9.11).
- **Execution:** retries are new attempts, with a default policy (§4, §15.3); allowlisted step environment (§12.1); dry runs run `sh`/`exec` unless `destructive: true` and stub agents (§17.10); `max_wall` counts active time; budget stops exit 9 (§15.5, §17.2); `mill daemon stop` is a crash, `--drain` waits (§15.7).
- **Storage and protocol:** global artifact store (§6.3), redaction is the only change before persistence (§5.4), AF_UNIX socket everywhere with a `~/.cache/mill/milld.sock` fallback (§16, §20), new intents (§16.3), YAML v4 with text-splice edits (§27.2), backend package layout (§27.4).
- **Board and run popup:** crashed and budget-cancelled pipelines sit in FAILED; user-cancelled and untrusted ones in READY (§4, §18.2, §18.3); conditional columns sit between the text and the runs strip (§18.3); the board shows the mode name at the left of the help bar (§18.4); branches and failed attempts are indented sub-rows (§18.6).
- **Editor:** loop YAML is `with.body` + `until` + `max_iters` with `out` and `exhausted` (§10.3); `M3201` is an error only with an automatic trigger (§18.7.9); `:w!` saves broken pipelines disarmed; samples are committed and checked for secrets (`M9208`, §18.7.12).
- **Open questions** 3, 9, 10 and 12 are resolved (§30).

## v2.2 (2026-11-12): wizard and command line

The wizard and command-line plans (`docs/plans/wizard.md`, `docs/plans/command-line.md`) were decided and merged. The record is in `docs/agents/DECISIONS.md` (ADR-045 to ADR-052).

- **Command line:** one vim-style `:` line on the terminal's last row, from every screen except a pending y/n confirm (a confirm takes only `y`, `n`, `esc`, like nvim's `confirm()`); completion with a wildmenu, history with `ctrl+r`, counts, `!`, echoes in the error shape, `:help commands`; `/` is search only, also on the last row (§18.4, §18.5, §18.9, §18.10, §29.8). New codes `M1302`–`M1308`.
- **The board's bottom prompt is gone:** `:run <pipeline> k=v` and `:pipe <expr>` replace it; `i` on the board only echoes that hint. In the run popup, items move to `gi` (§18.2, §18.5, §18.6, §17.5).
- **Wizard:** four steps (start › shape › run › review), the editor's NORMAL/INSERT keys, sources describe, template (with plugin templates and patterns), duplicate, import a file and blank; the name is chosen last, on review; create runs a dry run, then opens the editor (§18.8, §17.1).
- **Drafter:** *describe it* drafts with a local model through Ollama by default (`drafter.*` settings, a Drafter section in settings, model resolved against pulled models), a headless agent when opted in, keyword rules when off or unavailable (`M2212`, `M2213`); the reply is validated against the schema and lint before it shows (§18.8, §18.10, §19.6, §16.3, §29.10).
- **Import:** type detection for n8n, `.millpkg`, mill pipelines and pipe expressions with a preview; n8n steps that waited for a person become a `handoff` (`M2307`); new codes `M2301`–`M2309`, `M3050` (§18.8, §18.7.9). A duplicate's triggers start as `manual`.

## v2.3 (2026-11-12): full-screen pages, following rex v2

Settings, help and the new-pipeline wizard become full-screen pages in the style of rex v2's settings, help and new-agent pages. The record is ADR-055 in `docs/agents/DECISIONS.md`; the rex proposals that would change a decided mill behavior are open questions 13–21 (§30).

- **Rules:** rule 1 (pages replace the board, `esc` leads back) and rule 10 (popups are square and sized to fit; pages own the screen) are rewritten with their reason (§18.1). A shared page frame: `≋ mill › <page>` in the board header's place, two panes with a list of at most 24 columns, an inspector on settings, the status line and the five-key help bar; `tab` switches panes, `1`–`9` and `[`/`]` jump; a one-line strip under 60 columns; every state in a word or glyph (§18.1.1).
- **Settings:** sections left with changed-count badges, rows right with values flush right and `•` for changed, an inspector with id, default, when it applies, options and help (`K` for all of it), search across sections with an empty state; `u`, `p`, `gd`, `i` and the Drafter section with its status row are kept (§18.10).
- **Help:** the eight groups left, rows right at full height (nothing dropped), `/` filter across groups, a live legend, `(off)` rows, `?` opens the current screen's group, `:help <topic|command|code>` and `:help commands` open the page on the right row (§18.5.1).
- **New pipeline:** a page with a step rail (`start › shape › run › review` with marks and words), the step pane and, at 110+ columns, the graph preview; errors and loading lines stay on the page and keep the draft; the agent list shows missing agents dimmed. The four steps, name last, dry run then edit, the drafter, import, duplicates starting manual and the modal keys are unchanged (§18.8).
- **CLI parity:** §17.5 names the pages and adds help (`mill <command> --help`, `mill explain`).

## Contents

1. [Summary](01-summary.md#1-summary)
2. [Principles](02-principles.md#2-principles)
3. [Goals and non-goals](03-goals.md#3-goals-and-non-goals)
4. [Concepts and glossary](04-concepts.md#4-concepts-and-glossary)
5. [Architecture](05-architecture.md#5-architecture)
6. [Data model: items, artifacts, workspaces](06-data-model.md#6-data-model-items-artifacts-workspaces)
7. [Pipeline file format](07-pipeline-format.md#7-pipeline-file-format)
8. [Expressions and templating](08-expressions.md#8-expressions-and-templating)
9. [Parts and plugins](09-parts-and-plugins.md#9-parts-and-plugins)
10. [Built-in parts catalog](10-builtin-parts.md#10-built-in-parts-catalog)
11. [Agents](11-agents.md#11-agents)
12. [Runners](12-runners.md#12-runners)
13. [Triggers](13-triggers.md#13-triggers)
14. [Piping and Unix composability](14-piping.md#14-piping-and-unix-composability)
15. [Execution semantics](15-execution.md#15-execution-semantics)
16. [Daemon wire protocol](16-wire-protocol.md#16-daemon-wire-protocol)
17. [CLI reference, including working without the TUI](17-cli.md#17-cli-reference)
18. [TUI reference](18-tui.md#18-tui-reference)
19. [Configuration (Lua)](19-config-lua.md#19-configuration-lua)
20. [Files and paths](20-paths.md#20-files-and-paths)
21. [Secrets](21-secrets.md#21-secrets)
22. [Security and trust](22-security.md#22-security-and-trust)
23. [Integrations and interop](23-integrations.md#23-integrations-and-interop)
24. [Observability and cost](24-observability.md#24-observability-and-cost)
25. [Optional hosted layer](25-hosted.md#25-optional-hosted-layer)
26. [Example pipelines](26-examples.md#26-example-pipelines)
27. [Tech stack (Go) and repository layout](27-tech-stack.md#27-tech-stack-go-and-repository-layout)
28. [Roadmap](28-roadmap.md#28-roadmap)
29. [Errors and diagnostics](29-errors.md#29-errors-and-diagnostics)
30. [Open questions](30-open-questions.md#30-open-questions)

[Appendix A. Reference soundsets](appendix-a-soundsets.md#appendix-a-reference-soundsets) · [Appendix B. Walkthrough: from install to result](appendix-b-walkthrough.md#appendix-b-walkthrough-from-install-to-result)

