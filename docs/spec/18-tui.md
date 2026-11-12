## 18. TUI reference

The TUI starts when you run `mill` with no subcommand. It boots in front of you (the boot splash, §18.13), connects to the daemon and shows **the board**: every pipeline, one line each, sorted by what deserves a look. Everything else (a run, the editor, the wizard, plugins, settings) opens over the board and closes back to it. There are no tabs.

### 18.1 Design rules

These rules come from rex and apply to every mill screen. A change that breaks one needs a reason written next to it.

| # | Rule | In practice |
|---|---|---|
| 1 | **One screen.** | The board is home. Everything else is a popup or a focus mode over it, and `esc` always returns to it. |
| 2 | **A row shows what you need to decide, nothing more.** | State, name, text, recent runs, age. Everything else is one `v` away. |
| 3 | **Columns appear only when a row needs them.** | The cost, long-run badge and budget columns, and the tag filter row, are hidden until some row has something to show in them. |
| 4 | **The header is the summary.** | `1 failed · 2 running · 14 ok today · $6.20 today`. If you read one line, read that one. |
| 5 | **Five keys in the help bar.** | `enter open · n new · e edit · R run · ? more`. Everything else lives under `?`. |
| 6 | **Rows fill the width exactly and never wrap.** | On narrow terminals, columns give way in a fixed order (§18.3); the text column shrinks last. |
| 7 | **Long sections fold.** | READY with more than 5 rows folds into its title line: `READY  12 hidden · 3 show`. |
| 8 | **Color is state, and only state.** | Red failed · amber retrying or running long · blue running · green ok · grey everything else. At most one colored word per row. |
| 9 | **Numbers are compact and aligned.** | `12m`, `812K`, `$4.10`, right-aligned in their column. Relative times on screen, absolute times in JSON. |
| 10 | **Overlays are square, sized to fit, and swallow the mouse.** | Width `clamp(w-8, 60, 110)`. A click never changes the selection behind an overlay. |
| 11 | **Nothing moves unless it's running.** | Spinners only on running work. `reduce_motion` stops them, and the screensaver's drift. |
| 12 | **Every screen has a command.** | Each popup and overlay maps to a CLI command (§17.5). Nothing needs the TUI. |

**Simple surface, deep engine.** The board stays one line per pipeline however much happens underneath. Scheduling, retries, budgets, plugins, MCP and crash recovery all happen in `milld` and only reach the screen as a state, a word of text, or a square in the run history.

### 18.2 The board

```
 ≋ mill                                         1 failed · 2 running · 14 ok today · $6.20 today

 FAILED  1
 ▸ ✗  nightly-sweep    build: make check failed on all 3 tries          ■■■■■■■■■✗    9h
 RUNNING  2
   ↻  issue-to-pr      build · rate limited, retry in 40s               ■■■■■■■■■↻   12m
   ⠋  inbox-triage     classify · 4 of 9 messages                       ■■■■■■■■■■   12s
 READY  4
   ✓  morning-brief    posted digest to #daily · next 07:30             ■■■■■■■■■■    4h
   ✓  docs-drift       no drift · next on merge                         ■■■■■■■■✗■    2h
   ✓  release-notes    draft release v1.4.0 · next on tag               ■■■■■■■■■■    3d
   ○  best-of-3        never run · R for a dry run                                    -

 λ
 enter open · n new · e edit · R run · ? more
```

**Sections.** Pipelines sit in three sections by their latest state: **FAILED**, **RUNNING** (including retrying) and **READY** (ok, never run, disabled). Each title shows its count. `1`, `2`, `3` jump to a section. On a long READY, `3` also shows and hides it. When a pipeline changes section (a run starts or fails), the selection stays on that pipeline.

**Header.** One line: `N failed · N running · N ok today · $X today`. `failed` is red while above zero. With `v`, it adds `· N runs today · p50 4m`.

**Prompt.** `i` focuses the bottom prompt (`λ`). It runs whatever you type: a pipeline name with inputs (`fix-flaky-test test=TestCache_Expiry`) or a pipe expression (`sh "git log -5" | agent claude/haiku "summarize"`, §14.2). `tab` completes pipeline names and input keys. The new run's pipeline is selected when it starts.

**Filter, sort, find.**
- `t` cycles the tag filter (`all · code · daily · mail …`). The filter row only shows while pipelines use more than one tag or a filter is active.
- `/` searches names, descriptions and tags. Tokens like `state:failed`, `trigger:cron`, `uses:linear` and `ok:<90` narrow further:

| Token | Matches |
|---|---|
| `state:failed`, `state:!ok` | Latest run state (`!` negates) |
| `tag:code` | Pipeline tags |
| `trigger:cron`, `trigger:webhook` | Trigger kind |
| `scope:project`, `scope:user` | Where the pipeline lives |
| `uses:linear` | Pipelines that use a plugin or part |
| `ok:<90`, `cost:>1`, `ran:<24h` | Numeric comparisons on success rate, typical cost, last run |

- `s` cycles the sort *inside* each section (attention, name, last run, success rate, cost), and `-` reverses it.
- Saved views from Lua (`mill.view "code" { filter = "tag:code" }`) are switched with `V`.

### 18.3 Row columns

By default a row shows five things: **state, name, text, runs, age**.

```
 ▸ ✗  nightly-sweep    build: make check failed on all 3 tries          ■■■■■■■■■✗    9h
     state name        text                                             runs          age
```

- **text** is the one thing worth knowing about that pipeline right now:

  | State | Text |
  |---|---|
  | failed | Where and why: `build: make check failed on all 3 tries` |
  | running | The current node and its progress: `build · loop 2/3` |
  | retrying | Why and when: `build · rate limited, retry in 40s` (amber) |
  | ok | The last result, then the next run: `draft PR #1293 · next on webhook`. The result comes from the run's `outputs.summary`, or the first output. |
  | never run | `never run · R for a dry run` |
  | disabled | `disabled · :enable to turn its triggers back on` |

- **runs** is the last 10 runs as squares, oldest to newest: `■` green ok, `✗` red failed, `↻` amber retrying, `■` blue running.
- **age** is the time since the pipeline's last event, right-aligned against the edge. `-` means never.

`v` switches to the **detailed view**, which adds the columns you need to compare pipelines:

```
 ▸ ✗  nightly-sweep    build: make check failed (3/3)    ■■■■■■■■■■■■■■■■■■■✗   90%   6m · $0.30   cron 03:00   in 14h    9h
     state name        text                              runs (20)              ok    typical      trigger      next      age
```

**Conditional columns** (rule 3): they appear in both views only while a row needs them:

| Column | Shows while | Looks like |
|---|---|---|
| **cost today** | Any pipeline has spent something today | `$4.10`, right-aligned |
| **long-run badge** | Any run is taking well over its usual time | `▴2×` (§18.12) |
| **budget** | Any running run has used more than 80% of a budget | `82% $` in amber, `100%` in red when stopped |
| **scope** | Both project and user pipelines are shown | `p` / `u`, dim |

**Narrow terminals.** Columns give way in this order until the text has 20 columns: scope, trigger, typical, ok, cost today, budget, then the runs strip shrinks to 5 squares, then the long-run badge. After that the text shrinks, then the name (down to 12).

### 18.4 Focus modes

| Mode | Enter | Exit |
|---|---|---|
| Boot | launching `mill` | hands off to the board (or setup on first launch); `q`, `esc`, `ctrl+c` quit |
| Board | default | |
| Prompt | `i` | `esc` |
| Search | `/` | `enter` keeps it, `esc` clears it |
| Command | `:` | `esc`, or after the command runs |
| Run | `enter` on a pipeline | `esc` |
| Log | `L` in a run | `L` or `esc` |
| Agent screen | `enter` on an agent node in a run | `esc`, `qq`, `ctrl+]` |
| Editor | `e` | `esc` unwinds one layer, then asks to save if changed; `:w`, `:q`. Inside: `NORMAL`, `VISUAL`, `CONNECT`, `INSERT` (§18.7.4) |
| Wizard | `n` or `:new` | `esc` cancels, `enter` on review creates |
| Plugins | `:plugins` | `esc` |
| Settings | `S` or `:settings` | `esc` |
| Stats | `:stats` | `esc` |
| Help | `?` | `esc`, `?` |
| Confirm | `dd`, `:disable`, `:q` | `y` / `n` |

The mode name shows at the left of the status line, the way vim shows `-- INSERT --`.

### 18.5 Keys and mouse

**Board**

| Key | Action |
|---|---|
| `j`, `k`, `↓`, `↑` | Move selection |
| `g`, `G` | First / last row |
| `1`, `2`, `3` | Jump to Failed / Running / Ready; on a long Ready, `3` shows and hides it |
| `tab` | Next failed pipeline (wraps) |
| `enter` | Open the latest run (§18.6) |
| `R` | Run now (a dry run if it has never run) |
| `F` | Rerun the latest failed run from the node that failed |
| `e` | Open in the editor (§18.7) |
| `n` | New pipeline wizard (§18.8) |
| `i` | Focus the prompt |
| `v` | Simple / detailed rows |
| `t` | Cycle the tag filter |
| `/` | Search |
| `s`, `-` | Cycle sort / reverse it |
| `V` | Cycle saved views |
| `p` | Toggle the preview line under the selected row: description, trigger, budget, last outputs |
| `x` | Disable / enable the selected pipeline's triggers (confirm) |
| `dd` | Delete the selected pipeline (confirm; the file goes to `.mill/trash/`) |
| `:` | Command mode |
| `?` | Help |
| `S` | Settings |
| `q`, `ctrl+c` | Quit (confirm with `:q`) |

**Mouse.** With `mouse_enabled` on (the default): click a row to select it, double-click to open its run, scroll to move the selection, click a filter chip to set the filter. Overlays and prompts swallow mouse events. Hold Option (iTerm2, Terminal.app) or Shift (Ghostty, kitty) to select text natively, or turn `mouse_enabled` off.

**Help overlay.** `?` opens a square panel. At 96+ columns it has two columns: Navigation, Actions, Mouse, Modes on the left; Commands, Run, Editor, Badges on the right. Narrower terminals get one column. If it doesn't fit, the badges and then the mouse groups are dropped with a dim `(truncated, enlarge terminal)` note. It lists every `:` command.

### 18.6 Run

`enter` opens the selected pipeline's latest run in a popup (`mill runs` and `:runs` list older ones; `enter` on any of them opens it the same way). One row per node, in run order, with a timeline column, so the whole run reads in one glance:

```
┌ issue-to-pr#42 · ↻ retrying · 12m · 812K tk · $4.10 of $15 ───────────────────────────────────────┐
│                                                                                                   │
│  ✓ webhook    event #1287 "Webhook sender drops events on 429"  ▏                             0s  │
│  ✓ ws         mill/issue-1287                                   ▏                             1s  │
│  ✓ plan       claude/opus · risk medium, 3 files · 210K tk       ████████████              5m38s  │
│  ✓ scope      pass                                                           ▏                0s  │
│  ↻ build      rate limited · retry in 40s · loop 2/3                          ████░░░      2m11s  │
│  ○ review                                                                                         │
│  ○ pr                                                                                             │
│  ○ ping                                                                                           │
│  – hand-back  skipped · scope passed                                                              │
│                                                                                                   │
│  enter agent · L log · F rerun from here · i items · esc back                                     │
│                                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

- **Columns:** state, node, what it did (or is doing), a timeline bar on the run's own clock (`░` is time spent waiting to retry), and duration. Nodes in `each` mode show `3/5` and expand with `za`.
- **Title:** run, state, elapsed time, tokens, and cost against the budget. The budget turns amber over 80%.
- **`enter` on an agent node** shows its screen, view-only, repainted by the daemon so it appears at once. `esc`, `qq` or `ctrl+]` goes back.
- **`L`** turns the popup into the log (below). **`i`** shows the selected node's input and output items. **`F`** reruns from the selected node, reusing earlier outputs. **`c`** cancels the run (confirm).
- **Branches** show as indented rows under the node that split (`├─ true`, `└─ else`). Skipped branches are dim with the reason.
- **Live:** the popup follows the run. When it finishes, the title turns green or red and the result line (`draft PR #1293`) appears at the bottom.

