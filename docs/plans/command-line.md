# Plan: the `:` command line

> **2026-11-12 (v2.3): fullscreen per rex v2.** Settings, help and the new-pipeline wizard are full-screen pages ([spec §18.1.1](../spec/18-tui.md#1811-pages), ADR-055). On a page the `:` line and `/` search take the help bar's row; `:help commands`, `:help <command>` and `:help <code>` open the help page ([§18.5.1](../spec/18-tui.md#1851-help-page)). Everything else here stands.

Status: **decided and merged into spec v2.2** (§18.2, §18.4, §18.5, §18.6, §18.7.14, §18.12, §17.5, §29) on 2026-11-12. The spec wins; section 8 is kept as history. Decisions: ADR-051 to ADR-053.
Spec: §18.1 (rules), §18.2 (board, the `λ` prompt), §18.4 (focus modes), §18.5 (keys), §18.6 (run), §18.7.4 and §18.7.14 (editor modes and `:`), §18.8 (wizard), §17.5 (TUI ↔ CLI), §29.4 and §29.8 (errors).
Prototype: `mockups/v2/index.html`: press `:` anywhere; `:help commands` lists every command.
Companion plan: [wizard.md](wizard.md).

User requirement: commands are vim-like, typed after `:` on the bottom row, never after `/`; `:` works from everywhere and sits in the bottom-left corner like nvim's cmdline. `/` is search only.

## 1. Design

mill copies nvim's command line, not a palette.

1. **One line, the terminal's last row, column 0.** `:` is drawn in the bottom-left corner with the cursor after it, exactly where nvim draws its cmdline. It is never inside an overlay, never centred or indented. While it is open it replaces whatever the last row showed (the help bar, the last echo). Overlays and popups don't move: the command line draws below them on the terminal's own last row.
2. **Global.** `:` opens it from every NORMAL-like state: the boot screen, the board, every popup and overlay (run, log, items, cause, agent screen, plugins, settings, stats, help, `:messages`, `:runs`, `:help commands`; not a pending y/n confirm, which swallows `:` as nvim's `confirm()` does, ADR-052), the editor (NORMAL, VISUAL, CONNECT, the port picker, the inspector when not typing, help, diff, the YAML pane, the problems list) and the wizard (NEW · NORMAL, the file picker, help, while drafting). No screen swallows `:`.
3. **INSERT types it.** In INSERT and in every text field (the wizard's fields, the inspector's fields, the prompt and YAML editors, the palette's query, any `/` search, the settings value editor, setup's trusted-folders field) `:` is a literal colon, as in nvim. The palette counts as a text field because part ids contain `:` (`mcp:slack/send_message`).
4. **Idle.** When it isn't open, the last row shows the help bar, or the echo of the last command until the next key, left-aligned like nvim's message area. The mode name stays where §18.4 puts it: at the left of the status line (the row above) on screens that have one (popups, overlays, the editor, the wizard: `RUN`, `LOG`, `NORMAL`, `NEW · NORMAL`), and at the left of the help bar on the board, which has no status line. While the command line is open the board's help bar (and its mode name) is replaced, as nvim replaces `-- INSERT --`.
5. **Context.** Completion lists and orders commands for the screen it was opened from (board commands first on the board, editor commands in the editor, …), then the global ones. A command that belongs to another screen echoes `M1307 :sort works in the board, not here`; global commands (`:new`, `:run`, `:pipe`, `:settings`, `:plugins`, `:q`, `:help`, …) work everywhere. `esc` returns to exactly the state it was opened from: the same popup, selection, mode and pending field.
6. **Mouse.** A click on the last row opens the command line (with `mouse_enabled` on, not while a text field has focus). It is the one click an overlay doesn't swallow (§18.1 rule 10).
7. **Search is `/`, also on the bottom row.** `/` opens a search line in the same place (`/state:failed▌`), on the board, in the log, in plugins and settings, in the wizard's gallery and list, and as the editor's jump-to-node. `/` never runs a command, and no command starts with `/`.
8. **Results echo on the same row**, in mill's error shape: `error[M1305]: unknown sort "bogus"  ·  help: attention, name, last run, ok, cost`, or a plain result (`≋ nightly-sweep#31 started`). The echo fades on the next key. Every result and error is also kept in `:messages` (§29.8), newest first.

### 1.1 Completion

`tab` completes, `shift+tab` goes back. The first `tab` fills the longest common prefix and, when several candidates remain, opens the **wildmenu** on the row above (the status line's row, as nvim's `wildmenu` does); further `tab`s cycle it and fill the line. Typing closes it. `ctrl+n`/`ctrl+p` move in it too.

What completes, by argument:

| Argument | Candidates |
|---|---|
| command name | Commands for this screen first, then global ones, alphabetical within each group |
| `<pipeline>` | Pipeline names |
| `:run <p> …` | That pipeline's undeclared inputs as `key=`, then `--dry-run` |
| `:set` | Settings keys as `key=`; after `key=`, the key's values |
| `<node>` | Node ids of the pipeline in the editor |
| `<path>` | Folders (ending in `/`) and files, from the current directory |
| `<topic>` | `commands`, `keys`, screen names, every command, every error code |
| enumerations | Sorts, views, themes, soundsets, steps, templates, triggers, budgets, agents, plugins, settings sections |

### 1.2 History

The last 200 command lines, per user (`~/.local/state/mill/cmdline_history`), shared by every screen. `↑`/`↓` (and `ctrl+p`/`ctrl+n` when the wildmenu is closed) walk the entries that start with what is typed so far, as in vim. `ctrl+r` searches backwards for a substring: `(history) bog▌  :sort bogus`; `ctrl+r` again goes older, `enter` runs the match, `esc` (or `tab`, `→`) puts it on the line to edit, `ctrl+g` cancels.

### 1.3 Grammar

`:[count]command[!] [args]`, like vim:

- **Abbreviations.** A unique prefix works (`:mes`, `:sett`); an ambiguous one is an error (`M1304 :s is ambiguous: :sort :settings :set :stats :sound`), and the line's hint shows the candidates while you type. Fixed short forms from vim: `:q :w :wq :x :e :e! :h :n :r :u :red :se :f :mes :qa :est`.
- **Bang.** `!` forces: `:q!` (leave without saving), `:w!` (save with lint errors, ADR-034), `:e!` (revert the buffer), `:new!` (start the wizard over).
- **Count.** A number alone jumps: `:3` selects the third board row, the third node in run order, the third wizard step, the third log line, or the third run row. Before a command it repeats it: `:3undo`.
- **Arguments** are words; `"…"` keeps spaces together. `k=v` pairs pass inputs (`:run issue-to-pr task="fix the flaky test"`).

### 1.4 The board's `λ` prompt becomes commands

The `λ` row (§18.2, opened with `i`) is removed. Its two jobs become commands with the same completion: `:run <pipeline> k=v …` (with `--dry-run`) and `:pipe <expr>`. The board's last row is the help bar; the row above it is free.

`i` means INSERT everywhere (enter a text field). The board has no text field, so `i` there does nothing but echo `i edits a text field · on the board, :run <pipeline> k=v runs one and :pipe runs a pipe`. In the run popup, items move from `i` to `gi` (as in the editor); `i` stays as a quiet alias for one release.

## 2. Bottom-row states

100-column terminal; the last two or three rows of the screen. Captured from the prototype.

**Idle on the board** (the help bar; mode name at its left only for non-default modes):

```

 enter open · n new · e edit · R run · ? more
```

**`:` typed** (the row above shows what's available here):

```
 board first: sort filter view enable disable rerun quit qall … · tab lists them · ↑ history · ctr…
:▌
```

**Typing a command** (the hint names the command, what it does and its CLI):

```
 :ru… could be :runs :run · tab completes
:ru▌
```

**Wildmenu** after `tab` (longest common prefix filled; the next `tab` selects `run`):

```
 run  runs
:run▌
```

**Completing inputs** after `:run issue-to-pr ` + `tab`:

```
 issue=  repo=  --dry-run
:run issue-to-pr ▌
```

**Error echo** (same row, until the next key; also in `:messages`):

```
error[M1305]: unknown sort "bogus"  ·  help: attention, name, last run, ok, cost
```

```
error[M1306]: unknown command ":foo"  ·  help: :help commands lists them
```

```
error[M1304]: :s is ambiguous: :sort :settings :set :stats :sound  ·  help: type more letters, or …
```

**Result echo:**

```
≋ nightly-sweep#31 started
```

**History search** (`ctrl+r`):

```
(history) sort▌  :sort bogus                                    enter run · esc edit · ctrl+r older
```

**Search** (`/`, same row, never a command):

```
/state:failed▌                                                               enter keep · esc clear
```

### 2.1 Open over other screens

Over the run popup (the popup stays put; the hint row covers the status line):

```
 ≋ mill                                           2 failed · 3 running · 14 ok today · $13.70 today
      all   code   github   mail   daily
    ┌ issue-to-pr#42 · ↻ retrying · 7m20s · 410K tk · $2.69 of $4.50 ──────────────────────────┐
 FAI│                                                                                          │
   ↯│  ✓ webhook     event #1287 "Webhook sender drops events on …  ▏                      1s  │ 3h
   ⊗│  ✓ ws          mill/issue-1287                                ▏                      1s  │ 1h
 RUN│  ✓ plan        claude/opus · risk medium, 3 files · 210K tk   ████████            5m38s  │
 ▸ ↻│  ✓ scope       pass · risk medium, 3 files                            ▏              1s  │ 7m
   ⠋│    └─ else – hand-back skipped · scope passed                                            │12s
   ⠋│  ↻ build       try 2 · rate limited, next try in 21s · loop…           ▏            19s  │12s
 REA│    └ try 1 ✗   M6201 rate limited by the provider (429) · r…          █           1m20s  │
   ✓│  ○ review                                                                                │ 2h
   ✓│  ○ pr                                                                                    │ 4h
   ✓│  ○ ping                                                                                  │ 3d
   ○│                                                                                          │  -
    │                                                                                          │
    │  enter agent · L log · F rerun from here · gi items · esc back                           │
    └──────────────────────────────────────────────────────────────────────────────────────────┘

 :cancel · cancel this run · CLI: mill cancel <run>
:cancel▌
```

In the editor (`:goto ju` + `tab`):

```
│  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ││ est  ≈ 2s                │
│                                                                      ││                          │
└──────────────────────────────────────────────────────────────────────┘└──────────────────────────┘
 :goto · jump to a node · CLI: mill show <p> --outline
:goto judge ▌
```

In the wizard (`:template ` + `tab`; the wildmenu replaces `NEW · NORMAL` while it is open):

```
    └──────────────────────────────────────────────────────────────────────────────────────────┘

 issue-to-pr  nightly-sweep  morning-brief  standup-digest  plan-check-handoff  best-of-n         >
:template ▌
```

At 80 columns over the run popup:

```
    │  enter agent · L log · F rerun from here · gi items · esc back       │
    └──────────────────────────────────────────────────────────────────────┘

 :log · this run's log · CLI: mill logs <run>
:log▌
```

## 3. Keys

| Key | Where | Action |
|---|---|---|
| `:` | any NORMAL-like state | Open the command line on the last row |
| click on the last row | same | Open it (mouse) |
| type, `backspace` | the line | Edit; `backspace` on an empty line closes it, as in vim |
| `ctrl+w` / `ctrl+u` | the line | Delete a word / clear the line |
| `tab` / `shift+tab` | the line | Complete; open and cycle the wildmenu |
| `ctrl+n` / `ctrl+p` | wildmenu open | Next / previous candidate |
| `↑` / `↓` (`ctrl+p` / `ctrl+n`) | the line | Older / newer history entry that starts with what's typed |
| `ctrl+r` | the line | Reverse history search; again for older |
| `enter` | the line | Run; the result or error echoes on the same row |
| `esc`, `ctrl+c` | the line | Cancel; back to exactly where you were |
| any key | after an echo | Clears the echo (and does its usual job) |
| `/` | board, log, plugins, settings, wizard lists, editor | Search on the last row; `enter` keeps, `esc` clears; `n`/`N` hits in the log |

## 4. Command catalog

Grouped by the screen they belong to; global commands work everywhere. `[…]` is optional; without a pipeline, board commands use the selected row. Generated from the prototype's table (`CMDS`), so the two agree.

**global**

| Command | Does | CLI |
|---|---|---|
| `:quit` (`:q`), `:q!` | Close what's open (popup, overlay, wizard, editor); on the board, quit (confirm); `!` drops unsaved edits | `ctrl+c` (milld keeps running) |
| `:qall` (`:qa`) | Quit from anywhere (asks to save the editor) | — |
| `:help [<topic>\|<command>\|<code>]` (`:h`) | Keys for this screen; `:help commands` lists every command; `:help run`, `:help M2301` | `mill <command> --help` · `mill explain <code>` |
| `:messages` (`:mes`) | This session's errors, warnings and command results | `mill daemon logs` |
| `:settings [<filter>]` | Settings | `mill config list` |
| `:set key=value`, `key?`, `key&` (`:se`) | Change a setting, show it, reset it | `mill config set <key> <value>` · `mill config get` · `mill config reset` |
| `:plugins` | Plugins, MCP servers and agents | `mill plugin ls` |
| `:stats` | Runs, success, time, cost | `mill stats` |
| `:runs [<pipeline>]` | Older runs | `mill runs [--pipeline p]` |
| `:new [<name>] [--from …] [--trigger …] [--budget …] [--agent …]` (`:n`), `:new!` | The new pipeline wizard (wizard.md §9) | `mill new …` |
| `:edit [<pipeline>]` (`:e`), `:e!` | Open in the editor; `:e!` reverts the buffer | `mill edit <pipeline>` |
| `:run <pipeline> [k=v]… [--dry-run]` (`:r`) | Run with inputs (replaces the `λ` prompt) | `mill run <p> [--input k=v]… [--dry-run]` |
| `:pipe <expr>` | Run a pipe expression (replaces the `λ` prompt) | `mill pipe '<expr>'` |
| `:trust [<path>]` | Trust a project's pipelines | `mill trust <path>` |
| `:explain <code>` | What an error code means | `mill explain <code>` |
| `:theme [<theme>]` | Color theme | `mill config set theme <name>` |
| `:sound [on\|off\|<set>]` | Sound on, off, or a soundset | `mill config set sound.set <set>` |
| `:bg` (`:detach`) | Back to the shell; runs keep going | — (`mill` reattaches) |
| `:screensaver` | Start the screensaver now | — |
| `:<n>` | Jump to row / node / step / line `n` | — |

**board**

| Command | Does | CLI |
|---|---|---|
| `:sort <by>` | Sort inside each section (attention, name, last run, ok, cost) | `mill ls --sort <by>` |
| `:filter <expr>` (`:f`) | Filter the rows (same tokens as `/`) | `mill ls --filter <expr>` |
| `:view <name>` | A saved view | `mill ls --view <name>` |
| `:enable [<pipeline>]` / `:disable [<pipeline>]` | Arm / disarm its triggers | `mill trigger enable\|disable <p>` |
| `:rerun [<pipeline>]` | Rerun the latest failed run from where it failed | `mill rerun <run> --from <node>` |

**editor**

| Command | Does | CLI |
|---|---|---|
| `:write` (`:w`), `:w!` | Save; refuses lint errors (`M2203`), `!` saves anyway with triggers off | `mill node …` / `mill pipeline set …` (they save) |
| `:wq` (`:x`) | Save and close | — |
| `:lint`, `:copen`, `:cclose` | Lint and open / close the problems list | `mill lint <p>` |
| `:goto <node>` | Jump to a node | `mill show <p> --outline` |
| `:refs [<node>]` | Everything that uses a node | `mill refs <p>/<n>` |
| `:wrap loop` | Wrap the selection in a loop | `mill wrap <p> loop <n>…` |
| `:extract <name>` | The selection becomes a sub-pipeline | `mill extract <p> <n>… --as <name>` |
| `:layout reset` | Unpin every node | `mill layout <p> reset` |
| `:pattern save <name>` | Save the selection as a pattern | `mill pattern save <p> <n>… --as <name>` |
| `:prompt file\|inline` | Move a prompt to a file, or inline it | `mill node set <p>/<n> prompt=@file` |
| `:sample edit` | Edit the pinned sample | `mill node pin <p>/<n> --file items.jsonl` |
| `:[n]undo` (`:u`), `:[n]redo` (`:red`) | Undo / redo, with a count | — |
| `:dry` | Dry run the buffer | `mill run <p> --dry-run` |
| `:estimate` (`:est`) | Cost and time per run | `mill estimate <p>` |
| `:diff`, `:yaml`, `:outline` | The diff, the YAML pane, the outline | `mill show <p> --yaml\|--outline` |

**wizard**

| Command | Does | CLI |
|---|---|---|
| `:create [--dry-run]` (`:w`, `:wq`, `:x`) | Create and open the editor | `mill new …` (every step is a flag) |
| `:step <name>\|<n>` | Go to start, shape, run or review | — |
| `:describe "<sentence>"` | Start from a sentence | `mill new --describe "…"` |
| `:template <id>` | Start from a template or pattern | `mill new --from template:<id>` |
| `:dup <pipeline>` | Start from a copy | `mill new --from pipeline:<p>` |
| `:import <path\|url\|gh:…>` | Start from a file | `mill new --from <file\|url\|->` |
| `:name <name>` | Set the name | `mill new <name>` |
| `:trigger <spec>` | Set the trigger (`schedule 0 3 * * *`, `webhook /x`, `after p`, …) | `mill new --trigger <spec>` |
| `:budget small\|medium\|large` | Set the budget | `mill new --budget …` |
| `:agent <id/model>` | Set the agent | `mill new --agent <id/model>` |

**run popup** (and its log, items, cause and agent views)

| Command | Does | CLI |
|---|---|---|
| `:cancel` | Cancel this run (confirm) | `mill cancel <run>` |
| `:log` | This run's log | `mill logs <run>` |
| `:items` | The selected node's items | `mill items <run>/<node>` |
| `:attach` | The selected agent's screen, view-only | `mill attach <run>/<node>` |

**plugins**

| Command | Does | CLI |
|---|---|---|
| `:install <plugin>` | Install (the confirm shows what it adds and can touch) | `mill plugin add <id>` |
| `:update [<plugin>]` (`:up`) | Update one, or every one with an update | `mill plugin update [<id>]` |
| `:remove <plugin>` | Remove one nothing uses (`M5105` otherwise) | `mill plugin rm <id>` |
| `:check [<plugin>]` | Health-check a plugin, MCP server or agent | `mill doctor` · `mill agent test <id>` |

**settings**

| Command | Does | CLI |
|---|---|---|
| `:section <name>` | Jump to a section | `mill config list` |
| `:reset [<key>]` | Reset the selected (or named) setting; `:set key&` does it anywhere | `mill config reset <key>` |

Boot: `:q` quits (and stops setup on first run); `:help <code>`, `:explain`, `:set`, `:sound`, `:theme` work; anything else echoes `M1307 … works once mill is up`.

### 4.1 Error codes

`M1xxx` is "CLI, selectors, arguments" (§29.4). `M1301` stays "unknown flag"; commands get their own codes, checked free in §29.4 and §18.7.9:

| Code | Meaning | Example |
|---|---|---|
| `M1302` | An input isn't `key=value` | `:run issue-to-pr fix` |
| `M1303` | The pipeline has no such input | `:run issue-to-pr tsk=…` |
| `M1304` | Ambiguous abbreviation | `:s` |
| `M1305` | An argument isn't one of the values it takes | `:sort bogus`, `:step 9`, `:3` with two rows |
| `M1306` | Unknown command (with `did you mean`) | `:foo` |
| `M1307` | The command works on another screen (names it) | `:sort` in the editor, `:install` on the board |
| `M1308` | The command needs an argument | `:run`, `:explain` |

`M1302` and `M1303` were already used by the prototype's `λ` prompt; this plan specs them. Command-specific errors keep their own codes (`M1201` pipeline not found, `M2203` lint errors not saved, `M3012` unknown node, `M5101` plugin not found, `M8004` unknown setting).

## 5. Audit: `/` commands and bottom prompts

No `/`-prefixed command exists in the spec or the prototype; every `/` is a search. The audit found these prompts that weren't on the `:` row, or keys that conflict with it:

**Spec**

| Where | Today | Fix (spec patch §8) |
|---|---|---|
| §18.2 drawing + **Prompt** | A `λ` row above the help bar; `i` focuses it; it runs a pipeline with inputs or a pipe | Remove the row and the paragraph; `:run` / `:pipe` with completion; new **Command line** paragraph |
| §18.4 | `Prompt` mode row; `Command` row says only `:`; mode-name example `PROMPT  enter run · …` | Drop `Prompt`; `Command` row: from any NORMAL-like state, last row; mode-name paragraph rewritten |
| §18.5 board keys | `i` focus the prompt; `:` "Command mode" | `i` removed from the board; `:` command line (last row) |
| §18.5 mouse | "Overlays and prompts swallow mouse events" | Except a click on the last row, which opens `:` |
| §18.6 run | `i` shows items | `gi` (as in the editor) |
| §18.7.5, §18.7.6 | `/` jump, `r` rename and `gP` extract prompts have no stated place | They edit on the last row, like `:` and `/` |
| §18.7.14 | `:` row lists a few commands | Points to the catalog; adds counts, `:goto`, `:refs`, `:dry` … |
| §18.8 | Wizard keys: `backspace` back, typing in place | wizard.md (NORMAL / INSERT, `:` commands) |
| §18.9, §18.10 | `/` filter and search drawn in the overlay | Typed on the last row; the overlay shows the active filter |
| §18.12 | "prompts … never time out" | "the command line …" |
| §17.5 | Row `` `λ` prompt `` | Row `:run`, `:pipe` |
| §29.8 | Errors appear "in the statusline" | Errors from a `:` command echo on the last row; others stay in the status line |
| `docs/spec/index.md` (v2 changes table), `docs/tui.md`, `docs/plans/editor.md` §4 | Mention the `λ` prompt | Same wording change when §18.2 is patched (outside this plan's edit scope) |

**Prototype** (all fixed in `mockups/v2/index.html`)

| Where | Was | Now |
|---|---|---|
| Board | `λ` row on the second-to-last row; `i` opened it; `PROMPT`/`COMMAND`/`SEARCH` labels on the help bar | No `λ` row; `:` and `/` edit on the last row; `i` echoes the `:run`/`:pipe` hint |
| Board search | Edited in the header's right corner | Edited on the last row; the header keeps showing the active filter |
| Log, plugins, settings | `/` edited inside the popup / overlay | Last row |
| Wizard gallery and list filters | Inline | Last row |
| Editor line prompts (rename, extract, `/` jump) | On the status line's row | Last row |
| `:` swallowed | Boot, confirm dialogs, help overlay (any key closed it), editor help, port picker, wizard help / picker / drafting | Opens the command line; `esc` returns. Exception (ADR-052): a pending y/n confirm still swallows `:`, as nvim's `confirm()` |
| Run popup | `i` items | `gi` (`i` kept as an alias) |
| Unknown command | `M1301` (that's "unknown flag") | `M1306`; new `M1304`, `M1305` (extended), `M1307`, `M1308` |
| Page footer | "i λ prompt (tab completes inputs)" | ":run p k=v · :pipe (tab completes, ↑ history, ctrl+r searches it) · :help commands" |

## 6. Implementation (Go)

- `internal/surface/tui/cmdline`: the Bubble Tea component: line editing, wildmenu, history (file-backed, 200 entries, `ctrl+r`), echo, and the parser (`[count]name[!] args`, quotes, `k=v`). Pure functions for parse and complete, table-tested.
- A command registry: `Command{Name, Aliases, Context, Arg ArgKind, CLI string, Help string, Run func(ctx, Args) error}`. Screens register their commands on init; `:help commands`, completion and `mill help` read the same table, so the catalog can't drift.
- The root model owns the last row: every screen renders into `height-1` rows (plus the status line where it has one) and the root draws the help bar, the echo or the command line. That is what makes `:` global and bottom-left by construction, and why no screen can swallow it: the root sees `:` before the focused screen unless that screen reports a focused text field (`TextFocused() bool`).
- Errors use the shared `errs` package, so an echo, `:messages` and `mill … --json` show the same code and help line.

## 7. Decisions (2026-11-12)

| # | Decision | Chosen | By |
|---|---|---|---|
| 1 | The `λ` prompt | Removed; `:run <p> k=v` and `:pipe <expr>` | default (ADR-053) |
| 2 | `i` on the board | Hint only: echoes the `:run`/`:pipe` hint | user (ADR-051) |
| 3 | Wildmenu | One row above the command line, `<` `>` scroll | default |
| 4 | Ordering | This screen first, then global, alphabetical; abbreviations never resolve by context | default |
| 5 | Palette and `:` | `:` types a colon in the palette query | default |
| 6 | History | Per user, 200 entries, state dir | default |
| 7 | Confirm dialogs and `:` | **A pending y/n confirm swallows `:`**, as nvim's `confirm()`: only `y`, `n`, `esc`. `:` works on every other screen. | user (ADR-052) |
| 8 | Run popup `i` | `gi`; `i` a quiet alias for one release | default |
| 9 | Echo lifetime | Until the next key, 12 s at most | default |

## 8. Spec patches (merged in v2.2; history only, the spec wins)

### §18.2 The board

In the drawing, delete the `λ` line (the blank line above it stays). Replace the **Prompt** paragraph with:

````md
**Command line.** `:` opens the command line on the last row (§18.5). `:run <pipeline> k=v …` runs a pipeline with inputs (`:run fix-flaky-test test=TestCache_Expiry`, `--dry-run` for a dry run) and `:pipe <expr>` runs a pipe expression (`:pipe sh "git log -5" | agent claude/haiku "summarize"`, §14.2). `tab` completes pipeline names and input keys. The new run's pipeline is selected when it starts.
````

### §18.4 Focus modes

Delete the `Prompt` row. Replace the `Search` and `Command` rows with:

````md
| Search | `/` (board, log, plugins, settings, wizard lists, editor) | `enter` keeps it, `esc` clears it |
| Command | `:` from any screen or mode except a text field, or a click on the last row | `esc` (back to exactly where you were), or after the command runs |
````

Replace the paragraph under the table with:

````md
The mode name shows the way vim shows `-- INSERT --`: at the left of the status line on screens that have one (popups, overlays, the editor, the wizard: `RUN`, `LOG`, `NORMAL`, `NEW · INSERT`), and at the left of the help bar on the board, which has no status line (the default board mode shows no name). The last row belongs to the help bar, the echo of the last command, or the command line and `/` search while you type, always starting at column 0.
````

### §18.5 Keys and mouse

In the board key table, delete the `i` row and replace the `:` row with `` | `:` | Command line on the last row (below) | ``. In **Mouse**, replace "Overlays and prompts swallow mouse events." with "Overlays swallow mouse events, except a click on the last row, which opens the command line." Add after **Mouse**:

````md
**Command line.** `:` opens one command line, vim-style, on the terminal's last row with `:` in the bottom-left corner. It works from every screen and mode (the boot screen, the board, every popup and overlay, a confirm, the editor's NORMAL, VISUAL and CONNECT, the wizard); in INSERT and other text fields `:` types a colon. Overlays don't move; the line draws below them.

- Commands are `:[count]name[!] [args]`. Unique prefixes and vim's short forms work (`:q :w :wq :x :e! :mes :se`); an ambiguous prefix is `M1304`. `!` forces (`:q!`, `:w!`, `:e!`, `:new!`). A number alone jumps to that row, node, step or line; before a command it repeats it (`:3undo`).
- `tab` completes commands (this screen's first), pipelines, node ids, settings keys and values, `k=v` inputs and paths; several matches open a one-row wildmenu above the line. `↑`/`↓` walk the history entries that start with what's typed; `ctrl+r` searches it. `esc` cancels, `enter` runs.
- The result or error echoes on the same row in the usual shape (`error[M1305]: unknown sort "bogus" · help: …`) until the next key, and goes to `:messages`. A command from another screen echoes `M1307` naming where it works; global commands work everywhere.
- `:help commands` lists every command with its CLI equivalent; `:help <command>` and `:help <code>` explain one. `/` is search only, on the same row; no command starts with `/`.
````

### §18.6 Run

Replace "**`i`** shows the selected node's input and output items." with "**`gi`** shows the selected node's input and output items." and in the drawing's help line `i items` with `gi items`.

### §18.7.14 Keys (the `:` row)

````md
| `:` | Command line (§18.5): `:w`, `:w!`, `:q`, `:q!`, `:wq`, `:x`, `:e!` (revert), `:lint`, `:copen`, `:cclose`, `:goto <node>`, `:refs`, `:wrap loop`, `:extract <name>`, `:layout reset`, `:pattern save <name>`, `:prompt file\|inline`, `:sample edit`, `:dry`, `:estimate`, `:[n]undo`, `:[n]redo`, `:<n>` (n-th node in run order). Editor commands complete first. `r` (rename), `gP` (extract) and `/` (jump) edit on the same last row |
````

### §18.12

Replace "Popups, the editor, the wizard, settings, prompts and confirmations never time out" with "Popups, the editor, the wizard, settings, the command line and confirmations never time out".

### §17.5

Replace the `` `λ` prompt `` row with:

````md
| `:run`, `:pipe` | `mill run <p> --input k=v`, `mill pipe '…'` |
| `:` commands | Each lists its CLI in `:help commands` |
````

### §29.4 and §29.8

In §29.4's `M1xxx` examples add: `` `M1304` ambiguous command · `M1306` unknown command · `M1307` command belongs to another screen ``. The full list for the explain pages (§29.5): `M1302`–`M1308` as in §4.1 above.

In §29.8, replace the first bullet's first sentence with: "**Never modal, never steal focus.** An error from a `:` command echoes on the last row; any other error appears in the status line as one red line with its code: `M5201 linear.issue.create: team "ENGG" not found  gx explain`."
