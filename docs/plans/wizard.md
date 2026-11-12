# Plan: the new-pipeline wizard (v2)

Status: **decided and merged into spec v2.2** (§18.8, §17, §19.6, §29) on 2026-11-12. The spec wins; sections 13–15 are kept as history. Decisions: ADR-045 to ADR-050.
Spec: §18.8 (wizards), §18.1 (design rules), §18.4 (focus modes), §18.7.4 (modal model), §18.7.6 (patterns), §17.1 and §17.8 (`mill new`), §9.9–9.10 (export, import), §23.3 (n8n), §29 (errors).
Prototype: `mockups/v2/index.html`: press `n` on the board, or the **Import file…** button, or drop a file on the terminal. `?` inside the wizard lists every key.
Companion plan: [command-line.md](command-line.md) (the `:` line the wizard shares with every screen).

The wizard is the first thing a new user builds with. Spec v2.1 has five steps that each ask one or two questions, a name before anything else, keys that don't match the editor, and an import that only takes n8n or a `.millpkg` path. This plan replaces it with four steps that use the editor's modal keys (§18.7.4), open from a sentence, a template, a copy or a file, and preview the graph, the YAML, lint and the estimate before anything is written.

Ranking, as in [editor.md](editor.md): **MUST** (v0.3 can't ship a usable wizard without it), **SHOULD** (fast and safe; v0.3 is weaker without it), **LATER**.

## 1. Problems with the v2.1 wizard

| # | Problem | Where | Effect |
|---|---|---|---|
| 1 | Five steps, three of them holding one value each (trigger, agent, part of limits) | §18.8 table | Ten key presses before you see what you're building |
| 2 | The name comes first | step 1 | You name a thing before you know what it is; `M2205` blocks you on step 1 |
| 3 | The graph and YAML only show on step 5 | step 5 | The agent's proposal is a checklist with no picture until the end |
| 4 | Keys differ from the editor: `backspace` goes back, text fields take keys in place, no `INSERT` | §18.8 "Keys" | Typing `j` in a name moves the cursor; editor habits don't carry over |
| 5 | "Limits" mixes five decisions (budget, max agents, destructive allow list, failure target, dry run first) | step 4 | The densest screen asks the questions people least want to answer |
| 6 | Import takes a path to an n8n file or a `.millpkg` only: no picker, no upload, no URL, no stdin, no preview | step 1 | The one fast path for n8n users is hidden and blind |
| 7 | No duplicate, no gallery, no patterns as starters | step 1 | "Like that one, but nightly" means copying files by hand |
| 8 | Lint and the estimate are only shown after create | step 5 / editor | A budget that the estimate already exceeds is found in the editor |
| 9 | `mill new` flags don't cover import sources (`--from file.json` only) or stdin | §17.1 | No CLI parity for upload, URL or pipe |
| 10 | No error codes for import problems | §29.4 | "Didn't work" has no `mill explain` page |

## 2. The new steps

`n`, `:new` or `mill new` opens a square overlay with four steps and a breadcrumb: `1 start › 2 shape › 3 run › 4 review`. One decision per region, a default on every field, and `enter` accepts the defaults all the way through: from a fresh `n`, six `enter`s draft the example sentence, walk the steps and create it.

| Step | One decision | Regions | Why this way |
|---|---|---|---|
| **1 start** | Where it comes from | Left: five sources (describe it, template, duplicate, import a file, blank). Right: that source's pane (sentence, gallery, list, path + preview, nothing) | Every later step depends on the source, so it comes first. Five sources fit one list; the pane changes, the list doesn't |
| **2 shape** | What it does | The steps (checklist for a drafted pipeline, node list for a copy or import), the agent and effort, a `refine` sentence, and a live graph/YAML preview (`Y`) | The proposal is judged by looking at it. The agent belongs with the steps it drives |
| **3 run** | When it runs and how far it may go | Trigger (+ its one setting), budget preset, on failure; `▸ advanced` folds max agents, the destructive allow list, scope | Everything that matters to an unattended run, in one screen. Rarely changed values are folded |
| **4 review** | Write it | Name (suggested from the source, editable), files it writes, a summary, the estimate against the budget, live lint, and `then ◂ dry run, then edit ▸` | The name is chosen when you know what it is. Nothing is written before `enter` here |

Defaults: source `describe it`, agent from the registry's first signed-in CLI (`claude/sonnet`, effort `medium`), trigger from the sentence or the source (`manual` otherwise), budget `medium`, on failure `GitHub issue` when `mill/github` is installed (`nothing` otherwise), scope `project`, then `dry run, then edit`.

Hands-off (§2 principle 1, §18.8): no step asks the *run* to wait. The wizard sets limits instead of approvals, an imported n8n step that waited for a person becomes a `handoff` that ends its branch (`M2307`), and `dry run first` is a dry run, not an approval gate.

§18.1 rules kept: one square overlay (rule 10), five keys in each help bar (rule 5), color only for state (✓ ✗ !), `esc` returns to the board with nothing written (rule 1), and every step has a `mill new` flag (rule 12).

## 3. Screens

100-column terminal (the overlay is 92 wide); the dimmed board behind it is cut away. The last two rows are the terminal's status line and help bar. Captured from the prototype.

### 3.1 start: describe it (NORMAL, then INSERT)

```
┌ ≋ new pipeline ──────────────────────────────────────────────────────────────────────────┐
│  1 start  ›  2 shape  ›  3 run  ›  4 review                                              │
│──────────────────────────────────────────────────────────────────────────────────────────│
│                                                                                          │
│  How do you want to start?                                                               │
│                                                                                          │
│  ✎ describe it            │ Say what it should do. An agent drafts the trigger and       │
│                           │ steps; you check them on the next screen. Nothing is         │
│  ⊞ template               │ written until you create it.                                 │
│                           │                                                              │
│  ⧉ duplicate              │ in a sentence                                                │
│                           │ every night find flaky tests, fix them, run the tests an…    │
│  ⇪ import a file          │                                                              │
│                           │ drafted by claude/haiku · headless · ≈ $0.01 · counts towa…  │
│  □ blank                  │                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  1/4 start   say what it should do · an ag…           nothing is written until create
 j/k choose · l open · i describe · enter next · ? more
```

`i` edits the sentence; the mode changes on the status line and the help bar follows it:

```
│  ⧉ duplicate              │ in a sentence                                                │
│                           │ every night find flaky tests, fix them and open a draft      │
│  ⇪ import a file          │ PR▌                                                          │
 NEW · INSERT  1/4 start   say what it should do · an ag…           nothing is written until create
 type edit · tab complete · enter draft it · esc normal · ctrl+c discard
```

### 3.2 start: template gallery (hjkl over a two-column grid)

```
│  ✎ describe it            │ 10 starters · / filters                                      │
│                           │                                                              │
│  ⊞ template               │ ⇡ issue-to-pr                 ◷ nightly-sweep                │
│                           │   labeled issue → plan → bu…    nightly fixes, tested, as…   │
│  ⧉ duplicate              │                                                              │
│                           │ ♪ morning-brief               # standup-dige…   mill/slack   │
│  ⇪ import a file          │   weekday digest of feeds, …    yesterday's merged PRs, s…   │
│                           │                                                              │
│  □ blank                  │ ⁂ plan → check → …   pattern  ⁂ best of N          pattern   │
│                           │   plan, assert on risk and …    fanout × 3 → check each →…   │
│                           │                                                              │
│                           │ ⁂ fix until green    pattern  ⁂ review → PR        pattern   │
│                           │   loop: agent → test, until…    independent review, appro…   │
│                           │                                                              │
│                           │ ┌ issue-to-pr · Y yaml ───────────────────────────────────┐  │
│                           │ │ ⌁ on.webhook /github                                    │  │
│                           │ │ └─ ⑂ ws git.worktree · issue-to-pr-…                    │  │
│                           │ │    └─ ◆ work agent · opus · high                        │  │
│                           │ └──────────────────────────────────────────────── +5 ─────┘  │
 NEW · NORMAL  1/4 start   built-in, plugin and pattern …           nothing is written until create
 hjkl choose · / filter · enter use it · esc cancel · ? more
```

### 3.3 start: duplicate

```
│  ✎ describe it            │ copies the file and the prompts it uses · the copy's trigge… │
│                           │                                                              │
│  ⊞ template               │ nightly-sweep         cron 03:00        Fix new vulnerabil…  │
│                           │ pr-fixer              webhook /review   Fix review comment…  │
│  ⧉ duplicate              │ multi-repo-refactor   manual            One refactor acros…  │
│                           │ issue-to-pr           webhook /github   Turn labeled issue…  │
│  ⇪ import a file          │ inbox-triage          every 15m         Sort new mail and …  │
 NEW · NORMAL  1/4 start   copy a pipeline you already have         nothing is written until create
 j/k choose · / filter · enter copy it · h sources · ? more
```

### 3.4 start: import a file (preview, file picker, error)

```
│  ✎ describe it            │ A file, a URL, gh:owner/repo/path, or a pipe expression. D…  │
│                           │                                                              │
│  ⊞ template               │ path  ~/Downloads/slack-digest.json                          │
│                           │       o browse…                                              │
│  ⧉ duplicate              │                                                              │
│                           │ ✓ n8n workflow · 6 nodes                                     │
│  ⇪ import a file          │ creates   .mill/pipelines/slack-digest.mill.yaml · scripts…  │
│                           │ mapped    4: Every weekday → on.cron · Fetch feed → http ·…  │
│  □ blank                  │ stubs     ! M2304 2 n8n nodes run on n8n through n8n.call …  │
│                           │ lint      ✗ 1 M3003 has-items.with.cond: ‹condition from n…  │
│                           │ trust     new file: creating it trusts this version (§9.10)  │
│                           │                                                              │
│  n8n workflow · nothing written yet · enter uses it                                      │
└──────────────────────────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  1/4 start   file, URL, gh:owner/repo/path…           nothing is written until create
 i path · o browse · enter use it · h sources · ? more
```

`o` opens the file picker over the pane (`hjkl` browse, `l`/`enter` open, `h` up a folder, `esc` close); folders first, each file tagged with what it looks like:

```
│               ┌ open ~/Downloads/ ───────────────────────────────────────┐               │
│  ✎ describe it│ ↰ ..                                                     │xpression. D…  │
│               │ ▤ broken.json                                       json │               │
│  ⊞ template   │ ▤ code-kit.millpkg                                bundle │               │
│               │ ▤ huge-export.json                                  json │               │
│  ⧉ duplicate  │ ▤ notes.txt                                              │               │
│               │ ▤ slack-digest.json                                 json │               │
│  ⇪ import a fi│                                                          │ml · scripts…  │
│               │ j/k move · l/enter open · h up a folder · esc close      │eed → http ·…  │
│  □ blank      └──────────────────────────────────────────────────────────┘gh n8n.call …  │
 NEW · NORMAL  1/4 start   file, URL, gh:owner/repo/path…           nothing is written until create
 hjkl browse · enter choose · h up a folder · esc close
```

An unknown file is refused with a code, inline and on the status line; nothing is written:

```
│  ⊞ template               │ path  ~/Downloads/notes.txt                                  │
│                           │       o browse…                                              │
│  ⧉ duplicate              │                                                              │
│                           │ ✗ M2301 notes.txt isn't an n8n workflow, a .millpkg, a mil…  │
│  ⇪ import a file          │   gx explains · nothing was written                          │
│                           │                                                              │
│  error[M2301]: notes.txt isn't an n8n workflow, a .millpkg, a mill pipeline …  gx        │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.5 shape

```
┌ ≋ new pipeline · find-flaky-tests ───────────────────────────────────────────────────────┐
│  ✓ start  ›  2 shape  ›  3 run  ›  4 review                  drafted from your sentence  │
│──────────────────────────────────────────────────────────────────────────────────────────│
│                                                                                          │
│  What it does · space turns a step on or off                                             │
│                                                ┌ graph · Y yaml ───────────────────────┐ │
│  agent   ◂ claude/sonnet ▸         effort me…  │ ⌁ on.cron 0 3 * * *                   │ │
│                                                │ └─ ⑂ ws git.worktree · find-flaky-t…  │ │
│  [ ] ⇄ fetch   fetch data from URLs or APIs    │    └─ ◆ work agent · sonnet · medium  │ │
│  [✓] ⑂ ws      changes code → its own branc…   │       └─ ⊨ check check · make check   │ │
│  [✓] ◆ work    does the task you described     │          ├─ true ⇡ pr gh.pr · draft…  │ │
│  [ ] ⊢ scope   check the agent's plan befor…   │          └─ false ↪ hand-off handof…  │ │
│  [✓] $ check   “tests” → run make check; fa…   │                                       │ │
│  [ ] ◇ review  independent review by a seco…   │                                       │ │
│  [✓] ⇡ pr      “draft pr” → open a draft PR    │                                       │ │
│  [ ] # notify  post the result to Slack        │                                       │ │
│                                                │                                       │ │
│  refine  also post the result to #dev          │                                       │ │
│                                                │                                       │ │
│  trigger “night” → every day at 03:00 (next…   └───────────────────────────────────────┘ │
│                                                                                          │
│  claude/haiku proposed 4 steps and a trigger · ≈ $0.01 · nothing written                 │
└──────────────────────────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  2/4 shape   5 steps · drafted from your s…           nothing is written until create
 j/k move · space on/off · h/l agent · enter next · ? more
```

Each proposed step says which words caused it (`“tests” → run make check`), so a wrong guess is visible and one `space` away.

### 3.6 run

```
┌ ≋ new pipeline · find-flaky-tests ───────────────────────────────────────────────────────┐
│  ✓ start  ›  ✓ shape  ›  3 run  ›  4 review                  drafted from your sentence  │
│──────────────────────────────────────────────────────────────────────────────────────────│
│                                                                                          │
│  When it runs, and its limits                                the run never waits on you  │
│                                                                                          │
│  trigger         ◂ schedule ▸                                                            │
│    schedule      0 3 * * *         every day at 03:00                                    │
│  budget          ◂ medium ▸        2M tk · $15 · 1h · ≈ $0.51 per run fits               │
│  on failure      ◂ GitHub issue ▸  opens an issue with the error and logs                │
│                                                                                          │
│  ▸ advanced      max agents 2 · allows gh.pr, gh.issue · project                         │
│                                                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  3/4 run   the run never waits on you, s…             nothing is written until create
 j/k field · h/l change · i edit · enter next · ? more
```

`za` or `enter` on `▸ advanced` unfolds max agents, one `allow` row per destructive part the pipeline uses, and scope (project / user).

### 3.7 review

```
┌ ≋ new pipeline · find-flaky-tests ───────────────────────────────────────────────────────┐
│  ✓ start  ›  ✓ shape  ›  ✓ run  ›  4 review                  drafted from your sentence  │
│──────────────────────────────────────────────────────────────────────────────────────────│
│                                                                                          │
│  Review · nothing is written until enter                                                 │
│                                                ┌ graph · Y yaml ───────────────────────┐ │
│  name      find-flaky-tests                    │ ⌁ on.cron 0 3 * * *                   │ │
│            ✓ free · lowercase, digits, dash…   │ └─ ⑂ ws git.worktree · find-flaky-t…  │ │
│                                                │    └─ ◆ work agent · sonnet · medium  │ │
│  writes    .mill/pipelines/find-flaky-tests…   │       └─ ⊨ check check · make check   │ │
│            prompts/find-flaky-tests.md         │          ├─ true ⇡ pr gh.pr · draft…  │ │
│  runs      every day at 03:00 · 5 steps · c…   │          └─ false ↪ hand-off handof…  │ │
│  limits    2M tk · $15 · 1h · on failure Gi…   │                                       │ │
│  allows    gh.pr, gh.issue                     │                                       │ │
│  estimate  ≈ $0.51 · 4m48s per run · fits t…   │                                       │ │
│  lint      ✓ clean                             │                                       │ │
│                                                │                                       │ │
│  then      ◂ dry run, then edit ▸  destruct…   │                                       │ │
│                                                └───────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  4/4 review   .mill/pipelines/find-flaky-te…          nothing is written until create
 i rename · h/l then · Y yaml · enter create · ? more
```

### 3.8 80 columns

The preview pane drops below 90 columns (`Y` toggles a full-width preview instead); the source list narrows and the pane text wraps.

```
┌ ≋ new pipeline ──────────────────────────────────────────────────────┐
│  1 start  ›  2 shape  ›  3 run  ›  4 review                          │
│──────────────────────────────────────────────────────────────────────│
│                                                                      │
│  How do you want to start?                                           │
│                                                                      │
│  ✎ describe it        │ Say what it should do. An agent drafts the   │
│                       │ trigger and steps; you check them on the     │
│  ⊞ template           │ next screen. Nothing is written until you    │
│                       │ create it.                                   │
│  ⧉ duplicate          │                                              │
│                       │ in a sentence                                │
│  ⇪ import a file      │ every night find flaky tests, fix them, …    │
│                       │                                              │
│  □ blank              │ drafted by claude/haiku · headless · ≈ $0.…  │
└──────────────────────────────────────────────────────────────────────┘
 NEW · NORMAL  1/4 start   say what it should do · an agent drafts the s…
 j/k choose · l open · i describe · enter next · ? more
```

## 4. Keys

The editor's modal model (§18.7.4): `NORMAL` moves and acts, `i` enters `INSERT` on a text field, `esc` goes back. The mode shows at the left of the status line: `NEW · NORMAL`, `NEW · INSERT`.

**NORMAL**

| Key | Action |
|---|---|
| `j` `k` `↓` `↑` | Next / previous row (source, step, field) |
| `h` `l` `←` `→` | Change a `◂ value ▸`; on start, `l` opens the source's pane and `h` goes back to the list; in the gallery and picker, move left / right |
| `g` `G` | First / last row |
| `i` `a` `enter` (on a text field) | Edit it (`INSERT`) |
| `cc` | Clear the field and edit it |
| `space` | Turn a step or a switch on / off |
| `enter` | Use the selection and go to the next step; on review, create |
| `tab` `shift+tab` | Next / previous step |
| `1` `2` `3` `4` | Jump to a step (the breadcrumb) |
| `za` | Fold / unfold `▸ advanced` |
| `Y` | Graph ↔ YAML preview (below 90 columns: show / hide the preview) |
| `/` | Filter the gallery or the duplicate list (types on the bottom row) |
| `o` | File picker (import) |
| `u` | Upload (browser prototype only: the browser's file dialog) |
| `gx` | Explain the error code on screen |
| `:` | Command line (§ command-line.md): `:w`/`:create` creates, `:q` cancels |
| `?` | Wizard keys |
| `esc` | Leave the pane, then cancel the wizard; nothing is written (`n` resumes the draft, `:new!` starts over) |

**INSERT** (sentence, refine, path, name, trigger setting, filters)

| Key | Action |
|---|---|
| type, `backspace` | Edit |
| `ctrl+w` `ctrl+u` | Delete a word / clear |
| `tab` | Complete: a path (folders end in `/`), a pipeline name (after a pipeline), a cron preset (`0 3 * * *` → `30 7 * * 1-5` → …) |
| `enter` | Done, and act: the sentence drafts, `refine` redrafts, a path loads, a name is checked |
| `esc` | Back to NORMAL, keep the text |
| `ctrl+c` | Back to NORMAL, drop the edit |
| `:` | Types a colon (INSERT never opens the command line) |

**Picker** (over the import pane): `j`/`k` move, `l`/`enter` open a folder or choose a file, `h`/`backspace` up a folder, `g`/`G` first / last, `esc`/`q` close. **Drafting** (an agent is drafting): `esc` or `ctrl+c` stops it; `:` still opens the command line.

## 5. Import and upload

### 5.1 Sources

| Source | TUI | CLI | Prototype |
|---|---|---|---|
| A local file | Path field (`i`) with `tab` completion; `o` file picker; a path pasted into the wizard (what a terminal does when you drop a file on it) | `mill new --from ./flow.json`, `mill import ./flow.json` | Same, over a made-up file system; plus a real `<input type=file>` (`u`, **Import file…**) and drag and drop onto the terminal |
| stdin | — (a TUI has no stdin) | `mill new --from - < flow.json`, `pbpaste \| mill new --from -` | `-` in the path field explains `M2308` |
| A URL | Path field | `mill new --from https://…/kit.millpkg` | Simulated: `.millpkg` URLs load the sample kit, others `M2303` |
| `gh:owner/repo[/path][@ref]` | Path field | `mill new --from gh:acme/mill-pipelines/release-notes` | Simulated with the sample kit |
| A pipe expression | Path field (anything with `\|` and a quoted stage) | `mill new --from 'sh "git log -5" \| agent claude/haiku "summarize"'` | Same |

Paste handling: the wizard strips the quotes and backslash escapes terminals add to dropped paths (`'/Users/a/My Flows/x.json'`, `/Users/a/My\ Flows/x.json`) and, when the paste looks like a path, URL or pipe, switches to *import a file* and loads it. In INSERT, a paste is plain text.

milld reads and fetches imports (principle 4: the client renders, the daemon owns disk and network), so a remote TUI imports from the daemon's machine. Fetches follow the same allowlist and size cap as plugin installs.

### 5.2 Type detection

Detection runs on content, then the name; it never writes anything.

| Order | Test | Kind | Result |
|---|---|---|---|
| 1 | > 5 MB | — | `M2306` |
| 2 | starts `PK` (zip) | — | `M2305` (a `.millpkg` is a tar.gz) |
| 3 | gzip magic, then tar `ustar` | `.millpkg` | `plugin.lua` + `pipelines/*.mill.yaml` → bundle preview; missing either → `M2305` |
| 4 | control bytes in the first 4 KB | — | `M2306` (binary) |
| 5 | JSON with `nodes[]` + `connections{}` (or n8n's export array of one) | n8n workflow | Map nodes, stub the rest |
| 6 | JSON or YAML with `mill:` | mill pipeline | Open as-is; `mill:` newer than this build → `M2302` |
| 7 | JSON that's neither | — | `M2301` |
| 8 | Bad JSON / YAML | — | `M2101` with `file:line` |
| 9 | One to four lines, first stage a part id, `\|` between stages | pipe expression | One node per stage |
| 10 | Anything else | — | `M2301` |

### 5.3 What each kind becomes

- **n8n workflow** (§23.3): Webhook, Schedule/Cron, Manual → `on:`; HTTP Request → `http`; IF → `if`; Switch → `switch`; Set → `map`; Code/Function → `exec` with the code moved to `scripts/<pipeline>-<node>.js`; Merge → `merge`; Wait → `wait`; Execute Workflow → `pipeline`. Everything else becomes an `n8n.call` stub that runs that node on your n8n through a webhook (`M2304`, a warning that lists the stubs and needs `N8N_URL`). A node that waits for a person (`…SendAndWait`, Form, approval) becomes a `handoff` that ends its branch (`M2307`). Sticky notes are dropped. Required settings the mapping can't fill get `‹placeholders›`, which lint shows (`M3003`).
- **mill pipeline**: opened as-is; the name is made unique (`hello` → `hello-2`) when taken. Its trigger, budget and agents are offered as `as in the file` on steps 2–3.
- **`.millpkg`**: the §9.10 summary (pipelines, plugins it needs with installed / will install, agents, secrets set / missing `M9202`, permissions, trust), and a choice of which pipeline to open when it has several. Creating it writes every pipeline and its prompts; plugins it needs are installed on create, through the normal plugin flow (§9.6).
- **pipe expression**: one node per stage, connected in order, trigger `manual`.

Imported files are new files: creating one trusts that version (§9.10, §22), and the preview says so.

### 5.4 Preview before writing

The pane shows, for every kind: what it is (`✓ n8n workflow · 6 nodes`), the files it creates, what was mapped and what was stubbed, lint (live, same engine as the editor), and trust. `enter` uses it and moves to *shape*; nothing is written until *review*. An error shows inline (`✗ M2301 …`, `gx explains`) and on the status line, and also goes to `:messages`.

### 5.5 Error codes (new, `M23xx` = import; checked free in §29.4 and §18.7.9)

| Code | Class | Message | Help |
|---|---|---|---|
| `M2301` | user | `<file> isn't an n8n workflow, a .millpkg, a mill pipeline or a pipe expression` | `mill explain M2301` lists what's accepted |
| `M2302` | user | `<file> is mill: <n>; this mill reads mill: 1` | update mill |
| `M2303` | env | `no such file` / `couldn't fetch <url>: <status>` / `<path> is a folder` | check the path; `o` browses |
| `M2304` | warning | `<n> n8n nodes run on n8n through n8n.call stubs` | set `N8N_URL`, or replace the stubs in the editor |
| `M2305` | user | `<file> is a zip` / `has no plugin.lua` / `has no pipelines/*.mill.yaml` | `mill export` makes a valid `.millpkg` |
| `M2306` | user | `<file> is <n> MB · imports stop at 5 MB` / `looks binary` | — |
| `M2307` | warning | `<node> waited for a person: now a handoff that ends its branch` | hands-off (§2) |
| `M2308` | user | `- reads stdin, which only works in a shell` | `mill new --from - < file` |
| `M2309` | user | `no template or pattern "<id>"` | `mill pattern ls` |
| `M2211` | user | `turn on at least one step` (a drafted pipeline with every step off) | — |
| `M3050` | lint | `"<v>" isn't a schedule` / `webhook path must start with /` | 5 fields or `@every 10m` |

Existing codes reused with their meaning: `M1210` bad name, `M2205` name taken, `M2101` parse error, `M1201` pipeline not found (duplicate, `after`), `M6101` agent CLI missing, `M9202` secret missing.

## 6. Describe

1. `i` on *describe it*, type a sentence, `enter`. The drafter (a local model through Ollama by default; `drafter.*` settings, §19.6; keyword rules when it's unavailable) drafts a proposal: steps from the step catalog (fetch, ws, work, scope, check, review, pr, notify), a trigger, an agent and effort, a description and a name. Each choice carries the words that caused it.
2. The draft is cheap and visible: `≈ $0.01`, counted in today's spend; it runs as a normal one-node run (`mill runs` shows it), so a failure is an error with a code, not a hang. `esc` stops it.
3. *shape* shows the result with a live graph. `space` turns steps on and off, `h`/`l` changes the agent, `refine` (`i` on the last row) redrafts with an extra sentence (`also post the result to #dev` adds `notify`). Words like `without review` remove a step.
4. Placeholders the agent couldn't fill (a channel, a repo) stay `‹…›` and show in lint on *review*; creating with them is allowed and the editor walks them (`]e`).

Offline or with no agent signed in, *describe it* says so on the pane and the keyword drafter (the prototype's) runs locally: same steps, no cost line. `mill new --describe "…"` prints the same proposal as YAML.

## 7. Templates and duplicate

- **Gallery**: built-in templates (`issue-to-pr`, `nightly-sweep`, `morning-brief`), templates shipped by installed plugins (tagged with the plugin, e.g. `standup-digest · mill/slack`), and the editor's patterns (§18.7.6) as starters (`plan → check → handoff`, `best of N`, `fix until green`, `review → PR`, …). Two columns at 90+ columns, one below; `hjkl` moves, `/` filters (on the bottom row), the selected entry previews below the grid (graph or YAML with `Y`).
- **Duplicate**: every pipeline, with its trigger and description; `/` filters. The copy gets `<name>-copy`, its prompts are copied, and its triggers start as `manual` (step 3 offers `as in <original>`), so a copy never fires twice by accident.

## 8. Review and create

- Live lint (the editor's engine, §18.7.9) and the estimate (§18.7.12) against the chosen budget: `≈ $0.51 · 4m48s per run · fits the budget`, or `! over max_cost_usd` (`M3208`). `M3201` (agents, no budget) can't happen from the wizard: a budget is always chosen.
- Errors don't block create (the editor fixes them, and a pipeline saved with errors keeps its triggers off, ADR-034); the line says `✗ 2 · ]e walks them in the editor`.
- `then`: **dry run, then edit** (default) or **open the editor**. A dry run waits until lint is clean (§18.7.12).
- `enter` (or `:w`, `:create`) writes the pipeline file, its prompt files and any scripts, then **hands off to the editor** on the new pipeline: the selection on the first placeholder or problem, the status line naming what was written (`created .mill/pipelines/find-flaky-tests.mill.yaml + prompts/find-flaky-tests.md · ✓ lint clean · dry run starting`). A blank pipeline opens the editor with the trigger palette.
- `esc` before create writes nothing. The draft is kept for the session: `n` resumes it, `:new!` starts over.

## 9. CLI parity

Every step is a flag; no flags opens the same steps as terminal prompts (§17.5). Proposed §17.1 line:

```
mill new [<name>] [--from <source>] [--describe "…"] [--agent id/model] [--effort low|medium|high]
         [--steps a,b,c] [--trigger <spec>] [--budget small|medium|large|<tokens>,<usd>,<wall>]
         [--on-failure none|gh.issue|rex] [--allow part,…] [--scope project|user]
         [--then edit|dry-run|none] [--dry-run] [--json]
  <source>: template:<id> | pattern:<id> | pipeline:<name> | describe:"…" | <file> | <url> | gh:owner/repo[/path][@ref] | -
  <spec>:   manual | cron "<5 fields>" | webhook <path> | git <branch> | after <pipeline> | file <path>
```

| Wizard | `mill new` | `:` command (in the wizard) |
|---|---|---|
| describe it | `--describe "…"` or `--from describe:"…"` | `:describe …` |
| template / pattern | `--from template:issue-to-pr`, `--from pattern:best-of-n` | `:template <id>` |
| duplicate | `--from pipeline:nightly-sweep` | `:dup <pipeline>` |
| import a file | `--from <file\|url\|gh:…\|->` | `:import <path>` |
| blank | `--from blank` (or no `--from`, with `--steps` empty) | — |
| shape: steps, agent | `--steps ws,work,check,pr`, `--agent claude/sonnet --effort medium` | `:agent <id/model>` |
| run: trigger, budget, failure | `--trigger "cron 0 3 * * *"`, `--budget medium`, `--on-failure gh.issue` | `:trigger …`, `:budget …` |
| advanced | `--allow gh.pr`, `--scope user` | — |
| review: name, then | `<name>`, `--then dry-run` | `:name <name>`, `:create [--dry-run]` |
| preview only | `--dry-run` prints the YAML, lint and estimate, writes nothing (`--json` for scripts) | — |

`mill new` refuses the same things the wizard does, with the same codes. `mill import <src>` (§9.10) stays for bundles and keeps its own confirm; `mill new --from <src>` is the import path that goes through the wizard's choices.

## 10. Ranking

| Feature | Rank | Why |
|---|---|---|
| Four steps, breadcrumb, `enter` through defaults | MUST | The fastest path to a working pipeline |
| NORMAL / INSERT with `hjkl`, `i`, `esc`, `NEW · <mode>` on the status line | MUST | Same keys as the editor; no key typed into the wrong place |
| Sources: describe, template, duplicate, import, blank | MUST | Every way people start |
| Import: path + `tab`, picker, pasted paths, type detection, preview, error codes | MUST | The n8n migration path and `.millpkg` sharing |
| Live graph/YAML preview, live lint and estimate | MUST | Nothing is written blind |
| Editor hand-off on create, placeholders walked with `]e` | MUST | The wizard ends where building continues |
| `mill new` parity (every step a flag, `--from -`, `--dry-run --json`) | MUST | Principle 9 |
| `:` commands in the wizard (`:w`, `:q`, `:template`, …) | SHOULD | Speed for people who know what they want |
| Refine sentence, "without X" removals, why-notes | SHOULD | Makes the draft steerable without the editor |
| Plugin templates in the gallery | SHOULD | Plugins ship starters (§9) |
| Dry run on create | SHOULD | Catches a broken pipeline in the first minute |
| Draft kept for the session (`n` resumes) | SHOULD | `esc` by mistake is cheap |
| URL and `gh:` sources | LATER | Needs milld's fetch path and allowlist (registry work, v0.5) |
| Drafts that survive a restart (swap file like the editor's) | LATER | |
| Zapier import | LATER | §23.3 lists only n8n |

## 11. Implementation (Go)

- `internal/pipeline/importer`: `Detect(name string, r io.Reader) (Kind, error)` and one mapper per kind (`n8n`, `mill`, `millpkg`, `pipe`) returning a `Proposal{Doc, Files, Mapped, Stubs, Warnings}`. Pure functions, golden-tested with fixtures in `testdata/import/`; shared by `mill new --from`, `mill import` and the wizard.
- `internal/pipeline/draft`: the describe step: builds the prompt, runs a headless agent through the normal runner with a fixed output schema, validates the result into a `Proposal`. The keyword drafter is the offline fallback and the test double.
- `internal/surface/tui/wizard`: the Bubble Tea model (steps, panes, picker, preview) using the editor's `keymap` modes and the shared bottom-row command line. Writes go through `SavePipeline` like the editor's `:w`.
- `cmd/mill/new.go`: flag parsing into the same `WizardState`, so the TUI and the CLI build the identical document (one golden test per source kind compares both).

## 12. Decisions (2026-11-12)

All decided. **User** = the user chose it; **default** = the plan's default, accepted (ADR-050).

| # | Decision | Chosen | By |
|---|---|---|---|
| 1 | Number of steps | Four: start › shape › run › review | user (ADR-045) |
| 2 | When the name is chosen | Last, on review, suggested from the source | user (ADR-045) |
| 3 | Describe drafter | **Local LLM through Ollama**, like rex's summarizer; settings `drafter.backend` (`ollama` · `agent` · `off`), `drafter.base_url`, `drafter.model`, `drafter.agent`, `drafter.timeout` in `:settings` (§19.6). Unavailable → keyword rules with `M2212`, never blocks. Replaces `wizard.agent`. | user (ADR-047) |
| 4 | Budget default | `medium` | default |
| 5 | On failure default | GitHub issue when `mill/github` is installed, else nothing | default |
| 6 | After create | Dry run, then edit | user (ADR-046) |
| 7 | Unmapped n8n nodes | `n8n.call` stubs (`M2304`) | default |
| 8 | n8n steps that waited for a person | `handoff` that ends the branch (`M2307`) | user (ADR-048) |
| 9 | Duplicate's triggers | Start `manual`; `as in <original>` offered | user (ADR-049) |
| 10 | Where imports are read | milld | default |
| 11 | Size cap | 5 MB | default |
| 12 | `mill import` vs `mill new --from` | Keep both | default |
| 13 | Upload in the real TUI | Path field, picker, pasted paths; OSC 52/kitty transfer LATER | default |

## 13. Spec patch for §18.8 (merged in v2.2; history only, the spec wins)

Replace the "New pipeline (`n`)" subsection of §18.8 with:

````md
#### New pipeline (`n`)

`n` on the board, `:new`, or `mill new` (the same steps as terminal prompts, or flags) opens a square overlay with four steps and a breadcrumb: `1 start › 2 shape › 3 run › 4 review`. Every field has a default, so `enter` goes all the way through. Nothing is written until review.

| Step | You choose | Notes |
|---|---|---|
| **1 start** | Where it comes from: **describe it** (a sentence), **template** (built-ins, plugin templates and the editor's patterns, in a gallery), **duplicate** (a pipeline you have), **import a file**, or **blank** | A sentence goes to a headless agent (`wizard.agent`, the cheapest signed-in model by default), which proposes the steps, trigger and agent, each with the words that caused it. Import takes a path (`tab` completes; `o` opens a file picker; a path pasted by dropping a file on the terminal also works), a URL, `gh:owner/repo/path`, or a pipe expression; it detects n8n workflows, `.millpkg` bundles, mill pipelines and pipe expressions, and previews what it would create before anything is written. |
| **2 shape** | What it does: the steps (`space` toggles), the agent and effort, a `refine` sentence that redrafts | A live graph beside it, `Y` for the YAML. A copy or an import shows its nodes instead of the checklist. |
| **3 run** | Trigger (and its one setting), budget preset, what happens on failure (nothing, a GitHub issue, a rex session) | `▸ advanced` folds max agents, the destructive steps it may run, and scope. The run never waits on you, so its limits are set here. |
| **4 review** | The name (suggested from the source), then **dry run, then edit** or **open the editor** | Shows the files it writes, the estimate against the budget, and live lint. `enter` (or `:w`) creates and opens the editor on the new pipeline; lint errors don't block it (§18.7.9). |

**Modes and keys.** The editor's model (§18.7.4): `NORMAL` moves with `hjkl` (`j`/`k` rows, `h`/`l` change a `◂ value ▸` or move between the source list and its pane), `enter` uses the selection and goes on, `tab`/`shift+tab` and `1`–`4` change step, `space` toggles, `Y` switches the preview, `o` opens the file picker, `/` filters, `gx` explains an error, `:` opens the command line, `?` lists every key, `esc` leaves the pane and then cancels (nothing written; `n` resumes the draft). `i` (or `enter` on a text field) enters `INSERT`: `tab` completes paths, names and cron presets, `enter` finishes (and drafts, loads or checks), `esc` keeps the text, `ctrl+c` drops it. The status line shows `NEW · NORMAL` or `NEW · INSERT` and the step.

**Import.** n8n workflows map as in §23.3; unmapped nodes become `n8n.call` stubs (`M2304`) and a node that waited for a person becomes a `handoff` (`M2307`). mill pipelines open as-is; a `.millpkg` shows the §9.10 summary and which of its pipelines to open. Imports are read by milld, capped at 5 MB, and refused with a code: `M2301` unknown kind, `M2302` newer `mill:` version, `M2303` unreadable source, `M2305` not a valid `.millpkg`, `M2306` too large or binary, `M2308` `-` outside a shell. A created import is trusted (§22).

**Errors** show inline and on the status line: `M1210` bad name, `M2205` name taken, `M2211` no steps on, `M2309` unknown template, `M3050` invalid trigger setting, `M6101` agent CLI missing.

**CLI parity:** `mill new [<name>] --from template:<id>|pattern:<id>|pipeline:<p>|describe:"…"|<file>|<url>|gh:…|- --agent claude/sonnet --effort medium --steps ws,work,check,pr --trigger "cron 0 3 * * *" --budget medium --on-failure gh.issue --allow gh.pr --scope project --then dry-run`. `--dry-run` prints the YAML, lint and estimate and writes nothing.
````

## 14. Spec patch for §17 (merged in v2.2; history only)

In §17.1, replace the `mill new` lines with:

````md
mill new [<name>] [--from <source>] [--describe "…"] [--agent id/model] [--effort low|medium|high]
         [--steps a,b,c] [--trigger spec] [--budget small|medium|large] [--on-failure none|gh.issue|rex]
         [--allow part,...] [--scope project|user] [--then edit|dry-run|none] [--dry-run] [--json]   # no flags: the wizard
         # <source>: template:<id> | pattern:<id> | pipeline:<name> | describe:"…" | <file> | <url> | gh:owner/repo[/path][@ref] | -
````

In §17.5's table, the Wizard row becomes: `` `mill new` (prompts in the terminal, or flags; `--from -` reads stdin) ``.

## 15. Spec patch for §29 (merged in v2.2; history only)

In §29.4's table, the `M2xxx` examples gain `` `M2301` not an importable file ``; the `M1xxx` examples are covered in [command-line.md](command-line.md). Add to the explain pages (§29.5) one page per new code from §5.5 above: `M2211`, `M2301`–`M2309`, `M3050`. In §18.7.9's lint table add:

````md
| `M3050` | error | A trigger setting is invalid: a cron schedule without 5 fields (or `@every`), a webhook path without a leading `/` | Focus the trigger |
````