**Log.** `L` inside a run shows its log in the same popup:
- One line per event: relative line numbers (the current line shows its absolute number), time, a colored bar per node, node, level, message.
- Agent output is folded under the line that started it (`▸ 214 lines of agent output`). `za`, `zo`, `zc` open and close folds.
- `/` searches and highlights. `n` and `N` jump between hits. `f` follows new lines. `o` shows only the selected node. `gx` explains an error code on the line.
- `L` or `esc` goes back to the run.

### 18.7 Editor

`e` on the board opens the selected pipeline in the editor, a full-screen focus mode. It is where pipelines are made: you add parts, connect their ports, branch, loop, fix what lint finds, fill in prompts and expressions, and try the result with a dry run, without leaving the terminal and without ever writing YAML by hand (though you always can). `esc` unwinds one layer at a time and finally returns to the board, asking to save if anything changed.

The editor is the TUI face of the same edit operations the CLI exposes (§17.8, §18.7.16). Every action has a command.

#### 18.7.1 Rules

| Rule | In practice |
|---|---|
| **Files are the truth.** | Every editor action is a change to the pipeline YAML, or to the layout sidecar for positions (§7.6). There is no hidden editor-only state. View preferences (zoom, pane widths, the minimap, the last selection) are saved in `tui-state.json` and never change behavior. Pinned samples (§18.7.12) are files too. |
| **The buffer is YAML.** | The editor holds the pipeline as a YAML document (the `yaml.v3` node tree), not as a separate model. The graph, the inspector, lint and the layout are all derived from it on every change. That's what keeps comments, key order and formatting intact, and what makes the YAML pane (`Y`) and the graph two views of one thing. |
| **One edit, one undo step.** | Each action (add, connect, rename, a quick fix, a field edit) is one entry on the undo stack, with the description shown in the status line (`undo: connect check.true → judge`). |
| **Hands-off.** | The palette has no approval, question or "wait for a person" parts, and lint refuses any that a plugin offers (`M3206`). Where a person is needed, the editor offers a `handoff`, which ends the branch (§10.3). |
| **Same operations as the CLI.** | The TUI and `mill node`, `mill edge`, `mill wrap`, `mill lint --fix` call the same package (`internal/pipeline/edit`). A change you can make with a key you can make with a command, and both produce the same YAML. |
| **Never silently lossy.** | When an action would drop something (deleting a node that others reference, disconnecting an edge that comes from an expression), the status line says exactly what will change, and lint shows what broke. Nothing is removed from the file that you didn't remove. |

**Design rules bent here** (§18.1 asks for the reason next to each):

| Rule | Bend | Reason |
|---|---|---|
| 1 `esc` returns to the board | `esc` unwinds one layer: completion list → field → inspector → panel → canvas → board | The editor nests (a field inside the inspector inside the editor). One `esc` per layer is predictable; a single `esc` that threw away a half-typed expression would not be. |
| 5 Five keys in the help bar | Still five, but the five change with the mode and focus (canvas, inspector, connect, palette, panel) | Each mode has different first actions. The bar never shows more than five. |
| 8 Color is state, and only state | Colors on the graph mean lint state (red error, amber warning) or, during a dry run, run state. The diff view uses green and red for added and removed lines. Selection is a background, never a color. | Lint and dry-run results *are* the states of an editor. Diff colors are a convention people read without thinking. |
| 10 Overlays are square and centered | The completion list and pickers are square, but anchored next to the field or node they belong to | They must stay beside the text being typed or the port being chosen. |

