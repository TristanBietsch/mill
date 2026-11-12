## 9. Parts and plugins

Everything mill can do comes from **parts**, and parts are shipped in **plugins**. The core only schedules, hosts and stores.

- A **part** is one capability the engine uses: a node type, a trigger, a runner, a secrets backend, a sink or an artifact store.
- A **plugin** is a package you install, share, import and export. It can hold parts, MCP server definitions, agent definitions, pipelines, prompts, config defaults and TUI extras. The Linear plugin, for example, brings Linear nodes, a Linear trigger, a ready-made "Linear issue → PR" pipeline and its icon.
- A **pipeline bundle** is a plugin that mostly contains pipelines. Exporting a pipeline produces one, and importing one installs its pipelines plus any plugins they need (§9.9–9.10). One package format covers both cases.

### 9.1 Kinds of capability

| Capability | What it provides | Examples |
|---|---|---|
| `node` | A step type | `sh`, `http`, `agent`, `assert`, `linear.issue.create` |
| `trigger` | A source of runs | `cron`, `webhook`, `fswatch`, `linear.poll` |
| `runner` | A place to execute | `local`, `pty`, `container`, `ssh` |
| `secrets` | A secret backend | `env`, `keychain`, `1password`, `vault` |
| `sink` | A destination for run events | `otel`, `prometheus`, `notify`, `webhook-out` |
| `artifacts` | Artifact storage | `local`, `s3` |

### 9.2 Ways to extend, from least to most effort

| Level | What you write | Integration |
|---|---|---|
| **0. Any command** | Nothing. Use `uses: sh` or `uses: exec` with any program. | stdin/stdout, exit code |
| **1. Lua transform** | A few lines inline in a pipeline (`uses: lua`). | Sandboxed |
| **2. MCP server** | Nothing, if the server already exists. Declare it with `mill.mcp` and its tools become nodes. | Auto-discovered |
| **3. Lua part** | A node, trigger or sink written in Lua, inline in your config (`mill.part`) or in a plugin. Runs inside `milld` with a capability-scoped API (HTTP to allowed hosts, JSON, granted secrets). | Full schemas and ports; no separate process |
| **4. Executable part** | A program in any language speaking the parts protocol (§9.3). | Full: schemas, ports, streaming, cancel, resume |
| **5. WASM part** (later) | A WASI module | Sandboxed and portable |

Levels 3–5 are usually distributed as plugins.

### 9.3 Parts protocol (v1)

An executable part is a program that `milld` starts with stdin and stdout connected. It uses the same envelope as the daemon protocol: one JSON object per line, `{ "v": 1, "kind": "...", "type": "...", "id": "...", "data": { ... } }`. Stderr is captured as the part's log. Lua and WASM parts implement the same lifecycle through function calls instead of stdio.

**Lifecycle**

1. `milld` → `Describe`. The part answers `Manifest`: id, version, capabilities, and for each capability its config schema, port definitions, output schema, and flags (`streaming`, `cancellable`, `resumable`, `idempotent`, `destructive`, `needs_secrets`, `needs_ws`, `interactive`). A part flagged `interactive` waits for a person; it can be installed, but lint refuses it in a pipeline (`M3206`, §2).
2. `milld` → `Open` with a node-run (or trigger, or runner) context: resolved `with` config, workspace, granted secrets, budget remaining, a scratch dir.
3. Items flow: `milld` → `Item` (input), part → `Item` (output, with a `port`). `milld` → `EndOfInput`.
4. While working, the part may send `Log`, `Progress` (percent, message), `Usage` (tokens, cost), `Artifact` (register a file). There is no way for a part to ask a person anything: if it can't proceed, it fails with a clear error.
5. The part ends the node-run with `Done` or `Fail` (error class, message, retryable flag).
6. `milld` can send `Cancel` at any time; the part must finish with `Fail` (class `cancelled`) within the grace period, or it's killed.
7. `Close` ends the context. Long-lived parts handle many contexts at once, multiplexed by context id; per-call parts exit after `Close`.

**Rules**
- Unknown message types are ignored. A part that doesn't support something answers `Error` with code `unsupported`.
- A part that declares `resumable` gets `Resume` with its last checkpoint after a daemon restart. It saves checkpoints by sending `Checkpoint` events.
- A part may only read secrets it was granted in `Open`.
- Health: `milld` sends `Ping` to idle long-lived parts; no `Pong` within the timeout means restart.
- Some parts also answer `PartCall` requests from clients, for things like filling a dropdown in the inspector (list Linear teams, list Slack channels).

**SDKs** (thin helpers, optional): Go, TypeScript, Python, plus the built-in Lua API. They handle the envelope, manifest and lifecycle, so a part is a handful of handlers.

### 9.4 Resolution order

Parts are resolved by id in this order. A later source overrides an earlier one with the same id (and `mill part ls` shows which one won):

