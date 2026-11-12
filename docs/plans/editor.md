# Plan: the pipeline editor

Status: accepted 2026-11-11 (decisions in §5) · spec: §18.7 in `docs/spec/18-tui.md` · prototype: `mockups/v2/index.html` (press `e` on a pipeline, `?` in the editor lists every key).

The editor is where pipelines get made. Today's §18.7 (v2 draft) had a graph, an inspector, a palette and a handful of keys: enough to look at a pipeline, not enough to build an agent pipeline without opening the YAML. This plan lists everything the editor needs, ranks it, says how to build it in Go, and lists the decisions that still need an answer.

Ranking:

- **MUST**: without it you can't build a real agent pipeline (branching, loops, handoffs, budgets, safe destructive steps) in the editor, or you'd build a broken one without knowing.
- **SHOULD**: makes building fast and safe; v0.3 is weaker without it but still works.
- **LATER**: valuable, not needed for v0.3.

## 1. Feature inventory

### 1.1 Files and model

| Feature | Rank | Why |
|---|---|---|
| Buffer is the YAML node tree (`go.yaml.in/yaml/v4`, edits as text splices); graph, layout, lint, inspector derived on every change | MUST | Files are the truth (§2.5). Anything else drifts from the file and loses comments. |
| Every action = one YAML edit (or a sidecar pin), one undo step, named | MUST | Predictable undo; the CLI can do the same thing. |
| Edits shared with the CLI (`internal/pipeline/edit`) | MUST | Principle 9 (headless first) and one tested code path. |
| Layout sidecar with slot pins only | MUST | Moving a box must never touch the pipeline file (§5.3). |
| Edge inference identical to the compiler (refs, `in`, `needs`, `edges:`) | MUST | The graph must show what will run, not an approximation. |
| Keep the file's style (`in:` vs `edges:`, flow vs block maps) | SHOULD | Hand edits and editor edits on the same file stay readable. |
| Swap file for unsaved buffers, recovery on next open | SHOULD | Terminals die; a lost half-built pipeline is a bad first week. |
| Lua-defined pipelines read-only | MUST | §19.7 already says so; the editor must not pretend it can write them. |

### 1.2 Make

| Feature | Rank | Why |
|---|---|---|
| Palette (`a`) with fuzzy search and categories (exec, data, control, agents, git/gh, io, mcp, plugins, triggers, patterns) | MUST | The only way to discover parts from plugins and MCP servers. |
| Insert after a port, before a node, as a branch, on an edge (`a`, `i`, `o`, edge focus) | MUST | Real pipelines are edited in the middle, not only at the end. |
| Required settings filled with placeholders, focus jumps to them, placeholders are lint errors | MUST | A new node is never silently incomplete. |
| Triggers as root nodes, add/remove/edit (`gt`) | MUST | Every unattended pipeline needs one; they belong on the graph. |
| Patterns: plan → check → handoff, best of N, fix until green, review → PR, triage switch, each with a limit, failure → handoff | MUST | These *are* agent pipelines (§11.5). Building each from parts by hand is where people give up. |
| Yank, paste connected/unconnected, duplicate with reference rewriting | SHOULD | Fanning out similar steps is common. |
| Delete with healing for single in/out nodes | SHOULD | Removing a step in a chain shouldn't leave a gap to reconnect. |
| Rename with every reference updated (`r`) | MUST | Ids appear in expressions everywhere; a manual rename breaks pipelines. |
| Wrap the selection in a loop (`gl`) / unwrap (`gU`) | MUST | Fix-until-green on a multi-node body has no other editor path. |
| Extract to a sub-pipeline (`gP`) | SHOULD | Reuse; also keeps big pipelines readable. |
| Replace a node's part keeping compatible settings | SHOULD | Swapping `agent` for `agent.loop` is a common refinement. |
| Save selection as a pattern; plugin-provided patterns | LATER | Team conventions; needs the pattern format to settle first. |

### 1.3 Move and navigate

