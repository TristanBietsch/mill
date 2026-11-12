## 19. Configuration (Lua)

mill is configured in **Lua**. Pipelines stay in YAML (§7) because the canvas has to read and write them losslessly. Everything else (settings, plugins, agents, MCP servers, secrets backends, runners, hooks, custom commands, keymaps and themes) is Lua.

Why Lua: config can use logic (different settings per machine, a loop to define ten similar agents), it supports comments, people already know it from Neovim, WezTerm and Hammerspoon, and it's tiny to embed. mill already uses Lua for transforms and Lua parts, so it's one language for every kind of extension.

### 19.1 Files

| File | Purpose |
|---|---|
| `~/.config/mill/init.lua` | Your config. Hand-written, fully programmable. |
| `~/.config/mill/settings.lua` | Written by `mill config set` and the TUI settings screen. A plain `return { … }` table marked "managed by mill". Loaded *before* `init.lua`, so anything in `init.lua` wins. |
| `~/.config/mill/lua/` | Your own modules, available to `require` (`require("work")`). |
| `<repo>/.mill/mill.lua` | Project config: plugins, agents and defaults for this repo, versioned with it. |
| `plugin.lua` | Plugin manifests (§9.5). Data-only. |

### 19.2 Load order and precedence

From lowest to highest:

1. Built-in defaults
2. `settings.lua` (managed)
3. `init.lua` (yours)
4. `.mill/mill.lua` (project)
5. Environment variables (`MILL_MAX_CONCURRENT_AGENTS=2`)
6. Command-line flags

`mill config get <key> --why` shows the effective value and which layer set it. `mill config set` warns when `init.lua` or a project file will override what it just wrote.

`milld` loads config at startup and reloads it when any config file changes or on `mill reload`. A reload that fails keeps the previous config running and reports the error with file and line, so a typo never takes the daemon down.

### 19.3 Example `init.lua`

```lua
local host = mill.host()   -- hostname, os, arch

mill.setup {
  max_concurrent_runs   = 8,
  max_concurrent_agents = host.name == "workstation" and 6 or 2,
  default_runner        = "local",
  keep_workspaces       = "on_failure",
  retention             = "30d",
  trusted_dirs          = { "~/code" },
  notify                = "desktop",
  theme                 = "auto",
  gateway = { listen = "127.0.0.1:7337" },
}

mill.plugins {
  "mill/github",
  { "acme/linear", version = "^1.2" },
}

mill.secrets { backends = { "keychain", "1password", "env" } }

-- agents: same fields as §11.1; merged by id with the built-in entries
mill.agent "claude" {
  models = {
    { id = "opus",   args = { "--model", "opus" },   price = { input = 15, output = 75 } },
    { id = "sonnet", args = { "--model", "sonnet" }, price = { input = 3,  output = 15 } },
  },
}
mill.agent "local-qwen" {
  command  = { "ollama", "run" },
  models   = { { id = "qwen", args = { "qwen2.5-coder" } } },
  prompt   = { via = "stdin" },
  headless = { output = "stdout-text" },
  detect   = { kind = "exit" },
}

mill.mcp "slack" {
  command = { "npx", "-y", "@modelcontextprotocol/server-slack" },
  env     = { SLACK_BOT_TOKEN = mill.secret("SLACK_TOKEN") },
}

mill.runner "gpu-box" { kind = "ssh", host = "gpu.local", max_concurrent = 2 }

-- an inline Lua part (§9.2, level 3)
mill.part(mill.node {
  id = "my.slugify",
  with = { field = "string" },
  run = function(ctx, item) item[ctx.with.field] = mill.slug(item[ctx.with.field]); return item end,
})

-- runtime hooks
mill.on("run_updated", function(run)
  if run.state == "failed" and run.tags.prod then
    mill.start("page-me", { run = run.id })
  end
end)

-- custom commands: `:standup` in the TUI, `mill cmd standup` on the CLI
mill.command("standup", function(args)
  return mill.start("morning-brief", { since = args[1] or "24h" })
end)

-- TUI tweaks
mill.keymap("canvas", "gp", ":run publish<cr>")
mill.theme { accent = "#4FB3BF", glyphs = { assert = "⊢", agent = "◆" } }
mill.setup { sound = { set = "foundry", volume = 0.5, when = "unfocused" } }

-- machine-specific overrides
pcall(require, "local")
```