#### 18.7.2 Screen

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⊢ gate · assert ────────────────┐
│                                                                                       ││ WITH                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ that*       ${{ nodes.judge.ite…│
│ ┌ ⧉ fanout ─────┐      ┌ ⊨ check ────!─┐      ┌ ◭ judge ──────┐      ┌ ⊢ gate ─────✗─┐││   ✗ M3014 no field "scroe" on … │
│▶│agent.fanout   ├─────▶│check each true├─────▶│agent.judge    ├─────▶│assert         ├││ else        ◂ route ▸           │
│ │×3 opus·gpt-5·…│┐     │make che… false│◌     │rubric.md      │      │judge.scr… else├││ NODE                            │
│ └───────────────┘│  ·  └───────────────┘·   · └───────────────┘ ·   ·└───────────────┘││ id          gate                │
│  └───────────────┘                                                                    ││ uses        assert              │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ in          judge ⋯             │
│                                                                                       ││ mode        batch default       │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ when        –                   │
│                                                                                       ││ needs       –                   │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ runner      local (setting)     │
│                                                                                       ││ workspace   inherit default     │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ retry.max   –                   │
◂                                                                                       ▸│ timeout     –                   │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ cache       false default       │
│                                                                                       ││ on_error    fail default        │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ discard     –                   │
│                                                                                       ││ output      –                   │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ LINT                            │
│                                                                                       ││ ✗ M3014 with.that uses judge.s… │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││   ga use score (+1)             │
│                                                                                       ││ TEST                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ in   1 item · no sample         │
│                                                                                       ││ out  –                          │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ est  ≈ 2s                       │
│                                                                                       ││                                 │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││                                 │
│                                                                                       ││                                 │
└───────────────────────────────────────────────────────────────────────────────────────┘└─────────────────────────────────┘
 NORMAL  best-of-3 · 7 nodes · 6 edges   3 lint errors · ]e walks through them, ga fixes the one selected   gate  cards 5,0
 a add · c connect · enter inspect · ga fix · ? more
```

| Area | What it shows |
|---|---|
| **Header** | The file, `[+]` when the buffer differs from the file on disk, then on the right the estimated cost per run against the budget (§18.7.12) and the lint summary (`✗ 3  ! 1`, or `✓ lint`). During a dry run the right side shows its progress instead. |
| **Canvas** | The graph (§18.7.3). Scrolls to keep the selection visible; `zh zj zk zl` pan by a quarter screen. |
| **Right pane** | One of three, never more: the **inspector** for the selected node (§18.7.10), the **pipeline panel** (`gp`, §18.7.11), or the **YAML pane** (`Y`, §18.7.13). |
| **Bottom panel** | Opens over the lower part of the canvas when asked for: **problems** (`!`), **items** (`gi`), **references** (`gr`). One at a time, eight rows. |
| **Status line** | The mode (`NORMAL`, `VISUAL`, `CONNECT`, `INSERT`), the pipeline and its size, a context line (the focused edge and its schema, the connect check, a what-if, the last undo), and on the right the selected node, the zoom and the slot. |
| **Help bar** | Five keys for the current mode and focus (rule 5 above). |

**Narrow terminals.** Below 110 columns the right pane narrows to 28 columns. Below 96 it hides, and `enter` shows the inspector as a square overlay over the canvas instead. Below 80 columns the canvas starts at the compact zoom. The editor needs at least 80×24.

#### 18.7.3 How the graph is drawn

**Layout.** The graph flows left to right in layers: triggers in the first column, every node one column to the right of its furthest upstream node. Within a column, a node takes the row of its first upstream when that row is free, so the main path stays on one line and branches drop below it. Each node sits in a **slot** (column, row). The layout is deterministic: the same file always draws the same way, which is also what `mill render` prints.

**Manual positions.** `H J K L` move the selected nodes one slot and pin them there. Pins are saved in the layout sidecar (§7.6), never in the pipeline. A node can't be moved left of its upstream or right of its downstream: edges always point right, which keeps routing readable in a cell grid. `g=` unpins the selection and `:layout reset` unpins everything.

**Nodes.** At the default zoom each node is a card, 17 cells wide:

```
┌ ◭ judge ────✗┐      title: glyph, id, and a lint badge (✗ error, ! warning)
│agent.judge    │      line 1: the part and its key setting
│claude/opus    ├──▶   line 2: badges (below), or a second key setting
└───────────────┘
```

| Badge | Meaning |
|---|---|
| `each×4` | `mode: each` with `concurrency: 4` (`each` without a limit); `stream` for stream mode |
| `?` | Has a `when`; the inspector shows the expression |
| `↻2` | `retry.max` is 2 |
| `⌛10m` | `timeout` |
| `⊙` | `cache` is on |
| `⇢err` | `on_error: route` (the node grows an `error` port) |
| `$0.94` | Estimated cost per run (§18.7.12), when estimates are on (`ze`) |

**Glyphs.** Each part has a glyph. Built-ins: `⌁` trigger, `$` sh, exec · `⊨` check · `≡` data parts · `?` if · `⋔` switch · `⋈` merge · `↻` loop, agent.loop · `⊢` assert · `↪` handoff · `◷` wait · `▣` pipeline · `◆` agent · `⧉` agent.fanout · `◭` agent.judge · `◇` agent.review · `⊙` agent.extract · `⑂` git parts · `⇡` gh parts · `⇄` http, openapi · `◎` mcp · `▤` file parts · `♪` notify. Plugins set their own (`glyph` in the part manifest, §9.7).

**Special nodes.**

| Node | Drawn as |
|---|---|
| Trigger | Rounded card (`╭╮╰╯`) in the first column, one per entry under `on:`. Every entry node (a node with no inputs) is connected from the triggers. With no `on:`, a dim `⌁ manual` card stands in for the implicit manual trigger; it isn't in the file. |
| `handoff`, `fail` | A card with a double right side (`╖ ║ ╜`) and no output port: the branch ends here. |
| `agent.fanout` | A card with a second card drawn behind it, and `×3` with the candidates' agents on line 2. `zo` lists each candidate under it (read-only, from `with.agents`). |
| `loop` | A dashed container (`┌┄┐`) around its body. The container *is* the loop node: its title row (`↻ retry · until test.passed · ≤3`) is what you select. Edges into the loop enter its first body node through the left wall; `out` leaves through the right wall (`┼`) and `exhausted` through the bottom border, labeled there. The bottom border reads `↺ next try`. The last body node's ports feed the loop, so they're never flagged as unconnected. `zc` folds the container into one card. |
| `pipeline` | A card with `▣`; `zo` draws the sub-pipeline inside a dashed container, read-only, and `enter` on it opens that pipeline in the editor (`ctrl+o` comes back). |

**Ports.** The input port is on the left side, output ports are on the right, one row each: `out` first (unlabeled), then the named ones (`true`, `false`, `case.<name>`, `default`, `else`, `exhausted`, `error`) labeled inside the card. A card grows taller when it has more than two output ports. An output port with nothing connected is drawn as `◌` after its label in amber, because items sent there are dropped (`M3030`). A port listed in the node's `discard` (§7.3) is drawn as `⌀`: dropped on purpose.

**Edges.**

| Edge | Drawn as | Comes from |
|---|---|---|
| Data | `──▶` solid | `in:`, explicit `edges:`, or inferred from a `${{ nodes.x… }}` reference (§7.4) |
| Inferred | `──▶` solid; marked `⋯` in the inspector's `in` row and in the status line when focused | A reference, when the node has no `in:`. Removing it means removing the reference (§18.7.7). |
| Ordering | `┄┄▷` dotted | `needs:`, or a reference to a node that isn't already upstream of a node with an explicit `in:` |
| From a trigger | `──▶` dim | Implicit: every entry node starts when the run starts |
| Branch | `─ false ─▶` | A named port; the label sits on the first straight run of the edge |

Edges route in right angles through the gaps between cards and merge into trunks (`├`, `┬`) where one port feeds several nodes. During a dry run, an edge that carried items shows the count before its arrow (`─ 3 ─▶`).

**Zoom.** `+` and `-` step through three levels; `=` fits the whole graph at the largest level that fits.

| Level | Node | Use |
|---|---|---|
| cards | 17×4 card with badges and port labels | Editing; the default |
| compact | One line: `[◆ plan    ✗]` | Seeing a whole pipeline of 20–40 nodes |
| dots | The glyph only; the id shows for the selected node | The shape of very large graphs |

**Minimap and outline.** `M` shows a minimap in the canvas corner: every node as `▪`, the selection as `■`, and the visible area as a box. `O` replaces the canvas with the outline (the same tree `mill show --outline` prints, §17.7); `enter` on a line goes back to the canvas on that node.

#### 18.7.4 Modes and focus

| Mode | Enter | Does | Leave |
|---|---|---|---|
| `NORMAL` | default | Move, select, every one-key action | |
| `VISUAL` | `v` | Multi-select: `hjkl` moves the cursor, `space` adds or removes a node, `V` adds everything downstream. Actions (`y`, `dd`, `H J K L`, `gl`, `gP`, alignment) apply to the whole selection. | `esc` |
| `CONNECT` | `c`, `C` | Draws a pending edge from a port; `hjkl` chooses the target (§18.7.7) | `enter` connects, `esc` cancels |
| `INSERT` | `enter` or `i` on a field, typing in the palette, the YAML pane | Text entry with completion | `enter` (single line) or `esc` |

Focus is on the **canvas**, the **right pane** or the **bottom panel**. `enter` on a node moves focus to the inspector; `esc` moves it back. The mode and focus show at the left of the status line.

#### 18.7.5 Moving around

| Key | Action |
|---|---|
| `h j k l`, arrows | Move to the nearest node in that direction that shares its row or column, or nowhere |
| `w` / `b` | Next / previous node in run order (topological) |
| `gg` / `G` | First trigger / last node |
| `/` | Jump to a node: fuzzy search over ids, parts and settings; `enter` selects |
| `tab` / `shift+tab` | Focus the selected node's edges one at a time (inputs, then outputs) |
| `gd` | Go to definition: on the canvas, the selected node's upstream (repeat to cycle); in the inspector, the node named in the expression under the cursor |
| `gr` | Find references: every field, `when`, output and edge that uses the selected node, in the references panel |
| `ctrl+o` | Back to where you were before the last jump |
| `zz` | Center the view on the selection |
| `zh zj zk zl` | Pan a quarter screen |
| `+` `-` `=` | Zoom in, out, fit |
| `M` / `O` | Minimap / outline |

#### 18.7.6 Making nodes

**The palette.** `a` opens a searchable palette over the canvas:

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ◭ judge · agent.judge ──────────┐
│                                                                                       ││ WITH                            │
│ ·┌ a insert after judge ────────────────────────────────────────────────────────────┐ ││ rubric*     @prompts/rubric.md  │
│ ┌│ all  exec  data  control  agents  git/gh  io  mcp  plugins  triggers  patterns   │┐││ agent       claude/opus         │
│▶││ > rev▌                                                                           │├││ NODE                            │
│ ││──────────────────────────────────────────────────────────────────────────────────│├││ id          judge               │
│ └│ ⁂ review → PR              independent review, ap…   │ ⁂ review → PR             │┘││ uses        agent.judge         │
│  │ ◇ agent.review             independent review of …   │ pattern                   │ ││ in          check.true          │
│ ·│                                                      │                           │ ││ mode        batch default       │
│  │                                                      │ independent review,       │ ││ when        –                   │
│ ·│                                                      │ approved → draft PR, else │ ││ needs       –                   │
│  │                                                      │ a comment                 │ ││ runner      local (setting)     │
│ ·│                                                      │                           │ ││ workspace   inherit default     │
│  │                                                      │ inserts review → approved │ ││ retry.max   –                   │
│ ·│                                                      │ → pr → review-notes       │ ││ timeout     –                   │
◂  │                                                      │                           │ ▸│ cache       false default       │
│ ·│                                                      │                           │ ││ on_error    fail default        │
│  │                                                      │                           │ ││ output      { winner: string, s…│
│ ·│                                                      │                           │ ││ TEST                            │
│  │                                                      │                           │ ││ in   3 items · pinned sample    │
│ ·│                                                      │                           │ ││ out  {"winner":"claude/opus","… │
│  │                                                      │                           │ ││ est  ≈ $0.28 · 1m00s            │
│ ·│                                                      │                           │ ││                                 │
│  │                                                      │                           │ ││                                 │
│ ·│                                                      │                           │ ││                                 │
│  │                                                      │ insert after judge        │ ││                                 │
│ ·│                                                                                  │ ││                                 │
│  └─ type search · tab category · enter insert · esc close ──────────────────────────┘ ││                                 │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││                                 │
   ⋮
 INSERT  best-of-3 · 7 nodes · 6 edges   3 lint errors · ]e walks through them, ga fixes the one selected  judge  cards 4,0
 type search · tab category · enter insert · esc close
```

- **Categories:** `all`, `exec`, `data`, `control`, `agents`, `git/gh`, `io`, `mcp`, `plugins`, `triggers`, `patterns`. `tab` and `shift+tab` move between them. Typing filters with fuzzy matching over the part id, its description and its plugin. MCP tools appear as `mcp:<server>/<tool>`, plugin parts with their plugin's name on the right.
- **Each row:** glyph, part id, a short description, flags (`destructive` in muted amber; `needs ws` when the part works on a workspace), and where it comes from (`built-in`, a plugin, an MCP server).
- **The side column** shows the selected part's ports, required settings and what will happen: `insert between scope.out and build`.
- **Parts that wait for a person** (a plugin part whose manifest says `interactive`) are listed greyed with `waits for a person · M3206`, and can't be inserted. Use a `handoff` instead.
- The palette remembers the last category and query for this session.

**Where the new node goes.** The action that opened the palette decides it, and the side column says it before you press `enter`:

| Key | Inserts | YAML change |
|---|---|---|
| `a` | After the selected port. If that port already feeds nodes, the new node goes in between. | New node with `in: <node>[.<port>]`; the old consumers' `in` now names the new node |
| `i` | Before the selected node, between it and its upstream | New node takes the old `in`; the selected node gets `in: <new>` |
| `o` | A new branch from the selected port, next to whatever it already feeds | New node with `in: <node>[.<port>]`; nothing else changes |
| `a` with an edge focused | On that edge only | As `a`, for one consumer |
| `gt` | A trigger (palette opens on `triggers`) | New entry under `on:` |

New nodes get an id from the part (`gh.pr` → `pr`, then `pr-2`), the part's required settings filled with placeholders (`‹prompt›`), and focus moves to the first placeholder in the inspector. Unfilled placeholders are lint errors (`M3003`), so `]e` walks you through what still needs an answer.

**Copy, duplicate, delete, rename.**

| Key | Action |
|---|---|
| `yy` | Yank the selected nodes (with `VISUAL`, the subgraph and the edges inside it) |
| `p` / `P` | Paste after the selected port, connected / paste unconnected below the selection. Ids get a suffix (`check-2`) and references inside the pasted nodes are rewritten to the copies. |
| `dd` | Delete. A node with one input and one output is healed over (its upstream connects to its downstream). Otherwise its edges go with it and lint shows what's now unconnected. The status line names every reference that now points nowhere. |
| `r` | Rename the id. The prompt shows how many references will change (`4 references in 3 nodes and outputs`); every `in`, `needs`, loop `body`, `${{ nodes.<id>… }}`, output, layout pin and sample moves with it. |

**Patterns.** The `patterns` category inserts small, pre-wired groups of nodes after the selected port. Their placeholders are lint errors until filled. Patterns are YAML files: built-ins ship embedded, plugins can ship more (`provides.patterns`), and `:pattern save <name>` saves the current selection to `~/.config/mill/patterns/<name>.mill.yaml`.

| Pattern | Inserts |
|---|---|
| plan → check → handoff | `plan` (agent with an output schema: plan, files, risk) → `scope` (assert on risk and size, `else: route`) → `hand-back` (handoff on `scope.else`). Continues from `scope`. |
| best of N | `fanout` (agent.fanout, three agents) → `check` (`each`, `make check`, `false` discarded) → `judge` (agent.judge on `check.true`) |
| fix until green | A `loop` with `fix` (agent) → `test` (check) in its body, `until: ${{ nodes.test.item.passed }}`, `max_iters: 3`, and a handoff on `exhausted` |
| review → PR | `review` (agent.review of the upstream diff) → `approved` (assert verdict, `else: route`) → `pr` (gh.pr, draft) and a handoff comment on `approved.else` |
| triage with switch | `classify` (agent.extract with an enum field) → `route` (switch with one case per value, `default` discarded) |
| each with a limit | `split` → `work` (agent, `each`, `concurrency: 2`) → `gather` (merge `all`) |
| failure → handoff | Sets `on_error: route` on the selected node and adds a handoff on its `error` port |

**Wrap and extract.**

| Key | Action | YAML change |
|---|---|---|
| `gl` | Wrap the selection in a loop. The selection must be a connected chain with one way in and one way out. | New `loop` node with `with.body` listing the selection, `until: ‹until›`, `max_iters: 3`; the first body node reads `in: <loop>.body`; downstream reads from the loop |
| `gP` | Extract the selection to a sub-pipeline | A new file `<name>.mill.yaml` with the selection; references across the cut become its `inputs` and `outputs`; the selection is replaced by one `pipeline` node. A confirmation lists both files' changes first. |
| `gU` | Unwrap a loop, or inline a `pipeline` node | The inverse of the two above |

#### 18.7.7 Connecting

`c` starts a connection from the selected node. If it has more than one output port and none is focused, a **port picker** opens next to the node:

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⊨ check · check ────────────────┐
│                                                                                       ││ WITH                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ sh*         make check          │
│ ┌ ⧉ fanout ─────┐      ┌ ⊨ check ────!─┐  ┌ connect which port of check? ──┐e ─────✗─┐││ ok_codes    –                   │
│▶│agent.fanout   ├─────▶│check each true├──│ 1   true               → judge │         ├││ NODE                            │
│ │×3 opus·gpt-5·…│┐     │make che… false│◌ │ 2   false      ◌ not connected │scr… else├││ id          check               │
│ └───────────────┘│  ·  └───────────────┘· │                                │─────────┘││ uses        check               │
│  └───────────────┘                        └─ j/k · enter · esc ────────────┘          ││ in          fanout              │
   ⋮
 NORMAL  best-of-3 · 7 nodes · 6 edges   3 lint errors · ]e walks through them, ga fixes the one selected  check  cards 3,0
 j/k choose · enter ok · esc cancel
```

After `2` (the unconnected `false` port), `CONNECT` mode with `judge` as the first target; `hjkl` moves the target:

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⊨ check · check ────────────────┐
│                                                                                       ││ WITH                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ sh*         make check          │
│ ┌ ⧉ fanout ─────┐      ┌ ⊨ check ────!─┐      ┌ ◭ judge ──────┐      ┌ ⊢ gate ─────✗─┐││ ok_codes    –                   │
│▶│agent.fanout   ├─────▶│check each true├───╌╌▶│agent.judge    ├─────▶│assert         ├││ NODE                            │
│ │×3 opus·gpt-5·…│┐     │make che… false│╌╌╌┆  │rubric.md      │      │judge.scr… else├││ id          check               │
│ └───────────────┘│  ·  └───────────────┘·   · └───────────────┘ ·   ·└───────────────┘││ uses        check               │
 CONNECT  best-of-3 · 7 nodes · 6 edges   check.false → judge ✓                                            check  cards 3,0
 hjkl target · enter connect · a new node · n needs · esc cancel
```

1. Pick the port (`j`/`k` and `enter`, or its number). Ports already connected are marked; connecting one again adds a second consumer.
2. `CONNECT` mode draws a pending dashed edge. `h j k l` move the target between nodes. Targets that would make a cycle outside a loop are dim and refused (`M3020`); handoffs and triggers can't be targets.
3. **The check runs as you move.** The status line compares what the source produces with what the target uses: `judge.out {winner, score, reason} → gate · uses item.score ✓`. A mismatch shows the code and the field (`✗ M3014 gate uses item.scroe; judge.out has winner, score, reason`). Parts that need a workspace say so (`✗ M3103 gh.pr needs items with a workspace; none upstream`).
4. `enter` connects. `a` instead opens the palette and connects to a new node there. `n` toggles between a data edge and an ordering edge (`needs`). `esc` cancels.