1. Built-ins embedded in `milld`.
2. Plugins installed in user scope (from `mill.plugins` in `init.lua`).
3. Plugins installed in project scope (from `.mill/mill.lua`).
4. Inline parts from `mill.part` in your config.
5. Executables named `mill-part-<name>` on `PATH` (like git subcommands).
6. MCP servers declared with `mill.mcp`, exposed as `mcp:<server>/<tool>`.

Part ids are namespaced by plugin (`linear.issue.create`). If two plugins collide, alias one in config: `{ "acme/linear", as = "lin" }` turns its parts into `lin.issue.create`.

### 9.5 Plugin package format

A plugin is a directory (or a `.millpkg` archive of one, a gzipped tar):

```
mill-linear/
  plugin.lua          manifest (required)
  parts/              executable parts or Lua parts
  pipelines/          *.mill.yaml (+ layout sidecars), offered as templates
  prompts/            prompt files referenced by pipelines
  icons/              optional glyph overrides
  README.md
  LICENSE
```

`plugin.lua` returns a table. It's evaluated in a data-only sandbox (no `io`, `os`, network or `require` outside the plugin), so reading a manifest can never run code.

```lua
return {
  name        = "acme/linear",
  version     = "1.2.0",
  description = "Linear issues, comments and projects for mill",
  license     = "MIT",
  homepage    = "https://github.com/acme/mill-linear",
  mill        = ">=0.5 <2",          -- compatible engine versions

  requires = { "mill/http >=1" },     -- other plugins

  provides = {
    parts = {
      { id = "linear.issue.create", kind = "node",    lua  = "parts/issue_create.lua" },
      { id = "linear.issue.update", kind = "node",    lua  = "parts/issue_update.lua" },
      { id = "linear.comment",      kind = "node",    lua  = "parts/comment.lua" },
      { id = "linear.poll",         kind = "trigger", lua  = "parts/poll.lua" },
      { id = "linear.webhook",      kind = "trigger", exec = "parts/bin/linear-hook" },
    },
    pipelines = { "pipelines/linear-issue-to-pr.mill.yaml" },
    mcp       = {},   -- or wrap an existing MCP server instead of writing parts
    agents    = {},
    commands  = { { name = "linear-login", lua = "parts/login.lua" } },
    soundsets = {},   -- e.g. { "sounds/linear-chimes" }
    patterns  = {},   -- editor patterns, e.g. { "patterns/triage-to-linear.mill.yaml" } (§18.7.6)
  },

  secrets = {
    LINEAR_API_KEY = { description = "Personal API key", help = "Linear → Settings → API" },
  },

  permissions = {
    network     = { "api.linear.app" },
    filesystem  = {},                     -- none outside its scratch dir
    destructive = { "linear.issue.update" },
  },

  build = {                               -- only for executable parts
    { os = "darwin", arch = "arm64", url = "https://.../linear-hook-darwin-arm64", sha256 = "…" },
    { os = "linux",  arch = "amd64", url = "https://.../linear-hook-linux-amd64",  sha256 = "…" },
    { from = "go", module = "github.com/acme/mill-linear/cmd/linear-hook" },  -- fallback
  },
}
```

A plugin doesn't have to write any parts. A thin plugin can simply declare an existing MCP server (for example the vendor's official Linear MCP server), add nicer forms and icons for its tools, and ship pipeline templates.

### 9.6 Installing plugins

Plugins are declared in Lua config, the way an editor's plugin manager works, and `mill plugin` commands keep that declaration and the lockfile in sync.

```lua
-- ~/.config/mill/init.lua
mill.plugins {
  "mill/github",                                   -- official plugin, latest compatible
  { "acme/linear", version = "^1.2" },
  { "gh:someone/mill-notion", tag = "v0.4.1" },    -- straight from a GitHub repo
  { dir = "~/code/mill-plugin-standup", dev = true },  -- local plugin, hot-reloaded
}
```

```
mill plugin add acme/linear[@^1.2] [--scope user|project]
mill plugin ls [--outdated]
mill plugin info acme/linear            # parts, permissions, secrets, dependents
mill plugin update [name|--all]
mill plugin rm acme/linear [--force]    # refuses while pipelines use it, lists them
mill plugin search linear               # searches the registry (§25)
```

`mill plugin add` adds the line to `init.lua` (or `.mill/mill.lua` with `--scope project`) and installs immediately. Editing the Lua file by hand works too: `milld` notices the change and installs what's missing, or `mill plugin sync` does it on demand.

**Sources:** registry names (`owner/name`), `gh:owner/repo[@tag]`, any git URL, a local `dir`, a `.millpkg` file, or an HTTPS URL to one.

**Install flow:**
1. Resolve versions against semver ranges and `requires`. Conflicts fail with a clear explanation.
2. Download, verify checksums, and verify the signature when one is present (the registry requires signatures).
3. Show a summary the first time: parts it adds, permissions (network hosts, filesystem, destructive actions), secrets it needs, and any build step. Ask once. `--yes` accepts in scripts.
4. Build or fetch executables for this platform.
5. Write `mill.lock` (project) or the user lock: exact versions plus hashes of the archive and every executable.
6. Prompt for missing secrets (`mill secrets set LINEAR_API_KEY`), unless `--no-secrets`.