| Feature | Rank | Why |
|---|---|---|
| Deterministic layered layout, left to right, slots | MUST | Same file, same picture; `mill render` matches. |
| Directional `hjkl`, run-order `w`/`b`, `gg`/`G`, jump `/`, jump back `ctrl+o` | MUST | Keyboard-first is the product. |
| Follow references (`gd`) and find usages (`gr`) | MUST | Expressions tie nodes together invisibly otherwise. |
| Manual nudges (`H J K L`, drag) saved as pins; edges always point right | SHOULD | Readability on bigger graphs; the right-pointing rule keeps routing sane in a cell grid. |
| Zoom: cards, compact, dots; fit (`=`) | MUST | Agent pipelines reach 15–30 nodes; cards alone don't fit. |
| Visual mode multi-select, select downstream (`V`) | MUST | Wrap, extract, move and delete work on groups. |
| Minimap (`M`), outline view (`O`) | SHOULD | Orientation on big graphs; the outline is also the accessible view. |
| Align selection to a row or column | LATER | Nice, but pins and auto-layout cover most of it. |
| Folding loops, sub-pipelines and fanout candidates (`zo`/`zc`) | SHOULD | Big loops crowd the canvas. |

### 1.4 Connect

| Feature | Rank | Why |
|---|---|---|
| Port-aware connect with a port picker (`c`) | MUST | Branches only work if you can wire `false`, `else`, `case.*`, `exhausted`, `error`. |
| Ordering-only edges (`C`, `needs`), drawn dotted | MUST | Some steps share no data but must run in order. |
| Live compatibility check while choosing a target (schema fields, workspace, cycles) | MUST | Catches the commonest mistakes before they exist. |
| Disconnect, reroute, insert on edge with edge focus (`tab`) | MUST | Edges are first-class objects to edit. |
| Explain inferred edges and offer the reference or an explicit `in:` | MUST | Otherwise "I deleted the edge and it came back" is the first bug report. |
| Mouse: drag from a port to a node | SHOULD | Expected by anyone who used n8n. |
| Edge labels with item counts during dry runs | SHOULD | Shows where items went (and where they didn't). |

### 1.5 Branches, loops, errors

| Feature | Rank | Why |
|---|---|---|
| `if`, `switch` (editable cases = ports), `assert` with `fail/route/drop`, `check` true/false | MUST | The decision points of hands-off pipelines. |
| Loop containers with `until`, required `max_iters`, `exhausted` port | MUST | Bounded retries are how agents work unattended. |
| `on_error: route` adds an `error` port | MUST | Failures need a destination other than "the run fails". |
| `handoff` as a terminal node; edges out of it refused | MUST | Hands-off principle: a person is reached by ending the branch. |
| `merge` modes, ordered inputs for zip/concat | SHOULD | Joins after fanouts and branches. |
| `discard` for intentional drops | MUST | Without it, M3030 can't tell a bug from a decision. |
| What-if skip preview (`zs`) | SHOULD | Skip propagation is the least intuitive rule in §15.2. |
| Fanout drawn stacked with its candidates | SHOULD | Makes best-of-N readable at a glance. |

### 1.6 Lint

| Feature | Rank | Why |
|---|---|---|
| Live client-side lint on every edit; daemon lint (debounced) for machine-dependent checks | MUST | You need to see a mistake when you make it. |
| Badges on nodes and edges, header counts, problems panel (`!`), `]e`/`[e` | MUST | Same quickfix model as the rest of mill (§29.8). |
| Quick fixes (`ga`), fix-all-safe (`gA`), never auto-widen policy | MUST | Speed without silently granting destructive permissions. |
| Checks M3001–M3302, M9201–M9203, M9208 (§18.7.9) | MUST (errors) / SHOULD (warnings, hints) | Graph, references, schemas, workspace, budget, policy, hands-off. |
| Save blocked on errors unless `:w!`; saved-broken pipelines stay disarmed | MUST | A broken pipeline must never run unattended. |
| Explain (`gx`) | SHOULD | Same pages as `mill explain`. |

### 1.7 Inspector and pipeline panel

| Feature | Rank | Why |
|---|---|---|
| Forms generated from part schemas; typed fields (choice, toggle, number, list, part/agent pickers) | MUST | Plugins and MCP tools need forms without custom UI. |
| Expression completion (`inputs.`, `nodes.<id>.item.<field>`, `event.`, `secrets.`, `loop.`) | MUST | Expressions are where pipelines break; completion is the cheapest fix. |
| Live preview of expressions against samples or the last dry run | MUST | Turns "will this work?" into a glance. |
| Prompt editor (multi-line, completion, token estimate, `$EDITOR`, move to/from `prompts/*.md`) | MUST | Agent pipelines are mostly prompts. |
| Output schema editor, infer from sample | MUST | Downstream completion, lint and the agent's own output contract (§11.3) depend on it. |
| Node settings incl. inherited defaults shown with their source | MUST | Retry, timeout, runner, workspace decide whether unattended runs survive. |
| Dynamic choices via `PartCall` (Slack channels, Linear teams) | SHOULD | Avoids copy-pasting ids. |
| Pipeline panel: inputs, outputs, secrets (set/missing), budget presets, `allow_destructive` checklist, defaults, triggers, env | MUST | Budget and policy are mandatory for hands-off agent pipelines. |

### 1.8 Test while building

| Feature | Rank | Why |
|---|---|---|
| Dry run of the buffer (`R`), states painted on the graph | MUST | The only safe way to see a pipeline work before it runs for real. |
| Run one node against its input (`.`), with a cost confirm for agents | MUST | Prompt tuning without rerunning the whole pipeline. |
| Items panel for nodes and edges (`gi`) | MUST | "What did this produce?" is the first debugging question. |
| Pinned samples as files (`m`, `.mill/samples/`) | SHOULD | Repeatable previews and single-node runs; committable fixtures. |
| Cost/time estimates per node and per run, worst case vs budget (`M3208`) | SHOULD | Budgets are guesses without them. |

### 1.9 Safety net

| Feature | Rank | Why |
|---|---|---|
| Undo/redo, unlimited per session | MUST | Every editor needs it. |
| YAML pane synced both ways (`Y`), `$EDITOR` round trip (`E`) | MUST | The file stays the truth and the escape hatch. |
| Diff against disk (`D`) | SHOULD | Review before `:w`. |
| Conflict detection (`M2201`) with reload/diff/overwrite | MUST | CLI edits and git checkouts happen while the editor is open. |

## 2. Implementation phases (Go)

Each phase ends with something usable and tested; the CLI commands land with the operations they expose, before the TUI uses them.

| Phase | Builds | Done when |
|---|---|---|
| **E0 · edit core** | `internal/pipeline/edit`: load into a `yaml/v4` node tree (edits applied as text splices by `pipeline/yamledit`), a typed path API (get/set/unset/insert-after on mapping nodes, flow vs block preserved), edge inference shared with the compiler, operations (add, insert, branch, connect, disconnect, rename with reference rewrite, delete with heal, wrap/unwrap loop, extract, patterns), each returning a description for undo. Golden tests: input YAML + op → output YAML, byte for byte, comments kept. | `mill node/edge/pipeline/wrap/extract/pattern/refs` work from the CLI on every example pipeline with no comment or order loss. |
| **E1 · lint + fixes** | Lint rules as small functions over (document, derived graph, part manifests); fixes as edit operations flagged `safe` / `policy`; `mill lint --fix`, `--code/--node`; daemon `Lint` with `source` for machine checks. Explain pages for every new code. | Every code in §18.7.9 has a golden test and an explain page (§29.11 gates). |
| **E2 · layout + render** | `internal/pipeline/layout`: layering, slot assignment, loop containers, pins, orthogonal routing into a cell grid with junction characters; used by `mill render` first. | `mill render` draws every example deterministically at three zoom levels; snapshot tests. |
| **E3 · editor shell** | Bubble Tea model: canvas viewport over the cell grid (Charm `cellbuf` or a small own buffer), selection, `hjkl`/`w`/`b`/`/`, zoom, inspector (read-only), status line, help bar, undo stack, `:w` through `SavePipeline` with base hash, swap file. | Open, navigate and save any pipeline; `Y` shows the YAML. |
| **E4 · make + connect** | Palette (parts from `parts.load`, MCP tools, patterns), insert actions, connect mode with port picker and live check, edge focus, rename, yank/paste, visual mode, pins. | issue-to-pr can be rebuilt from an empty file with the keyboard. |
| **E5 · inspector editing** | Schema-driven forms, field editors, expression completion and preview (cel-go with a sample activation), prompt editor, output schema editor, pipeline panel, `PartCall` choices. | best-of-3 and nightly-sweep can be built without `E`. |
| **E6 · lint UX + test** | Badges, problems panel, `]e`, `ga`/`gA`; dry run via `StartRun{source, dry_run}` with live states; `RunNode`; items panel; samples; estimates via `Estimate`. | The v0.3 "done when" in §28 holds. |
| **E7 · polish** | Mouse (click, drag, port drag, pan), minimap, outline, what-if, folding, narrow-terminal rules, diff, conflict UI, sounds. | Usability pass with three people building a pipeline they haven't seen. |

The browser prototype already covers E3–E6 behavior (see section 4), so E3+ can be built against it as a reference.

## 3. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| YAML round trips aren't byte-perfect in any Go library (comment placement, flow style, quoting) | Editor edits produce noisy diffs; people stop trusting it | Edits are text splices (backend tech-stack §5): touch only the bytes they change, re-parse and compare; golden tests on real files; fall back to re-emitting a single node block, never the whole file. `goccy/go-yaml` stays the fallback if `yaml/v4` positions aren't enough. |
| Edge inference in the editor drifts from the compiler | The graph lies | One function in `internal/pipeline`, used by both; property test: compile(file) edges == editor edges. |
| Orthogonal routing gets unreadable past ~25 nodes | Big pipelines are unusable on cards | Compact/dots zoom, folding, extract to sub-pipeline; routing tests on generated graphs; "edges point right" rule. |
| Terminal width: cards need ~23 columns each | Few columns visible at 80–120 cols | Snap scrolling to whole columns, compact zoom by default under 80 cols, hide the right pane under 96. |
| Live preview evaluates expressions with real data | Secrets or large items leak into the screen | Secrets are opaque handles in the activation; previews are truncated; only pinned samples and recorded dry-run items are used, never live runs. |
| Quick fixes that widen policy slip into `gA` or `--fix` | A pipeline gains destructive permissions unnoticed | `policy` flag on fixes, enforced in the fix runner, with a test that `--fix` never changes `policy`. |
| Single-node runs of agents cost money | Surprise spend while tuning | Confirm with the estimate; respects the pipeline budget; `dry_run` for destructive parts. |
| Keymap collisions (vim habits, terminals eating `ctrl` keys) | Keys don't work or surprise | `keymap.bind` (§18.13) validates bindings; Lua overrides per screen; avoid `ctrl+h/j/i/m` which terminals map to other keys. |
| Scope creep toward a full IDE | v0.3 slips | MUST items only for v0.3; SHOULD items ordered by the phases above; LATER is explicitly out. |

## 4. What the prototype covers

`mockups/v2/index.html` implements, against made-up data: the document-as-buffer model with derived graph, layout, lint and inspector; the palette with categories, fuzzy search and patterns; insert after/before/branch/on edge; triggers; connect with port picker and live check; edge focus, disconnect, reroute; rename with reference rewrite; delete with heal; yank/paste; visual mode; wrap/unwrap loop; extract; pins (`H J K L`, drag); three zoom levels, fit, minimap, outline; lint with badges, panel, `]e`, `ga`/`gA`/`gx` and the safe/policy split; inspector forms with completion and preview (a small CEL subset evaluator), prompt editor, output schema editor, pipeline panel; what-if; dry run painted on the graph; single-node runs; items panel; samples; estimates; YAML pane with two-way editing and parse errors; diff; undo/redo; save rules; wizard output opening in the editor.

Specified but only stubbed or absent in the prototype: `PartCall` choices, the swap file and recovery, external-change conflicts, `$EDITOR` (`E`, `ctrl+e`), folding (`zo`/`zc` only report), fanout candidate expansion, saving patterns, `:prompt file`, align, daemon-side lint checks (secrets set, agents installed).

### Mockup ↔ spec coverage

The whole prototype, screen by screen, against `docs/spec/18-tui.md` (and the CLI parts it shows from `17-cli.md`, `29-errors.md`). **Covered** means the screen and every key and rule in its section behave as specified, against made-up data. **Partial** lists what's missing. Use the top bar to switch the terminal width (124, 100, 80) and the scenario (normal, fresh clone, config error, first run, boot fails).

| Screen | Spec | Status | Notes |
|---|---|---|---|
| Boot splash: cascade, statuses, slow step, ready line, hand-off, voice | §18.13 | covered | 22 steps, 70 ms cascade, 1.2 s minimum, `[  ..  ]` after 250 ms (scenario "fresh clone"), wide CJK cells, boot sounds from Appendix A. The browser needs one key press first for audio (prototype-only screen). |
| Boot failure: freeze, cause, fix, quit to stderr with exit code | §18.13 | covered | Scenario "boot fails": `q` drops to the shell with the §29.2 error and `echo $?` = 4. |
| First launch hand-off to setup | §18.13, §18.8 | covered | Scenario "first run". |
| First-run setup: 7 steps, ask once, refusals | §18.8 | covered | Sound step previews each set; trusted folders refuses `/`, home, relative and `..` (`M9301`). `--yes`/`--preview` are CLI-only and not mocked. |
| Board: header, sections, counts, fold, selection follows a pipeline | §18.2 | covered | `1 2 3`, `tab`, fold over 5 READY rows. |
| Board rows: five things, text per state, runs strip, age | §18.3 | covered | Strip uses `■ ✗ ↯ ⊗ ↻ ■` (spec v2.1). |
| Detailed view `v` | §18.3 | covered | runs (20), ok, typical, trigger, next, header `· N runs today · p50`. |
| Conditional columns: cost today, long-run badge, budget, scope | §18.3, §18.12 | covered | Budget `NN% $` appears once issue-to-pr passes 80% of $4.50; `▴2×` on the live run. The inverse `▴8×` past `max_wall` isn't reached by the demo data. |
| Narrow terminals: give-way order, name down to 12 | §18.3 | covered | Width button. |
| Filter row, `/` search tokens, `t`, `s` `-`, `V`, `p` | §18.2 | covered | Tokens `state: tag: trigger: scope: uses: ok: cost: ran:`. |
| `:` command line (`:run`, `:pipe`) | §18.2, §18.5 | covered | Replaced the board's bottom prompt in spec v2.2: `:run <pipeline> k=v` with a hint of missing inputs, `tab` completes names and input keys, `:pipe` runs pipe expressions; see [command-line.md](command-line.md). |
| Focus modes and the mode name | §18.4 | covered | Every mode works. The board shows the mode name at the left of the help bar; overlays show it in their status line (spec v2.1). |
| Board keys and mouse | §18.5 | covered | `ctrl+c` asks to quit like `q`. |
| Help overlay `?` | §18.5 | covered | Two columns at 96+, one column below with BADGES then MOUSE dropped and the truncation note. |
| Run popup: rows, timeline, title, budget amber, result line | §18.6 | covered | |
| Run popup: branches indented, `each` expands with `za` | §18.6 | covered | `└─ else` under `scope`; inbox-triage and multi-repo-refactor expand per item. |
| Run popup keys: `enter` agent or cause, `L`, `F`, `i`, `c` | §18.6, §29.8 | covered | `enter` on a failed, crashed or cancelled node shows the cause chain and the log around it. |
| Agent screen, view-only, `esc` `qq` `ctrl+]` | §18.6, §11.4 | covered | |
| Log in the popup: relative numbers, folds, search, follow, only-node, `gx` | §18.6 | covered | |
| `:runs` (older runs) | §18.6 | partial | Older runs open with generic "done" rows; their own data isn't modeled. |
| Editor | §18.7 | covered | See the section above for what's stubbed. Narrow rules from §18.7.2 included. |
| New pipeline wizard | §18.8 | covered | Review shows the YAML and lint of the document that will actually be created. |
| Plugins: views, columns, line under the list, detail page, keys | §18.9 | covered | Update with new permissions asks first; updates of a plugin a running pipeline uses queue until it finishes. |
| Settings: sections with relative numbers, rows, bottom help line, status line, `h l enter i p u gd /` | §18.10 | covered | Registry from §19.6. `theme` and `density` change the value but the prototype only draws dark/normal. |
| Sound: soundsets, events, groups, throttles, `when`, quiet hours, voice | §18.11, App. A | covered | `unfocused` uses the page's focus. |
| Screensaver: one-line logo, live counts, safe wake, drift once a minute | §18.12 | covered | `keep_awake` (caffeinate) can't be mocked. |
| `:stats` | §18.12 | covered | 1d / 7d / 30d runs and cost, p50, p90, totals. |
| `:bg` / `:detach` | §18.12 | partial | Drops to the shell; "the next `mill` restores them once" isn't modeled. |
| `:messages` | §29.8 | covered | Boot warnings and every error this session. |
| Config error line under the header | §29.8, §18.13 | covered | Scenario "config error". |
| Shell: `mill status`, `ls`, `runs`, `run`, `doctor`, `trust .`, `explain`, prompt segment | §17, §29.10, §22 | partial | Synthetic output for the commands the TUI points to; not the full CLI. |
| Backend-plan states on screen | §4, §18.2, §18.3, §18.6 | covered | `↯ crashed` (S18), untrusted projects with `mill trust .` (D18), retries as new attempts `try 1`/`try 2` (S8), `⊗ cancelled · budget` with the reason (S9), `M3021` ambiguous primary upstream (D9), `•••` redaction in logs and items (S1). Part of the spec since v2.1; the page footer still calls them provisional. Not yet in the prototype: `M3201` as a warning on manual-only pipelines (it still shows an error) and `M9208` for samples. |

## 5. Decisions

Each has the default this spec and prototype use, and the alternatives. **All 30 were decided on 2026-11-11** and are applied in spec §18.7 (v2.1): the user decided 1, 4, 7, 8 and 12; the rest take the default. Only 7 differs from its default. Recorded in `docs/agents/DECISIONS.md` (ADR-028, ADR-032 to ADR-035, ADR-044).

1. **Inferred edges when a node has no `in:` and references several nodes.** The backend plan settles this (D9) and the prototype now follows it: the input is the **primary upstream**, the referenced node downstream of all the others; when the references run in parallel there is none, and lint reports `M3021` with `in:` fixes. Alternatives: items from *every* referenced node; or require an explicit `in:` whenever a node references more than one node. **Decided (user):** primary upstream + `M3021` on ambiguity. Applied: §7.4, §18.7.3 (Inferred, Ordering), §18.7.9 (`M3021` ambiguous input; unreachable moved to `M3024`).
2. **References when `in:` is explicit.** Default: references to nodes not already upstream add a dotted ordering edge, no items. Alternatives: they add a data edge too (§7.4 read literally); or they're a lint error until `needs:` is written. **Decided:** default accepted.
3. **Connecting into a node whose input is inferred.** Default: write `in: [<inferred sources>, <new>]` so nothing is lost. Alternatives: write only the new source (simpler YAML, may change behavior); ask each time. **Decided:** default accepted.
4. **Loop YAML shape.** Default: `loop` with `with.body: [ids]`, `until`, `max_iters`; body entry reads `<loop>.body`; ports `out` and `exhausted`. Alternatives: nested `nodes:` inside the loop node (self-contained, harder to edit with `mill node`); a `loop:` key on each body node. **Decided (user):** the default, `with.body` list + `until` + `max_iters`, ports `out` and `exhausted` (§10.3).
5. **`exhausted` port on loops and `agent.loop`.** Default: yes, carrying the last try. Alternatives: no port; an exhausted loop fails the node (`on_error` then decides). **Decided:** default accepted.
6. **Intentional drops.** Default: a new node key `discard: [port, ...]`. Alternatives: connect to a `noop` named `drop-…`; a YAML comment directive (`# mill: discard false`); no warning at all for unconnected branch ports. **Decided:** default accepted.
7. **Missing budget on agent pipelines (`M3201`).** Default: error. Alternatives: warning; error only for pipelines with non-manual triggers. **Decided (user):** error when the pipeline has any automatic trigger, warning when it only runs by hand. This differs from the default above. Applied: §18.7.9 `M3201` is `error · warning`, and the best-of-3 drawings (manual) now show it as a warning (`✗ 2  ! 2`).
8. **Saving with lint errors.** Default: `:w` refuses, `:w!` saves and the pipeline stays disarmed until clean. Alternatives: always save, show errors; never save errors. **Decided (user):** the default. `:w` refuses, `:w!` saves and the pipeline stays disarmed (§18.7.9).
9. **Policy-widening fixes.** Default: never in `gA` / `--fix`; one at a time with `ga` and a visible "policy change" note. Alternatives: allowed in `gA` with a confirmation listing them; never offered as fixes at all (edit the pipeline panel instead). **Decided:** default accepted.
10. **Deleting a node in a chain.** Default: heal automatically when it has exactly one input and one output. Alternatives: never heal (lint shows the gap); always ask. **Decided:** default accepted.
11. **Layout pins.** Default: pins store slots (column, row) and edges must point right (moves that break this are refused). Alternatives: free cell positions; allow backward placement with routed back-edges. **Decided:** default accepted.
12. **Pinned samples location.** Default: `.mill/samples/<pipeline>/<node>.jsonl`, committed like code. Alternatives: inside the layout sidecar; user-only (`~/.local/share/mill/samples`), never committed. **Decided (user):** the default, committed; lint warns when a sample looks like it holds a secret (`M9208`, §18.7.9, §18.7.12).
13. **Dry run of unsaved changes.** Default: `R` runs the buffer (`StartRun{source}`) and records it as a normal dry run. Alternatives: require `:w` first; run the buffer but don't keep the run in history. **Decided:** default accepted.
14. **Single-node runs of agents.** Default: confirm with the estimate every time. Alternatives: confirm only above a cost threshold (a setting); never confirm in the editor. **Decided:** default accepted.
15. **Right pane default.** Default: inspector always visible beside the canvas (35 columns). Alternatives: hidden until `enter` (more canvas); a bottom pane instead of a right one. **Decided:** default accepted.
16. **Default zoom.** Default: cards; compact automatically under 80 columns. Alternatives: compact by default; remember per pipeline in `tui-state.json`. **Decided:** default accepted.
17. **Help bar contents.** Default: five keys that change with mode and focus (rule 5 kept). Alternatives: a fixed five for the whole editor; six keys in the editor. **Decided:** default accepted.
18. **Leaving the editor.** Default: `esc` unwinds one layer at a time, then asks to save. Alternatives: `esc` from anywhere leaves (with the save prompt); `:q` is the only way out. **Decided:** default accepted.
19. **Palette key layout.** Default: `a` after, `i` before, `o` branch, `gt` trigger. Alternatives: only `a`, with the insert position chosen inside the palette; `A`/`I` capitals for before/after like vim. **Decided:** default accepted.
20. **Key collisions with vim habits.** Default: `x` disconnects, `dd` deletes, `r` renames, `u` undoes, `space` opens the action menu. Alternatives: `x` deletes the node (vim `x`), `C` reroutes; `space` as a leader key for multi-key commands. **Decided:** default accepted.
21. **Colors in the editor.** Default: color only for lint state, dry-run state and diff lines; selection is a background; expressions use dim delimiters, not color. Alternatives: a distinct accent for expressions and ports (bends rule 8 further); monochrome editor with glyph-only states. **Decided:** default accepted.
22. **Expression preview data.** Default: pinned sample first, then the last dry run's recorded items; nothing live. Alternatives: also the latest real run's items (more realistic, risk of sensitive data); synthetic data generated from schemas only. **Decided:** default accepted.
23. **Expression preview engine.** Default: the real `cel-go` environment with a sample activation (the prototype uses a small subset evaluator). Alternatives: daemon-side evaluation via a new intent; preview off by default. **Decided:** default accepted.
24. **Hands-off enforcement for plugins.** Default: a manifest flag `interactive`; such parts are listed greyed and refused by lint (`M3206`). Alternatives: hide them entirely; allow them only inside `handoff.to`. **Decided:** default accepted.
25. **Patterns format and source.** Default: patterns are partial pipeline YAML files with `‹placeholders›`; built-ins embedded, plugins via `provides.patterns`, user ones in `~/.config/mill/patterns/`. Alternatives: Lua functions that build nodes; only built-in patterns in v0.3. **Decided:** default accepted.
26. **Extract to sub-pipeline: inputs.** Default: each outside node referenced becomes an input named after it (`inputs.<node>`), and the parent passes `nodes.<node>.item`. Alternatives: one `items` input carrying the stream only; ask for names in a dialog. **Decided:** default accepted.
27. **Where new nodes go in the YAML.** Default: right after the node they're inserted after (file order follows the flow). Alternatives: always at the end of `nodes:`; sorted topologically on every save. **Decided:** default accepted.
28. **Swap files.** Default: write the buffer to `~/.local/state/mill/swap/` after each change; offer recovery on next open. Alternatives: no swap (undo only); autosave to the real file with a dirty-marker. **Decided:** default accepted.
29. **Estimates.** Default: p50 of the node's history, else model price × a typical token count per part; shown as a range (one try to `max_iters`). Alternatives: history only (blank until a run exists); a per-part `estimate` field in manifests. **Decided:** default accepted.
30. **CLI command names.** Default: `mill node|edge|pipeline|wrap|unwrap|extract|pattern|refs|layout|estimate`. Alternatives: everything under `mill edit <verb>`; fewer top-level verbs (`wrap`/`extract` as `mill node wrap|extract`). **Decided:** default accepted.
