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
> **Status:** specification draft, **v2**. v2 keeps everything the engine does and redesigns the surface around rex's design rules: one board, one line per pipeline, everything readable at a glance (§18.1). No code yet. When this spec and the eventual source disagree, the source wins, and this document gets fixed.

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
| Seven-step wizard | **Five steps:** describe, trigger, agent, limits, review (§18.8) |
| Help bars with up to nine keys | **Five keys:** `enter open · n new · e edit · R run · ? more` |
| Bottom prompt unused | **`λ` prompt runs things:** a pipeline with inputs, or a pipe expression (§18.2) |
| Plugins as a tab | `:plugins` overlay; `:stats` added (§18.9, §18.12) |
| A blank screen until the board loaded | **Boot splash:** a clone of rex's bootloader with mill's own steps, `[  OK  ]`/`[ WARN ]`/`[ SKIP ]`/`[ FAIL ]` rows, and a cause and fix when boot fails (§18.13) |

Everything else (engine, parts and plugins, Lua config, agents, triggers, piping, errors, security, sound, Go) carries over from v1 unchanged except where it referred to the old screens.

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