`C` starts an ordering (`needs`) connection directly.

**What connecting writes.**

| Connection | YAML change |
|---|---|
| `a.out → b`, `b` has no `in` | `in: a` on `b` |
| `a.out → b`, `b` already has `in: x` | `in: [x, a]` (items from both) |
| `a.<port> → b` | `in: a.<port>` (or added to the list) |
| ordering `a ┄▷ b` | `needs: [a]` on `b` |
| Any, in a file that uses `edges:` | The edge is added to `edges:` instead, to keep the file's style |

**Edges, focused.** `tab` focuses the selected node's edges one at a time. With an edge focused:

| Key | Action |
|---|---|
| `x` | Remove it. For an inferred edge, the editor explains where it comes from (`from ${{ nodes.plan.item.plan }} in build.with.prompt`) and offers `gd` to the reference or `ga` to set an explicit `in:` instead. |
| `c` | Reroute: keep the source, choose a new target |
| `a` | Insert a node on it |
| `gi` | The items on it from the last dry run or the pinned sample |

`x` with no edge focused opens an edge picker for the selected node.

#### 18.7.8 Branches, loops and errors

| Part | Ports | In the editor |
|---|---|---|
| `if` | `true`, `false` | `cond` is an expression field with preview. Both ports are drawn; an unconnected one is `◌` (`M3030`). |
| `switch` | `case.<name>`…, `default` | `on` is the expression; `cases` is a list in the inspector: `+` adds a case (and a port), `r` renames it (updating edges), `J`/`K` reorder, `dd` removes. Each case is `name: expression` or `name: value`. |
| `assert` | `out`, plus `else` when `else: route` | `that` with preview; `else` cycles `fail · route · drop`. With `fail` the card's badge is `✗fails`, with `drop` it's `⌀drops`, so what happens to a failing item is visible without opening it. |
| `check` | `true`, `false` | The command and `ok_codes`. In `each` mode after a fanout it checks every candidate. |
| `merge` | `in` (many), `out` | `mode` cycles `all · any · zip · concat`. For `zip` and `concat` the input order matters, so the inputs are numbered on the canvas (`1▶ 2▶`) and `J`/`K` in the inspector reorder `in`. |
| `loop` | `body` (inside), `out`, `exhausted` | A container (§18.7.3). `until` (an expression, or `{ sh: … }` as a check), `max_iters` (required: `M3040`), and `loop.iter`, `loop.max` and `loop.prev` available in the body's expressions (§8). `exhausted` gets the last item when `max_iters` ran out: route it to a handoff. |
| `agent.loop` | `out`, `exhausted` | One card with `↻ ×3 · until make check`; no container, because the loop is inside the part. |
| `agent.fanout` → `agent.judge` | `out` | Fanout is drawn stacked (§18.7.3); its output is one item per candidate. The judge's inspector shows the rubric and the candidate fields it compares. |
| `handoff` | none | Its `to` part and that part's settings; the card is closed on the right. Anything connected after it is `M3031`. |
| any node, `on_error: route` | adds `error` | Error items (`class`, `message`, `node`, `exit_code`, `stderr_tail`, §15.8) leave from `error`. `on_error: continue` shows `⇢` on the card. |
| any node, `when` | | `?` on the card; the expression and its preview in the inspector. |
| any node, `discard` | | Ports listed in `discard` are drawn `⌀`: items there are dropped on purpose, and `M3030` stays quiet. |

A loop, with the what-if for `retry = exhausted` showing what gets skipped:

```
 ≋ nightly-sweep.mill.yaml                                                                ≈ $0.51–$1.53 of $6.00  ·  ✓ lint
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ↻ retry · loop ─────────────────┐
│                                                                                       ││ WITH                            │
│  ·   ·   ·   ·   ·   ┌┄ ↻ retry · until test.passed · ≤3 ┄┄┄┄┄┄┄┐·   ·   ·   ·   ·   ·││ body*       patch, test         │
│ ┌ ? found ──────┐    ┆ ┌ ◆ patch ──────┐      ┌ ⊨ test ───────┐ ┆    ┌ ⇡ pr ───────–─┐││ until*      ${{ nodes.test.item…│
│▶│if         true├─────▶│agent          ├─────▶│check      true├─┼───▶│gh.pr          │││   = true                        │
│ │size(ite… false│⌀   ┆ │sonnet         │      │make che… false│ ┆    │draft · main   │││ max_iters*  3                   │
│ └───────────────┘·   ┆ └───────────────┘ ·   ·└───────────────┘ ┆·   └───────────────┘││ NODE                            │
│                      └┄ ↺ next try ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄exhausted┄┬┄┄┘                     ││ id          retry               │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   │   ·   ·   ·   ·   ·   ·││ uses        loop                │
│                                                              │       ┌ ↪ file-issue ─╖││ in          found.true          │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   └──────▶│handoff        ║││ mode        batch default       │
│                                                                      │→ gh.issue     ║││ when        –                   │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   └───────────────╜││ needs       –                   │
│                                                                                       ││ runner      local (setting)     │
   ⋮
 NORMAL  nightly-sweep · 8 nodes · 7 edges   what if retry = exhausted: 1 skipped (pr)                     retry  cards 4,0
 a add · c connect · enter inspect · R dry run · ? more
```