### 19.4 Lua API

**Config phase** (runs at load, declares things):

| Function | Declares |
|---|---|
| `mill.setup{}` | Settings from the settings registry (§19.6) |
| `mill.plugins{}` / `mill.plugin(spec)` | Plugins to install and load (§9.6) |
| `mill.agent(id){}` | Agent registry entries (§11.1) |
| `mill.mcp(name){}` | MCP servers |
| `mill.runner(id){}` | Named runner configurations |
| `mill.secrets{}` | Secret backends and their order |
| `mill.part(def)` with `mill.node{}`, `mill.trigger{}`, `mill.sink{}` | Inline Lua parts |
| `mill.pipeline(name){}` | A pipeline defined in Lua (generated, read-only in the canvas; §19.7) |
| `mill.on(event, fn)` | Runtime hooks |
| `mill.soundset(name){}` | A custom soundset: event name → sound file (§18.11) |
| `mill.command(name, fn)` | Custom commands |
| `mill.keymap(screen, keys, action)`, `mill.theme{}` | TUI customization |

**Helpers** (available in both phases): `mill.host()`, `mill.env(name)`, `mill.path(p)`, `mill.secret(name)` (an opaque handle, resolved only where it's used), `mill.slug()`, `mill.json`, `mill.yaml`, `mill.log(level, msg)`.

**Runtime only** (inside hooks, commands and Lua parts): `mill.start(pipeline, inputs)`, `mill.runs(filter)`, `mill.cancel(run)`, `mill.rerun(run, from)`, `mill.notify(msg)`, `mill.sound.play(event)`, and the `ctx` object in parts (`ctx.with`, `ctx.http`, `ctx.secret`, `ctx.log`, `ctx.progress`, `ctx.artifact`).

Hook events: `daemon_started`, `config_reloaded`, `run_added`, `run_updated`, `node_updated`, `trigger_fired`, `plugin_installed`, `budget_warning`.

### 19.5 Execution model and safety

- **Your config is trusted**, the same way an editor config is. `init.lua` and your `lua/` modules get the standard library, with `os.execute` and `io.popen` available.
- **Project config is not trusted by default.** A cloned repo's `.mill/mill.lua` is first evaluated in a data-only sandbox: it can declare plugins, agents, settings and defaults, but hooks, commands, inline parts and anything touching `os`, `io` or the network are ignored and listed as "needs trust". `mill trust .` reviews and approves it (recorded by hash, like pipelines; §22). The same command arms the project's pipelines, which stay disarmed until then.
- The config phase must finish within 2 seconds. Hooks run on their own Lua state in `milld` with a per-call time limit. A hook that errors or times out is logged and skipped; it never fails a run or stops the daemon.
- Lua parts run in separate sandboxed Lua states, one per plugin, and only get the API their manifest's permissions allow.

### 19.6 Settings registry

One registry defines every setting: id, type, default, section, help text and validation. `mill.setup{}`, `settings.lua`, `mill config` and the TUI settings screen all go through it. Unknown keys produce a warning with the closest valid key ("unknown setting `max_agents`; did you mean `max_concurrent_agents`?"). Wrong types are errors with file and line.

| Setting | Default | Meaning |
|---|---|---|
| `max_concurrent_runs` | 8 | Global run cap |
| `max_concurrent_nodes` | 16 | Global node-run cap |
| `max_concurrent_agents` | 4 | Global cap on agent node-runs |
| `default_runner` | `local` | Runner when nodes and pipelines don't set one |
| `workspace_root` | `~/.local/share/mill/workspaces` | Where worktrees and temp dirs go |
| `keep_workspaces` | `on_failure` | `never`, `on_failure`, `always` |
| `retention` | `30d` | How long finished runs are kept |
| `trusted_dirs` | empty | Folders where agents' folder-trust dialogs may be answered automatically (refuses `/`, home, relative paths and `..`) |
| `env.passthrough` | `PATH HOME USER LOGNAME SHELL LANG LC_* TERM TZ TMPDIR SSH_AUTH_SOCK XDG_* GIT_* EDITOR` (+ Windows essentials) | Variables from `milld`'s environment that steps see (§12.1). Everything else is dropped |
| `agents.extract` | same agent, cheapest priced model | Agent and model for the extraction pass (§11.3), or `off` |
| `drafter.backend` | `ollama` | Who drafts a pipeline from a sentence (the wizard's *describe it*, `mill new --describe`, §18.8): `ollama` (a local model through Ollama's HTTP API: free, offline), `agent` (a headless registry agent: costs tokens, opt-in) or `off` (local keyword rules only). An unavailable backend falls back to keyword rules with `M2212`; it never blocks |
| `drafter.base_url` | `$OLLAMA_HOST`, else `http://127.0.0.1:11434` | Ollama's base URL. A URL off this machine receives your sentence |
| `drafter.model` | `llama3.2:3b` | Resolved against the pulled models (`/api/tags`): if it isn't pulled, the first pulled one of `llama3.2:3b`, `qwen2.5:3b`, `gemma2:2b`, `llama3.2:1b`, `phi3:mini` is used and logged, never an arbitrary model; none pulled makes the backend unavailable (`ollama pull <model>` fixes it) |
| `drafter.agent` | cheapest signed-in model | Agent and model for `drafter.backend = agent` (`claude/haiku`) |
| `drafter.timeout` | `20s` | Per request, 1s–2m. Only an unreachable server or a 5xx is retried, once; a timeout falls back to keyword rules |
| `shutdown.drain_timeout` | `10m` | How long `mill daemon stop --drain` waits for live node-runs (§15.7) |
| `plugins.sandbox` | `none` | `container` runs executable parts inside the container runner, with their declared permissions enforced |
| `plugins.registry` | official | Registry URL, or a list for private registries |
| `plugins.auto_update` | `off` | `off`, `patch` or `minor`. Never adds permissions on its own (§18.9) |
| `gateway.listen` | off | e.g. `127.0.0.1:7337` |
| `gateway.public_url` | empty | Set when using a tunnel or the hosted relay |
| `notify` | `desktop` | `desktop`, `bell`, `off`, or a sink part |
| `sound.set` | `millrace` | Soundset: `millrace`, `foundry`, `chiptune`, `bell`, `off`, or a custom set (§18.11) |
| `sound.volume` | 0.6 | 0.0 to 1.0 |
| `sound.voice` | on | The spoken "mill" when the TUI starts |
| `sound.interface`, `sound.editing`, `sound.runs` | on | Turn each group of sounds on or off |
| `sound.when` | `always` | `always`, or `unfocused` for run sounds only when the terminal isn't focused |
| `sound.quiet_hours` | empty | e.g. `22:00-08:00`: no sounds in that window |
| `mouse_enabled` | true | TUI mouse capture |
| `screensaver.enabled` | true | Show the logo and counts after `screensaver.after_minutes` idle (§18.12) |
| `screensaver.after_minutes` | 5 | 1–120. Idle minutes before the screensaver shows |
| `screensaver.keep_awake` | true | macOS only: keep the display awake while the screensaver shows |
| `reduce_motion` | false | No spinners animating, no screensaver drift |
| `theme`, `density` | `auto`, `normal` | TUI appearance |
| `editor` | `$EDITOR` | For `E`, `mill edit` and `mill config edit` |
| `telemetry` | `off` | Never on by default |

### 19.7 Pipelines in Lua (optional)

`mill.pipeline(name){}` builds a pipeline from Lua: useful for generating many similar pipelines (one per repo, one per customer). Lua-defined pipelines run like any other, appear in the library with a `λ` badge, and open in the canvas read-only with "defined in Lua: edit source" (`E` opens the Lua file). `mill show <p> --yaml` prints the YAML equivalent, and `mill export` writes them as YAML, so shared pipelines are always plain YAML.

### 19.8 Tooling

- `mill config edit` opens `init.lua` in your editor.
- `mill config check` loads all config layers without applying them and reports errors and warnings.
- `mill config list [--json]` shows every effective setting with its source layer.
- `mill lua-types` writes LuaLS type annotations for the whole `mill.*` API, so editors autocomplete and type-check your config, plugins and Lua parts.


---

[← spec index](index.md)