**Permissions** are enforced for Lua parts (the API only reaches allowed hosts and paths) and for WASM parts. For native executables they're declared and shown but can't be enforced on the host. To enforce them, set `plugins.sandbox = "container"` and executable parts from that plugin run in the container runner with the network and filesystem limited to what was declared.

### 9.7 Building plugins

```
mill plugin new linear --lang lua|go|ts|python   # scaffold with manifest, one part, a test, a README
mill plugin dev ./mill-linear                    # link into your config, hot-reload on save
mill plugin test ./mill-linear                   # runs the plugin's test pipelines against recorded fixtures
mill plugin pack ./mill-linear                   # → acme-linear-1.2.0.millpkg (+ checksums)
mill plugin publish ./mill-linear                # sign and upload to the registry
```

A Lua node part is a few lines:

```lua
-- parts/issue_create.lua
return mill.node {
  id = "linear.issue.create",
  glyph = "◩",
  with = { team = "string", title = "string", body = "string?", labels = "[string]?" },
  output = { id = "string", url = "string" },
  secrets = { "LINEAR_API_KEY" },
  run = function(ctx, item)
    local res = ctx.http.post("https://api.linear.app/graphql", {
      headers = { Authorization = ctx.secret("LINEAR_API_KEY") },
      json = { query = CREATE_ISSUE, variables = { input = ctx.with } },
    })
    local issue = res.json.data.issueCreate.issue
    return { id = issue.identifier, url = issue.url }
  end,
}
```

### 9.8 Using a plugin in a pipeline

```yaml
nodes:
  file:
    uses: linear.issue.create
    with:
      team: ENG
      title: "Flaky test: ${{ item.test }}"
      body: ${{ item.log_tail }}
```

The canvas palette (`a`) lists every installed plugin's parts with their glyphs, and the inspector builds forms from their schemas. Dropdowns like "team" are filled by the part through `PartCall`.

### 9.9 Exporting pipelines

```
mill export issue-to-pr                         # → issue-to-pr.millpkg
mill export issue-to-pr --dir ./share/          # unpacked folder instead of an archive
mill export issue-to-pr --gist                  # as a GitHub gist (needs gh)
mill export --tag code --out code-kit.millpkg   # several pipelines in one bundle
mill export issue-to-pr --to n8n|mermaid|gha    # other formats (§23)
```

The exporter builds a bundle with:
- the pipeline file and its layout sidecar;
- every file it references (`@prompts/plan.md`, templates, Lua scripts);
- a generated `plugin.lua` listing the pipelines and the plugins they need, with version ranges taken from your lockfile;
- the **names** of the secrets it needs (never their values) and the agents it uses.

Before writing, it checks the bundle for things that shouldn't leave your machine:
- **Absolute paths and machine-specific values** (`~/code/app`, a local username) are flagged. `--parametrize` turns them into pipeline inputs with your value as a hint, not a default.
- **Anything that looks like a secret** in the YAML or the referenced files blocks the export until it's removed or replaced with a `secrets.` reference.
- **Run history is never included.** Use `--with-example-run` to add one run's items, redacted, as a demo.

### 9.10 Importing pipelines

```
mill import ./issue-to-pr.millpkg
mill import gh:acme/mill-pipelines/release-notes
mill import https://example.com/kits/code-kit.millpkg [--scope user|project]
mill import <src> --map-agent codex=claude --dry-run
```

Import shows a summary before it changes anything:

```
≋ import issue-to-pr (from acme/mill-pipelines @ 3f2a9c1)
  pipelines   issue-to-pr
  plugins     mill/github ^1  (installed)   ·   acme/linear ^1.2  (will install)
  agents      claude (ok) · codex (missing → map with --map-agent or install)
  secrets     GITHUB_TOKEN (set) · LINEAR_API_KEY (missing)
  runners     local, container
  permissions network: api.github.com, api.linear.app · destructive: git.push, gh.pr
  trust       not trusted yet: review the files above, then confirm
proceed? [y/N]
```

On confirm, it installs the missing plugins, records the pipeline's hash as trusted, places the files in the chosen scope, updates the lockfile, and asks for missing secrets. `--dry-run` stops after the summary. An imported pipeline that fails `mill lint` on your machine is still imported but marked disabled, with the lint errors shown.

### 9.11 Lockfile

`mill.lock` (project) and `~/.local/share/mill/plugins/lock` (user) record, for each plugin: source, exact version, archive hash, executable hashes and resolved dependencies. Runs record which lock entry they used. `mill plugin sync --frozen` installs exactly what the lockfile says and fails on any drift, which is what CI should use.


---

[← spec index](index.md)