**What-if.** `zs` on a branching node (`if`, `switch`, `assert`, `check`, a loop, or a node with a `when`) cycles through its outcomes: `found = true`, `found = false`, … For each one, every node that would be `skipped` is dimmed with `–`, using the real propagation rule (§15.2: a node whose inputs are all skipped is skipped, unless it's a `merge` with `any`). The status line says `what if found = false: 4 skipped (fix, test, pr, file-issue)`. `esc` ends it. It changes nothing in the file.

#### 18.7.9 Lint

Lint runs on every change, as you type.

- **Two passes.** The client checks what it can on its own, at once: structure, the graph, references, expressions, part schemas, policy and budget. The daemon's `Lint` intent (§16.3) is sent with the buffer 300 ms after the last change for what needs the machine: secrets set, plugins installed, agents available, runners configured. Results merge into one list.
- **Where problems show.** A badge in the node's title (`✗` red, `!` amber); the card's border takes the same color; problem edges too. The header counts them. The inspector lists the selected node's problems with their fixes. `!` opens the problems panel:

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⊢ gate · assert ────────────────┐
│                                                                                       ││ WITH                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   · ││ that*       ${{ nodes.judge.ite…│
│ ┌ ⧉ fanout ─────┐      ┌ ⊨ check ────!─┐      ┌ ◭ judge ──────┐      ┌ ⊢ gate ─────✗─┐││   ✗ M3014 no field "scroe" on … │
│▶│agent.fanout   ├─────▶│check each true├─────▶│agent.judge    ├─────▶│assert         ├││ else        ◂ route ▸           │
│ │×3 opus·gpt-5·…│┐     │make che… false│◌     │rubric.md      │      │judge.scr… else├││ NODE                            │
│ └───────────────┘│  ·  └───────────────┘·   · └───────────────┘ ·   ·└───────────────┘││ id          gate                │
   ⋮
│                                                                                       ││ output      –                   │
└───────────────────────────────────────────────────────────────────────────────────────┘│ LINT                            │
┌ problems · 4 ─────────────────────────────────────────────────────────────────────────┐│ ✗ M3014 with.that uses judge.s… │
│ ✗ M3014  gate       with.that uses judge.scroe; judge.ou…                ga use score ││   ga use score (+1)             │
│ ✗ M3204  pr         gh.pr is destructive and isn't in po…     ga allow gh.pr (policy) ││ TEST                            │
│ ✗ M3201  pipeline   this pipeline runs agents but has no…   ga add the medium budget… ││ in   1 item · no sample         │
│ ! M3030  check      check.false isn't connected · items …   ga add a handoff on false ││ out  –                          │
│                                                                                       ││ est  ≈ 2s                       │
│                                                                                       ││                                 │
│                                                                                       ││                                 │
│                                                                                       ││                                 │
└───────────────────────────────────────────────────────────────────────────────────────┘└─────────────────────────────────┘
 NORMAL PROBLEMS  best-of-3 · 7 nodes · 6 edges   3 lint errors · ]e walks through them, ga fixes the one s gate  cards 5,0
 j/k move · enter go · ga fix · gA fix all safe · esc close
```

| Key | Action |
|---|---|
| `]e` / `[e` | Next / previous problem (selects the node and its field) |
| `!` | Problems panel (`:copen` works too) |
| `ga` | Apply the fix for the problem at the selection. With more than one fix, a picker opens. |
| `gA` | Apply every **safe** fix in the pipeline |
| `gx` | Explain the code (`mill explain`) |

**Safe and policy fixes.** A fix is safe when it only completes what you already wrote (a typo with one close match, a missing `max_iters`, a budget preset, declaring a secret you reference). A fix that **widens what the pipeline may do** (adding a part to `policy.allow_destructive`, allowing a runner) is never part of `gA` or `mill lint --fix`: you apply it one at a time with `ga`, and the status line says what it allows.

**Saving with problems.** `:w` refuses while there are errors (`M2203 3 lint errors, not saved · :w! saves anyway`). `:w!` saves; a pipeline saved with lint errors keeps its triggers disarmed and can't be run until lint passes, and the board shows `lint errors · e to fix` as its text. Warnings never block.

**Checks.**

| Code | Severity | Check | Quick fix |
|---|---|---|---|
| `M3001` | error | Unknown key in a node or its `with` | Rename to the closest key · remove it |
| `M3002` | error | Required setting missing | Add it with a placeholder and focus it |
| `M3003` | error | Placeholder `‹…›` not filled | Focus the field |
| `M3010` | error | Unknown part or version | Closest part id · `mill plugin add` for a known plugin |
| `M3012` | error | Unknown node in an expression | Closest node id |
| `M3013` | error | Unknown input `inputs.<name>` | Declare the input |
| `M3014` | error · warning | Unknown field on a node's output: error when the producer declares `output`, warning when it doesn't | Closest field · add the field to the producer's `output` |
| `M3020` | error | Cycle outside a loop | Remove the closing edge · wrap the cycle in a loop |
| `M3021` | error | Unreachable: only fed by a handoff, a `fail`, or a port that never emits | Reconnect to the nearest upstream |
| `M3022` | warning | Orphan: no inputs and nothing references it, so it runs at the start of every run | Connect after the nearest node · delete |
| `M3023` | warning | A data part's result is used by nothing and isn't in `outputs` | Add to `outputs` |
| `M3030` | warning | Branch port not connected (`false`, `else`, `case.*`, `default`, `exhausted`, `error`): items there are dropped | Add a handoff · connect to an existing node · add the port to `discard` |
| `M3031` | error | Edge out of a `handoff` or `fail` | Move the edge to the handoff's upstream |
| `M3032` | warning | `merge` with fewer than two inputs | Replace with a direct edge |
| `M3033` | hint | `needs` on a node that's already upstream | Remove it |
| `M3040` | error | `loop` without `max_iters` | Add `max_iters: 3` |
| `M3041` | error | Edge crosses a loop's border other than through `in`, `out` or `exhausted` | Route it through the loop's port |
| `M3042` | warning | `until` doesn't depend on anything in the loop's body, so it never changes | Focus `until` |
| `M3101` | error | Expression type mismatch (§29.4) | |
| `M3102` | error | The target needs fields the source doesn't produce (for example `agent.judge` gets items with no candidates) | Show both schemas |
| `M3103` | error | A part that needs a workspace (`gh.pr`, `git.commit`, `git.push`, `git.diff`, a code-changing agent) gets items with none upstream | Insert `git.worktree` before the first node that needs it |
| `M3110` | warning | Fields of an agent's result are used, but the agent declares no `output` | Declare `output` from the fields used |
| `M3111` | error | Agent prompt empty | Focus `prompt` |
| `M3112` | warning | Agent in `each` mode with no `concurrency` and no `budget.max_agents` | Set `concurrency: 2` |
| `M3201` | error | The pipeline runs agents but has no `budget` | Add the medium budget (§18.8) |
| `M3202` | warning | `budget.max_agents` is lower than a fanout's width, so candidates run one after another | Raise `max_agents` |
| `M3203` | error | Runner not allowed by `policy.allowed_runners` | Use an allowed runner · allow it (policy fix) |
| `M3204` | error | Destructive part not in `policy.allow_destructive`, including a handoff's `to` part | Allow it (policy fix) |
| `M3205` | hint | `allow_destructive` lists a part no node uses | Remove it |
| `M3206` | error | The part waits for a person (hands-off, §2) | Replace with a `handoff` |
| `M3207` | warning | A code-changing node runs with `workspace: none` (writes the main checkout; needs a confirmation on first run, §22) | `workspace: git` |
| `M3208` | warning | The estimated worst case per run is over `budget.max_cost_usd` or `max_wall`, so the run will be cancelled before it finishes | Raise the budget · lower `max_iters` or the fanout |
| `M3301` | warning | A trigger binds an event field its event doesn't have | Closest field |
| `M3302` | error | A required input has no default and no trigger fills it (a cron run couldn't start) | Add a default |
| `M9201` | error | Secret used but not declared | Declare it |
| `M9202` | warning | Secret declared but not set on this machine | `help: mill secrets set <NAME>` |
| `M9203` | hint | Secret declared but never used | Remove it |

`mill lint` runs the same checks with the same codes; `mill lint --fix` applies the safe fixes and `mill lint --fix --code M3204 --node pr` applies one fix explicitly (§17.1).

#### 18.7.10 Inspector

The inspector is a form generated from the part's schema (§9.3: the manifest's config schema), followed by the node's own settings, its problems, and its test data:

The selected node is `gate`, with its expression being edited and completion open:

```
 ≋ best-of-3.mill.yaml                                                             ≈ $1.94 per run · no budget  ·  ✗ 3  ! 1
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⊢ gate · assert ────────────────┐
│                                                                                       ││ WITH                            │
│ ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·  ┌ ${{ ─────────────────────────────┐│ that*       }} && ${{ nodes.ju  │
│ ┌ ⧉ fanout ─────┐      ┌ ⊨ check ────!─┐      ┌ ◭ j│ judge              agent.judge   ││   ✗ M3014 no field "scroe" on … │
│▶│agent.fanout   ├─────▶│check each true├─────▶│agen└──────────────────────────────────┘│ else        ◂ route ▸           │
│ │×3 opus·gpt-5·…│┐     │make che… false│◌     │rubric.md      │      │judge.scr… else├││ NODE                            │
│ └───────────────┘│  ·  └───────────────┘·   · └───────────────┘ ·   ·└───────────────┘││ id          gate                │
│  └───────────────┘                                                                    ││ uses        assert              │
   ⋮
 INSERT INSPECTOR  best-of-3 · 7 nodes · 6 edges   connect cancelled                                        gate  cards 5,0
 type edit · tab complete · enter done · esc keep · ctrl+c discard
```

**Sections.** `with` (the part's settings, required ones marked `*`), `node` (the node keys from §7.3), `lint` (this node's problems and fixes), `test` (pinned sample, last dry-run output, estimate). `zc`/`zo` fold a section.

**Fields.**

| Type | Shown as | Edit |
|---|---|---|
| text | the value, `…` when cut | `enter` edits in place; `E` opens `$EDITOR` |
| expression | `${{ … }}` with the delimiters dim, and a preview line under it | `enter` with completion (below) |
| prompt (multi-line) | the first line and `+12 lines · ≈ 410 tk` | `enter` opens the prompt editor; `E` opens `$EDITOR`; `gf` opens the `@file` it points to |
| choice | `◂ value ▸` | `h`/`l` |
| on/off | `● on` / `○ off` | `space` |
| number, duration | `3`, `10m` | `h`/`l` step, `enter` types |
| list | `a, b, c` | `enter` edits as text; `+` adds an item |
| part reference (`handoff.to`, `agent`) | `gh.issue`, `claude/opus` | `enter` opens a picker of parts or agents and models from the registry (§11.1) |
| dynamic choice | the value | Filled by the part through `PartCall` (§9.3): Slack channels, Linear teams |
| inherited | dim, with its source: `↻ 2 (defaults)` | Editing it writes the key on the node; `u` removes it again and the default shows |

`u` on a field resets it (removes the key, so the part's or pipeline's default applies). `gd` on an expression goes to the node it names.

**Node settings.** `uses` (`enter` opens the palette to replace the part, keeping settings that still apply), `id` (`r`), `in` (read-only: the edges; `gd` goes upstream), `mode`, `concurrency`, `when`, `needs`, `runner` (choices from config, filtered by policy), `workspace`, `retry` (`max`, `on`), `timeout`, `cache`, `on_error`, `discard`, `output`. For agent nodes also `agent` (id and model, from the registry), `effort`, and the prompt.

**Expressions.** Typing `${{` opens completion, anchored under the cursor. It knows the scope:

| Prefix | Offers |
|---|---|
| (start) | `inputs`, `nodes`, `event`, `item`, `items`, `run`, `env`, `secrets`, `loop` (in a loop body), `now`, and the helper functions (§8) |
| `inputs.` | Declared inputs, with type and default |
| `nodes.` | Upstream nodes first. A node that isn't upstream is marked `+ edge`: choosing it adds an inferred edge, and one that's downstream is refused (`M3020`). |
| `nodes.<id>.` | `item`, `items`, `state`, `exit_code`, `files`, `ws`, `diff` |
| `nodes.<id>.item.` | The fields of that node's `output` schema, with types |
| `event.` | The fields of the triggers' event schemas (from the trigger part manifests) |
| `secrets.` | Declared secrets only (opaque handles, §21) |

**Preview.** Under every expression field, a line shows its value against the node's input: the pinned sample if there is one, otherwise the last dry run's items. `= "medium"`, `= true`, `= 3 items`. An error shows its code in place (`✗ M3014 no field "scroe" on judge.out · did you mean score`). Without any data the line says `no sample · m pins one after a dry run`.

**Prompt editor.** `enter` on a prompt opens a multi-line editor over the canvas with line numbers, completion inside `${{ }}`, and a running token estimate. Typing `${{` closes it for you (`${{ ▌ }}`). `esc` keeps the text and closes; `ctrl+e` hands it to `$EDITOR` and reads it back; `ctrl+c` throws the edit away. `:prompt file` moves an inline prompt to `prompts/<pipeline>-<node>.md` and writes `@prompts/…` in its place; `:prompt inline` does the opposite.

**Output schema.** `enter` on `output` opens a small table: field, type (`string`, `number`, `bool`, `[string]`, `object`, `enum`), required. `+` adds, `dd` removes, `h`/`l` change the type, `I` infers the fields from the pinned sample. For agents, this is the schema mill asks the agent to write (§11.3), so lint and completion downstream know the fields.

#### 18.7.11 Pipeline panel

`gp` shows the pipeline's own settings in the right pane (`esc` returns to the inspector):

| Section | Edits |
|---|---|
| about | `description`, `tags` |
| triggers | One row per entry under `on:`; `enter` selects its card on the canvas; `gt` adds one; `on.concurrency` and `dedupe` |
| inputs | One row per input: name, type, default, `from` (a trigger binding); `+` adds, `dd` removes (lint shows where it was used) |
| outputs | `name: expression` with completion and preview; `+` adds |
| secrets | Declared names with `set` / `missing` on this machine (`M9202`); `+` adds |
| budget | A preset (`small`, `medium`, `large`, as in the wizard §18.8) or each limit: `max_tokens`, `max_cost_usd`, `max_wall`, `max_nodes`, `max_agents`. The estimate (§18.7.12) is shown next to it. |
| policy | `allow_destructive` as a checklist of the destructive parts this pipeline uses (`● gh.pr`), plus any listed but unused; `allowed_runners`; network rules |
| defaults | `runner`, `retry`, `timeout`, `workspace`, `agent` (id, model, effort): what every node inherits |
| env | `KEY: value` rows |

#### 18.7.12 Testing while you build

| Key | Action |
|---|---|
| `R` | **Dry run** the buffer (saved or not) with `--dry-run`: destructive parts report what they would have done (§17.10). Node states paint on the graph as they change, edges count their items, and the header shows progress. The run is a normal run in the store (`best-of-3#3 · dry run`). Editing during a dry run is allowed; the painted states are then marked `stale`. `R` refuses while there are lint errors and says which. |
| `.` | **Run this node alone** against its input: the pinned sample, else the last dry run's recorded input (`mill node run`, the same thing as `mill replay` for a node). Agent nodes ask first, with the estimate: `run judge alone? ≈ 60K tk · $0.30 · y/n`. The output appears in the items panel. |
| `gi` | **Items panel** for the selected node (input and output) or the focused edge: one item per row as JSON; `enter` expands one; `y` copies it. |
| `m` | **Pin** the items on the selected node's input as its sample. Samples live in `.mill/samples/<pipeline>/<node>.jsonl` (the trigger's event in `_event.json`), are plain files you can edit and commit, and are what previews and `.` use. `:sample edit` opens the file; `m` again unpins. |
| `ze` | Show estimates on the cards |

A dry run in progress, after `gA` and `ga` cleaned up lint (states paint the borders: blue working, green done, grey skipped):

```
 ≋ best-of-3.mill.yaml [+]                                                                          ⠋ dry run #4 · 2/7 · 4s
┌ cards ────────────────────────────────────────────────────────────────────────────────┐┌ ⇡ pr · gh.pr ───────────────────┐
│                                                                                       ││ WITH                            │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·││ title*      feat: ${{ inputs.ta…│
│ ┌ ⊨ check ────⠋─┐      ┌ ◭ judge ──────┐      ┌ ⊢ gate ───────┐      ┌ ⇡ pr ─────────┐││   = "feat: add retry with back… │
│▶│check each true├─────▶│agent.judge    ├─────▶│assert         ├─────▶│gh.pr          │││ body        @prompts/pr.md      │
│ │make che… false│⌀     │rubric.md      │      │judge.sco… else├─┐    │draft · main   │││ draft       ● on                │
│ └───────────────┘·   · └───────────────┘ ·   ·└───────────────┘ │·   └───────────────┘││ base        main default        │
│                                                                 │                     ││ NODE                            │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·  │·   ·   ·   ·   ·   ·││ id          pr                  │
│                                                                 │    ┌ ↪ no-winner ──╖││ uses        gh.pr  (mill/github)│
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·  └───▶│handoff        ║││ in          gate                │
│                                                                      │→ gh.issue     ║││ mode        batch default       │
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   └───────────────╜││ when        –                   │
│                                                                                       ││ needs       –                   │
   ⋮
 NORMAL  best-of-3 · 7 nodes · 6 edges   dry run #4 of the unsaved buffer · destructive steps only repor…     pr  cards 6,0
 a add · c connect · enter inspect · R dry run · ? more
```

**Estimates.** Each node's cost and time come from its own history when it has some (p50 of its last runs), otherwise from the part (agents: the model's `price` in the registry × a typical token count for the part; other parts: seconds). Fanouts multiply by their width, `each` by the sample's item count, loops give a range from one try to `max_iters`. The header shows the range per run against the budget: `≈ $2.40–$6.10 of $15`; the worst case over the budget is `M3208`. The pipeline panel and `mill estimate <pipeline>` show the breakdown. Estimates are labeled `≈` everywhere and never block anything.

#### 18.7.13 Safety net

- **Undo and redo:** `u` and `ctrl+r`, unlimited for the session, one step per action, each step named.
- **Diff:** `D` shows the buffer against the file on disk as a unified diff (green added, red removed), with the layout sidecar's changes summarized at the end (`layout: 2 nodes moved`).
- **YAML pane:** `Y` shows the YAML in the right pane, scrolled to the selected node with its block highlighted. Selecting another node scrolls it. `i` edits the YAML in place; on `esc` it's parsed, and the graph, lint and inspector update. A parse error (`M2101`, with line and column) keeps you in the pane and the graph keeps its last good state.
- **`$EDITOR`:** `E` opens the file in `$EDITOR` at the selected node (or field). When the editor exits, mill reloads the file and keeps the undo history, so a whole `$EDITOR` session is one undo step.
- **Rename** (`r`) rewrites every reference (§18.7.6), and `gr` shows them beforehand.
- **Changed underneath you.** If the file changes on disk while the editor is open (git checkout, another editor, `mill node` from a script), a line under the header says `M2201 best-of-3.mill.yaml changed on disk · :e reload · D diff · :w! overwrite`. Saving goes through `SavePipeline` with the base hash (§16.3), so nothing is overwritten by accident.
- **Unsaved buffers survive.** The buffer is written to `~/.local/state/mill/swap/<pipeline>.yaml` after each change. If the TUI or the terminal dies, the next `e` on that pipeline offers `recover · discard · diff`.
- **Lua-defined pipelines** (§19.7) open read-only: everything can be inspected, linted, dry-run and diffed; `E` opens the Lua source.

#### 18.7.14 Keys

**Canvas (`NORMAL`)**

| Key | Action |
|---|---|
| `h j k l`, `w b`, `gg G`, `/`, `tab`, `gd`, `gr`, `ctrl+o` | Move and jump (§18.7.5) |
| `enter` | Inspector for the selected node (or the items of the focused edge) |
| `space` | Action menu: every action that applies to the selection, with its key |
| `a` / `i` / `o` | Add after / before / as a new branch (§18.7.6) |
| `gt` | Add a trigger |
| `yy` / `p` / `P` | Yank / paste connected / paste unconnected |
| `dd` / `r` | Delete / rename |
| `c` / `C` / `x` | Connect / connect ordering-only / disconnect (§18.7.7) |
| `v` / `V` | Visual mode / select everything downstream |
| `H J K L` / `g=` | Move a slot and pin / unpin |
| `gl` / `gP` / `gU` | Wrap in a loop / extract to a sub-pipeline / unwrap or inline |
| `]e` `[e` / `!` / `ga` / `gA` / `gx` | Lint (§18.7.9) |
| `zs` / `zo` `zc` / `ze` | What-if / open or fold a container / estimates |
| `R` / `.` / `gi` / `m` | Dry run / run this node / items / pin a sample |
| `gp` / `Y` / `D` / `E` | Pipeline panel / YAML pane / diff / `$EDITOR` |
| `+ - =` / `zz` / `zh zj zk zl` / `M` / `O` | Zoom / center / pan / minimap / outline |
| `u` / `ctrl+r` | Undo / redo |
| `:` | Commands: `:w`, `:w!`, `:q`, `:q!`, `:wq`, `:e!` (revert), `:lint`, `:copen`, `:wrap loop`, `:extract <name>`, `:layout reset`, `:pattern save <name>`, `:prompt file|inline`, `:sample edit` |
| `?` | Editor help (every key above) |
| `esc` | Leave the editor (asks to save if changed) |

**Inspector**

| Key | Action |
|---|---|
| `j` / `k` | Next / previous field |
| `enter`, `i` | Edit the field (`INSERT`) |
| `h` / `l`, `space` | Change a choice, step a number, toggle |
| `+` / `dd` / `J` `K` | Add / remove / reorder list items and switch cases |
| `u` | Reset to the default |
| `gd` | Go to the node an expression names |
| `E` | Edit this field in `$EDITOR` |
| `esc` | Back to the canvas |

**`INSERT`** (a field, the palette, the prompt editor): `tab` accepts a completion, `ctrl+n`/`ctrl+p` move in the list, `enter` finishes a single-line field, `esc` keeps the text and leaves (in a field, `ctrl+c` throws the edit away).

**Help bars.** Canvas: `a add · c connect · enter inspect · R dry run · ? more`, with `ga fix` in place of `R dry run` while the selected node has a fixable problem. Edge focused: `a insert · c reroute · x remove · gi items · esc back`. `CONNECT`: `hjkl target · enter connect · a new node · n needs · esc cancel`. `VISUAL`: `hjkl move · space pick · gl loop · yy yank · esc done`. Inspector: `j/k field · enter edit · h/l change · gd go to · esc canvas`. Panel: `j/k move · enter go · ga fix · gA fix all safe · esc close`. Palette: `type search · tab category · enter insert · esc close`. Field: `type edit · tab complete · enter done · esc keep · ctrl+c discard`.

#### 18.7.15 Mouse

- Click a node to select it; double-click opens the inspector. Click an edge to focus it. Shift-click adds to the selection.
- Drag a node to move it to another slot (a pin, as with `H J K L`). Drag from an output port to a node to connect.
- Drag the empty canvas, or scroll, to pan.
- Click a field in the inspector to select it, again to edit. Click a problem to jump to it.
- Overlays swallow the mouse (§18.1 rule 10).

#### 18.7.16 CLI parity

| Editor | CLI |
|---|---|
| `a`, `i`, `o`, palette | `mill node add <p> <id> --uses <part> [--after n[.port] \| --before n \| --branch n.port] [--with k=v]...` |
| `gt` | `mill trigger add <p> <kind> [--with k=v]...` · `mill trigger rm <p> <kind>` |
| inspector edits | `mill node set <p>/<n> <key>=<value>...` · `mill node unset <p>/<n> <key>...` |
| pipeline panel | `mill pipeline set <p> <key>=<value>...` (`budget.max_cost_usd=15`, `policy.allow_destructive+=gh.pr`) |
| `dd` | `mill node rm <p>/<n> [--heal]` |
| `r` | `mill node mv <p>/<old> <new>` |
| `yy` `p` | `mill node cp <p>/<n> [<new>] [--after n[.port]]` |
| `c`, `C`, `x`, reroute | `mill edge add <p> <from>[.port] <to> [--needs]` · `mill edge rm <p> <from>[.port] <to>` |
| patterns | `mill pattern ls` · `mill pattern add <p> <pattern> --after <n>[.port]` · `mill pattern save <p> <n>... --as <name>` |
| `gl`, `gP`, `gU` | `mill wrap <p> loop <n>... [--until expr] [--max n]` · `mill extract <p> <n>... --as <name>` · `mill unwrap <p>/<n>` |
| lint, `ga`, `gA` | `mill lint <p> [--json] [--fix] [--code M… --node n]` |
| `gr` | `mill refs <p>/<n>` |
| `R`, `.`, `m` | `mill run <p> --dry-run` · `mill node run <p>/<n> [--sample \| --from-run <run>]` · `mill node pin <p>/<n> --from-run <run> \| --file items.jsonl` |
| estimates | `mill estimate <p> [--json]` |
| `H J K L`, `g=` | `mill layout <p> pin <n> <col>,<row>` · `mill layout <p> reset [<n>...]` |
| `O`, canvas | `mill show <p> --outline` · `mill render <p>` |
| `Y`, `E` | `mill show <p> --yaml` · `mill edit <p>` |

Every command edits through the daemon with the file's hash, keeps comments and formatting, lints, and refuses to save errors without `--force` (§17.8).

**Implementation.** `internal/pipeline/edit` holds every operation (add, insert, connect, rename, wrap, extract, the quick fixes) as functions over the `yaml.v3` node tree, tested once and used by the CLI and the TUI. `internal/pipeline/layout` computes slots and routes edges into a cell grid shared by the editor, the run popup and `mill render`. `internal/surface/tui/editor` is the Bubble Tea model: canvas, right pane, panels, palette and the key map from `keymap.bind` (§18.13). The browser prototype in `mockups/v2/index.html` follows these keys; the screens in this section are its output.

### 18.8 Wizards

#### New pipeline (`n`)

`n` on the board, `:new`, or `mill new` (the same steps as terminal prompts) opens a square overlay with five steps and a breadcrumb at the top: `✓ describe › 2 trigger › 3 agent › 4 limits › 5 review`.

| Step | You choose | Notes |
|---|---|---|
| **1 describe** | A name, then one of: a sentence ("every night find flaky tests, fix them and open a draft PR"), a template, or import | A sentence goes to a headless agent, which proposes the trigger, agent and steps. Templates come from built-ins and plugins. Import takes an n8n workflow or a `.millpkg`. Errors show inline (`M1210` bad name, `M2205` name taken). |
| **2 trigger** | Manual, schedule, webhook, git, file change, or after another pipeline | Manual also covers "run by a script or by an agent over MCP". |
| **3 agent** | Agent and model from the registry, with effort | Shows installed and signed in. A missing CLI is `M6101`. "None" makes a pipeline without agent steps. |
| **4 limits** | Budget (small, medium, large), max agents, which destructive steps are allowed, what happens on failure (nothing, a GitHub issue, a rex session), dry run first | The run never waits on you, so its limits are set here. |
| **5 review** | The proposed steps as a checklist (`space` toggles) beside the YAML and the lint result | `enter` creates and opens the editor; `r` creates and dry-runs. Nothing is written before this step. |

**Keys:** `j`/`k` move, `enter` next, `backspace` back, `space` toggle, `h`/`l` change a value, `esc` cancel (nothing saved). The status line shows `NEW` and the step.

**CLI parity:** `mill new <name> --from <template|describe:"…"> --trigger … --agent claude/sonnet --steps ws,work,check,pr --budget medium --allow gh.pr --on-failure gh.issue --dry-run`.

#### First-run setup (`mill setup`)

The first `mill` on a new machine runs setup right after the boot splash (§18.13), and `mill setup` runs it again any time. Each step shows what it found and what it will change, and asks once:

1. **Agents:** finds agent CLIs on `PATH` (Claude Code, Codex, Gemini, Aider, OpenCode, Ollama), checks each is signed in, and writes their entries to `init.lua`.
2. **Sound:** plays a short preview of each soundset and asks which one, or off. Also asks about the startup voice.
3. **Shell:** offers completion and the `≋` prompt segment for your shell, showing the exact lines before adding them.
4. **Trusted folders:** asks where agents may skip folder-trust prompts. Refuses `/`, your home folder, relative paths and `..`.
5. **Starter plugins:** offers `mill/github` and `mill/slack`, and `mill/rex` if rex is installed.
6. **Example:** writes `.mill/pipelines/hello.mill.yaml`, a two-step pipeline you can dry-run at once.
7. **Daemon:** starts `milld`.

`--yes` accepts every default, `--preview` prints what it would do and changes nothing, and `--force` overwrites an existing `init.lua` (after backing it up).


### 18.9 Plugins

`:plugins` opens mill's app manager over the board. `tab` cycles its views: **installed · updates · available · mcp · agents**. One row per item: state, name, version (`1.2.0 → 1.3.0` when an update waits), what it provides, health, and how many pipelines use it. The line under the list shows the selected item's permissions, secrets and users.

**Keys**

| Key | Action |
|---|---|
| `i` | Install the selected plugin (Available view) |
| `u` / `U` | Update the selected plugin / all of them |
| `x` | Remove (refused while pipelines use it; the error lists them) |
| `enter` | Open the full detail page |
| `c` | Run its health check now |
| `t` | Agents view: run `mill agent test` on the selected agent |
| `gd` | Open its manifest, or the line in `init.lua` that declares it |
| `/` | Filter installed plugins, or search the registry in Available |

**Rules**
- **Permissions are explicit.** Installing shows everything the plugin asks for. An update that asks for *new* permissions shows only what changed and needs a confirm; an update that asks for nothing new applies with one key.
- **Never mid-run.** Updates and removals apply between runs. A run always finishes with the plugin versions it started with (they're recorded on the run).
- **Health is checked, not assumed.** Each plugin's parts answer a ping, MCP servers are checked for running and signed in, agent CLIs for installed and signed in, secrets for resolving. Problems show in the health column with the same error codes as everywhere else, and `gx` explains them.
- **Hands-off updates (optional).** `plugins.auto_update` can be `off` (default), `patch` or `minor`. Automatic updates still never add permissions; those wait for you on this page.
- **Dev plugins** (`mill plugin dev <dir>`) show as `dev` with a hot-reload marker and are never auto-updated.

**CLI parity:** `mill plugin ls|info|add|update|rm|search`, `mill mcp ls`, `mill agent ls|test`.


### 18.10 Settings

`S` (or `:settings`) opens the settings page over the board. It's a view of the settings registry (§19.6), so every setting appears here automatically, with its type, default, help text and validation. Nothing is hidden in a file you have to find.

**Layout**
- **Left:** sections, navigated like the board (`j`/`k`, relative line numbers): General, Runs, Agents, Plugins, Appearance, Sound, Gateway, Advanced.
- **Right:** one row per setting: name, current value, and a short note of the choices. The selected row's help text, default and source sit on the bottom line.
- **Statusline:** the `SETTINGS` mode, the setting id under the cursor (`sound.set`), and the file changes are written to.

**Editing**

| Key | Action |
|---|---|
| `h` / `l` | Previous / next value for choices (`◂ millrace ▸`), or −/+ for numbers and sliders |
| `enter` / `space` | Toggle on/off, or open a text value for editing |
| `i` | Edit a text value in place; `esc` cancels, `enter` saves |
| `p` | Preview: plays the sound, shows the color, or renders a sample row |
| `u` | Reset to the default |
| `gd` | Go to where the value is set: opens `init.lua` or `.mill/mill.lua` at that line |
| `/` | Search every setting by name or help text |

**Rules**
- Changes apply immediately and are written to the managed `settings.lua` (§19.1). There's no save button and no restart.
- A value set in `init.lua`, a project file, an environment variable or a flag is shown with its source (`init.lua:9`) and can't be changed here, because the higher layer would override it anyway. `gd` takes you to it.
- Invalid values are refused with the same error format as everywhere else (§29), and the old value stays.
- Every setting on the page is also reachable as `mill config get|set <id>`.


### 18.11 Sound

mill makes small sounds as you use it and as runs progress, so you can follow what's happening without looking. Sound is on by default in the TUI and never plays anywhere else.

**Soundsets.** A soundset is a named collection of short sounds, one per event. Built-in sets (exact recipes in Appendix A):

| Set | Character |
|---|---|
| `millrace` (default) | Water and wood: soft clicks, a sluice opening, drops, a turning wheel |
| `foundry` | Industrial: switches, conveyor clunks, a press, a furnace roar for failures |
| `chiptune` | 8-bit blips and arpeggios |
| `bell` | Only the terminal bell, for failures and finished runs |
| `off` | Silence |

**Events**

| Group | Event | When it plays |
|---|---|---|
| Startup | `startup` | The spoken "mill" when the boot splash hands off to the board (below) |
| Interface | `boot_ok`, `boot_warn`, `boot_fail` | A boot splash row appears with that status (§18.13). SKIP rows are silent |
| Interface | `tick` | Moving the cursor (`j`/`k`, `h`/`l`) |
| | `open`, `back` | Opening a screen, pipeline or run; going back |
| | `toggle` | Switching a setting, folding, changing a filter |
| | `error` | Any error shown in the statusline |
| Editing | `place` | Adding a node on the canvas |
| | `connect`, `disconnect` | Making or removing an edge |
| | `save` | A pipeline was saved |
| | `lint_ok`, `lint_fail` | Lint passed / found problems |
| Runs | `run_start` | A run started (by you or a trigger) |
| | `node_done` | A node finished |
| | `retry` | A node is retrying |
| | `run_done` | A run finished successfully |
| | `run_failed` | A run failed |

**Startup voice**

Like rex, mill greets you by name. When the boot splash hands off to the board, a voice says **"mill"** once.

- **A different voice from rex.** The two tools should be told apart by ear. Direction for the recording: calmer and lower than rex's, with a slightly mechanical or "machine floor" texture to match the mill theme. The final voice is still to be made; until then the slot holds a placeholder.
- **The asset.** One short clip, `sounds/voice/mill.ogg`, embedded in the binary. Target under 800 ms, loudness-normalized to match the soundsets, with no music bed.
- **When it plays.** Once per TUI launch, at the hand-off from the boot splash (§18.13), never during its cascade. Not after a failed boot, not on daemon restarts, and never from the CLI.
- **Rules.** It follows the same rules as every other sound: off with `sound.voice = false`, silent during quiet hours, skipped with no audio device. Custom soundsets can replace it by mapping their own `startup` file, or leave it out to stay silent.

**Behavior**
- **Short and quiet.** Interface sounds are under 60 ms and mixed low. Run sounds can be longer (up to about half a second for `run_done` and `run_failed`).
- **No machine-gunning.** Holding `j` plays at most one `tick` every 40 ms. A pipeline chimes at most once every 1.5 s, and bursts of `node_done` from a wide fan-out are merged into one.
- **Low latency.** Sounds are embedded, decoded once at startup and played from memory, so they keep up with keystrokes.
- **Silent fallback.** With no audio device (SSH, containers, CI) mill plays nothing and never errors. The CLI, `--local` runs and anything not attached to a terminal never play sounds.
- **Your rules.** Groups can be turned on or off separately. `sound.when` can be `always`, `unfocused` (run sounds only when the terminal isn't focused, where the terminal reports focus) or `never` during `sound.quiet_hours`.
- **Custom soundsets.** A folder in `~/.config/mill/sounds/<name>/` with a `soundset.lua` that maps event names to `.wav` or `.ogg` files. Missing events fall back to the default set. Plugins can ship soundsets too (`provides.soundsets`).
- **From Lua.** `mill.sound.play("run_done")` in a hook or command, and `mill.setup { sound = { set = "foundry", volume = 0.6 } }` in config.


### 18.12 Screensaver, long runs, detach

**Screensaver.**

After 5 minutes (`screensaver.after_minutes`) with no key or mouse event, the screen gives way to the logo in the center with the live counts under it:

```
            ≋ mill

1 failed · 2 running · 14 ok today
```

- **Same size as rex's.** The logo is one line of normal text, the mark and the name (`≋ mill`), the same size and placement as rex's `∴ rex`. It isn't a large banner or block letters. Both tools' screensavers look like a matched pair.
- **Live counts.** Daemon events keep the counts current. `failed` shows in red while above zero; everything else stays dim. While the daemon is unreachable, the reconnect status shows under the counts.
- **Waking is safe.** Any key or mouse event (click, wheel) brings the screen back, and that key or click does nothing else. An `enter`, `R` or `ctrl+c` that wakes mill never opens, reruns or quits anything, and a half-typed `dd` from before is dropped.
- **Only when you're idle.** It only starts over the bare board. Popups, the editor, the wizard, settings, prompts and confirmations never time out, because there you're in the middle of something.
- **Burn-in safe.** Once a minute it moves a little (at most 6 columns and 2 rows from the center). With `reduce_motion` on it stays centered.
- **Costs nothing.** No CPU while idle: spinners stop and one timer fires at most once a minute. It plays no sound. The startup voice and run sounds are separate.
- **Keep awake (macOS).** With `screensaver.keep_awake` on (the default), mill runs `/usr/bin/caffeinate -d -w <mill pid>` while the screensaver shows, so the display doesn't sleep. It's stopped when the screensaver goes away or the TUI exits, and `-w` ends it if mill dies. On other systems the setting does nothing. If macOS turns the display off sooner than `screensaver.after_minutes`, mill doesn't turn it back on, so set the minutes below the system's display timeout.


**Long-run badge.** A small column before the age flags runs that are taking longer than the pipeline usually does (its p50 duration). The column is hidden while no row has a badge:

| Badge | Running for |
|---|---|
| (blank) | under 1.5× typical |
| `▴2×` dim | 1.5–3× typical |
| `▴4×` amber | 3–6× typical |
| `▴8×` red | 6× typical or more |
| `▴8×` red, inverse | past the run's `max_wall` (it's about to be cancelled) |

**Stats.** `:stats` opens one table: per pipeline, runs, success rate, p50 and p90 duration, and cost for today, 7 days and 30 days, with totals on the last line. `mill stats` prints the same.

**Detach without quitting.** `:bg` or `:detach` saves the selection, view and filter and exits the TUI. The daemon and every run continue. The next `mill` restores them once.


### 18.13 Boot splash

Every time `mill` opens the TUI from a terminal, it boots in front of you before the board appears. The splash is a clone of rex's bootloader, made for mill: the same status tokens, colors, timing, failure screen and sounds, with mill's own steps and words, so the two tools start up as a matched pair.

Only a bare `mill` shows it. Subcommands, pipes, `--json` and anything not attached to a terminal never do; they report problems as normal errors (§29.2).

```

  ≋ ミル · mill machine intelligence logistics layer
  起動中 booting...  1.12s

  [  OK  ] log.init            · ~/.local/state/mill/logs/tui.log
  [  OK  ] paths.ensure        · config + state dirs ok
  [  OK  ] tty.probe           · 124x34 · truecolor · en_US.UTF-8
  [  OK  ] config.load         · ~/.config/mill/init.lua + settings.lua
  [  OK  ] project.trust       · ~/code/app · trusted · .mill/mill.lua
  [ SKIP ] theme.apply         · default
  [  OK  ] audio.init          · soundset=millrace · vol=0.60 · voice=true
  [  OK  ] audio.load          · millrace · 起動準備
  [  OK  ] parts.load          · 部品 · 41 built-in · 23 from plugins
  [ WARN ] plugins.verify      · 8 plugins · lock ok · notion: auth expired (M6202)
  [  OK  ] agents.probe        · claude · codex · gemini · ollama
  [  OK  ] keymap.bind         · board + editor + wizard bindings ready
  [  OK  ] socket.resolve      · /run/user/501/mill/milld.sock
  [  OK  ] daemon              · already running · pid 48213
  [  OK  ] client.dial         · connected · 2ms
  [  OK  ] handshake           · 接続 · mill-tui · milld 0.5.0
  [  OK  ] subscribe           · 受信中 · event stream open
  [  OK  ] snapshot.parse      · 7 pipelines · 1 failed · 2 running · 4 ready
  [ WARN ] pipelines.lint      · 配管 · 6 clean · best-of-3: M3204
  [  OK  ] triggers.arm        · 水門 · 3 cron · 1 webhook · 1 git · next 03:00
  [  OK  ] state.restore       · selected=nightly-sweep · view=all
  [  OK  ] renderer.warm       · styles cached

                                                                            準備完了 ready · 1.31s
```

**Layout**

- **Header:** the mark and the name in katakana, `≋ ミル` (rex's is `∴ レックス`), then `· mill machine intelligence logistics layer` in dim text.
- **Status line:** `起動中 booting...` with the elapsed time, updating live. On a failure it changes to `起動失敗 boot failed` in red.
- **One row per step:** `[ STATUS ] <step>  · <what it found>`. The step name sits in a fixed 18-cell column (east-asian-width aware), with a 2-cell left margin. The splash is full screen, with no overlay and no board behind it.
- **Ready line:** `準備完了 ready · 1.31s`, right-aligned and green, once the last step is done.
- **Words.** The Japanese is shared with rex where the meaning is the same (起動中 booting, 準備完了 ready, 起動失敗 boot failed, 起動準備 getting ready, 接続 connected, 受信中 receiving). mill adds its own words for its own steps: 部品 *parts*, 配管 *piping* and 水門 *sluice gate*, for the triggers that let runs through. When `tty.probe` falls back to ASCII, the Japanese is dropped and `≋` becomes `~`.

**Statuses**

| Token | Color | Meaning | Boot continues | Sound |
|---|---|---|---|---|
| `[  OK  ]` | green, bold | The step worked | yes | `boot_ok` |
| `[ WARN ]` | amber | It worked with a problem mill can live with: the board opens, and the warning stays in `:messages` | yes | `boot_warn` |
| `[ SKIP ]` | grey | Nothing to do: first run, sound muted, no project here | yes | none |
| `[ FAIL ]` | red, bold | mill can't open the board. The splash freezes (below) | no | `boot_fail` |
| `[  ..  ]` | blue | The step is still working after 250 ms, usually `daemon` starting `milld`. The row is replaced in place when it finishes | waits | none |

The bracket characters are always grey; only the word inside is colored.

**Steps.** They run in this order, because later steps use what earlier ones found. Every rex step has a mill counterpart (`config.load` is rex's `settings.load`, `parts.load` its `registry.load`). `project.trust`, `plugins.verify`, `agents.probe`, `pipelines.lint` and `triggers.arm` are mill's own.

| # | Step | Checks | OK shows | Other outcomes |
|---|---|---|---|---|
| 1 | `log.init` | Opens the TUI log | the log path | WARN `logging to stderr` |
| 2 | `paths.ensure` | Config and state folders exist and are private (0700) (§20) | `config + state dirs ok` | FAIL can't create or tighten them |
| 3 | `tty.probe` | Size, color depth and locale | `124x34 · truecolor · en_US.UTF-8` | WARN non-UTF-8 locale (ASCII fallback) |
| 4 | `config.load` | Loads `init.lua` and `settings.lua` (§19) | the files loaded | SKIP no `init.lua` yet (defaults) · WARN a Lua error such as `M8001` (defaults, shown under the board's header) |
| 5 | `project.trust` | Finds `.mill/` above the current folder and checks trust (§22) | `~/code/app · trusted` | SKIP no project here · WARN not trusted, loaded data-only (`M9101`) |
| 6 | `theme.apply` | The color scheme | its name | SKIP default |
| 7 | `audio.init` | Opens the audio device with the soundset, volume and voice settings | `soundset=millrace · vol=0.60 · voice=true` | SKIP muted, `off`, quiet hours or no audio device |
| 8 | `audio.load` | Decodes the soundset into memory | `millrace · 起動準備` | SKIP muted |
| 9 | `parts.load` | The built-in parts registry merged with plugin parts | `部品 · 41 built-in · 23 from plugins` | FAIL built-ins corrupt · WARN a plugin's parts ignored (named) |
| 10 | `plugins.verify` | Installed plugins match `mill.lock`; MCP servers reachable | `8 plugins · lock ok` | SKIP none installed · WARN a changed plugin is disabled (`M5301`), an MCP server needs a login (`M6202`) |
| 11 | `agents.probe` | Agent CLIs from the registry on `PATH` (§11) | the agents found | WARN a pipeline uses an agent that isn't installed (`M6101`) |
| 12 | `keymap.bind` | Default keys plus overrides from Lua | `board + editor + wizard bindings ready` | WARN a binding in `init.lua` is invalid (ignored) |
| 13 | `socket.resolve` | The daemon socket path (§20) | the path | FAIL no path can be resolved |
| 14 | `daemon` | `milld` answers on the socket; starts it if not | `already running · pid 48213` or `started · pid 48213 · 38ms` | FAIL `milld` missing or exits during startup |
| 15 | `client.dial` | Connects to the socket | `connected · 2ms` | FAIL the socket disappeared |
| 16 | `handshake` | `Hello`, versions, the first `Snapshot` (§16) | `接続 · mill-tui · milld 0.5.0` | FAIL version or protocol mismatch (`M7003`) |
| 17 | `subscribe` | Opens the event stream | `受信中 · event stream open` | FAIL the daemon refused |
| 18 | `snapshot.parse` | Counts from the snapshot | `7 pipelines · 1 failed · 2 running · 4 ready` | SKIP no pipelines yet |
| 19 | `pipelines.lint` | The daemon's lint results for every pipeline | `配管 · 7 pipelines · lint clean` | SKIP none · WARN some have problems (counts plus the first one) |
| 20 | `triggers.arm` | Triggers armed in `milld` (§13) | `水門 · 3 cron · 1 webhook · 1 git · next 03:00` | SKIP nothing to arm · WARN one couldn't arm, for example the webhook port is in use (`M4401`) |
| 21 | `state.restore` | The saved selection, view and filter, or a `:bg` detach (§18.12) | `selected=nightly-sweep · view=all` | SKIP first run |
| 22 | `renderer.warm` | Builds and caches styles | `styles cached` | |

The splash only reports. It never fixes, installs or asks anything; that's `mill setup` and `mill doctor`. Steps 18–20 read what `milld` already knows from the snapshot. They don't lint or schedule anything themselves.

**Timing**

- **Cascade.** Rows appear 70 ms apart, so even an instant boot reads top to bottom.
- **Minimum.** The splash stays at least 1.2 s. It hands off only when both the last step is done and the minimum has passed.
- **Slow steps.** A step still working after 250 ms shows `[  ..  ]` with a spinner until it finishes:

```

  ≋ ミル · mill machine intelligence logistics layer
  起動中 booting...  0.61s

  [  OK  ] log.init            · ~/.local/state/mill/logs/tui.log
  [  OK  ] paths.ensure        · config + state dirs ok
  [  OK  ] tty.probe           · 124x34 · truecolor · en_US.UTF-8
  [  OK  ] config.load         · ~/.config/mill/init.lua + settings.lua
  [  OK  ] project.trust       · ~/code/app · trusted · .mill/mill.lua
  [ SKIP ] theme.apply         · default
  [  OK  ] audio.init          · soundset=millrace · vol=0.60 · voice=true
  [  OK  ] audio.load          · millrace · 起動準備
  [  OK  ] parts.load          · 部品 · 41 built-in · 23 from plugins
  [ WARN ] plugins.verify      · 8 plugins · lock ok · notion: auth expired (M6202)
  [  OK  ] agents.probe        · claude · codex · gemini · ollama
  [  OK  ] keymap.bind         · board + editor + wizard bindings ready
  [  OK  ] socket.resolve      · /run/user/501/mill/milld.sock
  [  ..  ] daemon              · starting milld ⠹
```

- **Hand-off.** The log is released, the board draws (with the restored view), the startup voice says "mill" (§18.11), the screensaver's idle timer starts, and the TUI starts listening for daemon events.
- **Short or narrow terminals.** When rows don't fit, the oldest scroll off the top; the header and status line stay. Descriptions are cut with `…` and never wrap.

**Failure**

```

  ≋ ミル · mill machine intelligence logistics layer
  起動失敗 boot failed

  [  OK  ] log.init            · ~/.local/state/mill/logs/tui.log
  [  OK  ] paths.ensure        · config + state dirs ok
  [  OK  ] tty.probe           · 124x34 · truecolor · en_US.UTF-8
  [  OK  ] config.load         · ~/.config/mill/init.lua + settings.lua
  [  OK  ] project.trust       · ~/code/app · trusted · .mill/mill.lua
  [ SKIP ] theme.apply         · default
  [  OK  ] audio.init          · soundset=millrace · vol=0.60 · voice=true
  [  OK  ] audio.load          · millrace · 起動準備
  [  OK  ] parts.load          · 部品 · 41 built-in · 23 from plugins
  [ WARN ] plugins.verify      · 8 plugins · lock ok · notion: auth expired (M6202)
  [  OK  ] agents.probe        · claude · codex · gemini · ollama
  [  OK  ] keymap.bind         · board + editor + wizard bindings ready
  [  OK  ] socket.resolve      · /run/user/501/mill/milld.sock
  [ FAIL ] daemon              · milld not found

  cause: error[M7004]: exec: "milld": executable file not found in $PATH
  fix:   make sure milld is on PATH or installed next to mill

                                                                          press q · ctrl+c to quit
```

- The splash freezes on the failed row. Below it, `cause:` shows the error in its short form with its code, and `fix:` shows the hint for that step. `press q · ctrl+c to quit` sits bottom right.
- **Keys.** During the splash, `q`, `esc` and `ctrl+c` quit, whether it's still booting or has failed. Every other key is ignored.
- **On quit,** the full error prints to stderr in the standard format (§29.2), and mill exits with its class's code (§17.2): 4 for the daemon steps, 6 for `paths.ensure`, 1 for `parts.load`.
- **WARN never blocks.** The board opens and the warning stays in `:messages` (§29.8). A config warning also shows under the board's header until it's fixed.

| Failed step | Code | Fix hint |
|---|---|---|
| `paths.ensure` | `M2001` | check that you can write to `~/.config/mill` and `~/.local/state/mill` |
| `parts.load` | `M0002` | mill's built-in parts are corrupt · reinstall or rebuild mill |
| `socket.resolve` | `M7002` | set `XDG_RUNTIME_DIR` or `MILL_SOCKET`, or make sure `HOME` is set |
| `daemon` | `M7004` | make sure `milld` is on PATH or installed next to mill |
| `client.dial` | `M7001` | the daemon socket disappeared · try `mill daemon start` |
| `handshake` | `M7003` | version mismatch · `mill daemon restart`, or build mill and milld at the same version |
| `subscribe` | `M7006` | the daemon refused the event stream · check `~/.local/state/mill/milld.log` |

**First launch.** On a new machine, with no `init.lua` and no saved view, the same splash runs with SKIPs where there's nothing yet. `daemon` starts `milld` for the first time. Then, instead of an empty board, it hands off to first-run setup (§18.8):

```

  ≋ ミル · mill machine intelligence logistics layer
  起動中 booting...  1.24s

  [  OK  ] log.init            · ~/.local/state/mill/logs/tui.log
  [  OK  ] paths.ensure        · config + state dirs created
  [  OK  ] tty.probe           · 124x34 · truecolor · en_US.UTF-8
  [ SKIP ] config.load         · no init.lua yet (defaults)
  [ SKIP ] project.trust       · no .mill/ here
  [ SKIP ] theme.apply         · default
  [  OK  ] audio.init          · soundset=millrace · vol=0.60 · voice=true
  [  OK  ] audio.load          · millrace · 起動準備
  [  OK  ] parts.load          · 部品 · 41 built-in
  [ SKIP ] plugins.verify      · no plugins yet
  [  OK  ] agents.probe        · claude · codex found on PATH
  [  OK  ] keymap.bind         · board + editor + wizard bindings ready
  [  OK  ] socket.resolve      · /run/user/501/mill/milld.sock
  [  OK  ] daemon              · started · pid 48213 · 38ms
  [  OK  ] client.dial         · connected · 1ms
  [  OK  ] handshake           · 接続 · mill-tui · milld 0.5.0
  [  OK  ] subscribe           · 受信中 · event stream open
  [ SKIP ] snapshot.parse      · no pipelines yet
  [ SKIP ] pipelines.lint      · no pipelines yet
  [ SKIP ] triggers.arm        · nothing to arm
  [ SKIP ] state.restore       · first run
  [  OK  ] renderer.warm       · styles cached

                                                   準備完了 ready · 1.38s · first run → mill setup
```

When setup finishes, the board opens. Every later launch goes straight from the splash to the board.

**Sound.** Each row plays its status sound as it appears (`boot_ok`, `boot_warn`, `boot_fail`; SKIP and `[  ..  ]` are silent). The sounds are short enough for the 70 ms cascade (Appendix A) and belong to the `interface` group, so muting interface sounds also mutes the splash. The startup voice plays once, at hand-off, never during the cascade.

**Same checks from the CLI.** `mill doctor` (§29.10) runs these steps plus its deeper checks, and prints the same `[ STATUS ] step · finding` rows without the cascade or sounds. `mill doctor --json` prints them as objects with `step`, `status`, `desc`, `code` and `ms`.

**Implementation.** `internal/surface/tui/splash.go` draws the splash and `splash_steps.go` holds the ordered step list, as in rex. Each step returns a status, a description and an error. The daemon spawn and the audio player can be swapped out in tests, so the hand-off, minimum time and failure freeze are tested without a real `milld`.



---

[← spec index](index.md)
