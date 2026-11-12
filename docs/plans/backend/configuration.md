# Configuration

Status: accepted with the 2026-11-12 refinement. Spec: [§19](../../spec/19-config-lua.md#192-load-order-and-precedence) (load order, settings registry, safety). Refines [architecture.md](architecture.md) §8.

How config is layered, validated, frozen into snapshots, reloaded and handed to the rest of the backend. `config` and `config/settings` are L5 ([layers.md](layers.md)); nothing below L5 imports them, consumers get values from a composition root.

## 1. Principles

1. **Config is data at the boundary.** Lua runs once per load and produces an immutable `config.Snapshot`. Code below L5 sees plain values (`sched.Limits{Nodes: 16}`), never Lua or the snapshot type [SRW ch. 14].
2. **Validate everything before anything changes** [SRW ch. 14]: a load either produces a fully valid snapshot or an error with file, line and a fix; a reload that fails keeps the last good snapshot running (spec §19.2).
3. **One registry, many front ends.** Every setting is defined once (id, type, default, range, help, apply mode) and `mill.setup{}`, `settings.lua`, env vars, flags, `mill config` and the TUI settings page all go through it (spec §19.6) [SRW ch. 15].
4. **Precedence is the Unix one** [TAOUP ch. 10]: built-in defaults < managed file < user file < project file < environment < flags, and `--why` always says which layer won.
5. **Config can't widen trust from an untrusted place** (ADR-029). A project file declares; it can't grant itself more power.

## 2. Layers

| # | Layer | File or source | Lua profile ([security.md](security.md) §3.2) | May set |
|---|---|---|---|---|
| 1 | Built-in defaults | registry in `config/settings` | n/a | everything |
| 2 | Managed | `~/.config/mill/settings.lua` | `data` | registry settings only |
| 3 | User | `~/.config/mill/init.lua` + `lua/` | `trusted` | everything, including hooks, commands, agents, runners, MCP servers, secret backends |
| 4 | Project | `<repo>/.mill/mill.lua` | `data` until `mill trust .`, then `trusted` | project-scoped settings, agents, plugins, defaults; **not** the trust-widening keys below |
| 5 | Environment | `MILL_<SETTING_ID_UPPER>` (e.g. `MILL_MAX_CONCURRENT_AGENTS=2`) | n/a | registry settings |
| 6 | Flags | `mill …`/`milld …` command-line flags | n/a | per command |

**Trust-widening keys** that only layers 1, 2, 3, 5 and 6 may set: `trusted_dirs`, `trusted_projects`, `env.passthrough`, `workspace_root`, `gateway.*`, `policy.allow_main_checkout`, `plugins.sandbox`, `secrets` backends. A project file that sets one gets a warning naming the key and the layer, and the value is ignored [BSRS ch. 5].

`mill config get <key> --why` prints the effective value, the layer and file:line that set it, and every lower layer it overrode.

## 3. The snapshot

```go
package config

type Snapshot struct {
	Settings settings.Values // typed, validated
	Agents   []agents.Entry  // merged by id (spec §11.1)
	Runners  []RunnerDef
	MCP      []MCPServerDef
	Secrets  SecretsDef      // backend order and references, never values
	Plugins  []PluginSpec
	Hooks    []HookRef       // Lua function refs, run by luaapi's dispatcher
	Origins  map[string]Origin // per key: layer, file, line (for --why)
	Hash     string          // sha256 of canonical JSON of the resolved values above, excluding Origins and Hooks bodies
}
```

- **Immutable** once built; shared by pointer and swapped whole ([concurrency.md](concurrency.md) §1).
- **Hash** is over resolved values, not source text, so a reformat doesn't change it. It's recorded as `config_hash` in `RunCreated`.
- **Frozen per run** (invariant 10 extended to config): a run keeps the agent registry, runner definitions and secret backend order of the snapshot it started with. Settings that are scheduling limits (pool sizes) apply to new grants immediately.

## 4. Validation

Five phases, all before a snapshot is published:

| Phase | Checks | Error |
|---|---|---|
| 1 Evaluate | Lua syntax and runtime errors, 2 s deadline per layer | M8001 with file:line |
| 2 Types | each key exists in the registry (did-you-mean otherwise), value type, range, enum | M8004 for unknown keys; the setting's validation error otherwise |
| 3 Cross-field | references resolve (a runner, agent or secret backend that's named exists); limits are coherent (`max_concurrent_agents` ≤ `max_concurrent_nodes` warns; `retention` ≥ 1 d); agent entries build a valid argv for every model | the setting's error with both keys named |
| 4 Policy | trust-widening keys from an untrusted or project layer (§2); `gateway.listen` public without `allow_public` | warning, value ignored |
| 5 Probe (optional) | agent binaries on PATH, MCP server commands exist | warnings only; `mill doctor` shows them |

Environment values are parsed with the setting's type and fail with the same validation error, naming `MILL_…` as the layer.

## 5. Reload

```mermaid
sequenceDiagram
  participant W as config watcher
  participant L as config.Load
  participant B as daemon/boot
  participant C as consumers
  W->>W: fsnotify on config files, debounce 250 ms (or mill reload / SIGHUP)
  W->>L: build new snapshot (off the hot path)
  alt invalid
    L-->>B: error (M8xxx, file:line)
    B->>C: ConfigStatus{error} (board line); last good snapshot stays
  else valid
    L-->>B: snapshot', diff vs current
    B->>C: apply in order: scheduler Resize → parthost diff (runners, MCP) → triggers re-arm changed → luaapi hooks swap
    B->>C: ConfigStatus{ok, restart_needed[]}
  end
```

Each setting declares an **apply mode** in the registry:

| Mode | Examples | When it takes effect |
|---|---|---|
| `hot` | pool sizes, `retention`, log level, `cancel_grace`, sounds | next use (next grant, next prune, next log line) |
| `new_runs` | agent registry, runners, secret backends, `env.passthrough` | runs started after the reload; live runs keep their snapshot |
| `restart` | `MILL_DATA_DIR`, socket path, `gateway.listen` | after `mill daemon restart`; `ConfigStatus.restart_needed` lists them and the board says so |

Shrinking a pool never kills work: the scheduler stops granting until usage drops under the new size. A reload that fails half-way through applying can't happen, because every consumer receives an already validated value and only applies it.

## 6. Client and daemon

- `milld` and `mill` read the same files with the same loader. The client loads with the runtime phase off (hook and command bodies recorded, not run) for TUI keymaps, themes and settings display.
- The client never sends config to the daemon. `mill config set` writes `settings.lua` (atomic temp + fsync + rename, stable key order, "managed by mill" header) and then sends `Reload`, so both sides agree by reading the same file.
- `mill run --local` builds its snapshot once per run; there's no watcher.

## 7. Secrets in config

Config holds secret **references** (backend names, `op://vault/item/field`, env var names), never values. `mill config list` and `--why` print references only. The daemon redactor registers MCP server env values that come from secret references (spec §21).

## 8. Evolving settings

- A renamed setting keeps an alias for at least one minor release; using the old name warns with the new one (M8004-style did-you-mean) and `mill config set` rewrites `settings.lua` to the new name [SEG ch. 15].
- A removed setting is first deprecated (warning, still honored), then ignored with a warning, never silently.
- Defaults that change are listed in the CHANGELOG with the old value.

## 9. Tests

| Test | Guards |
|---|---|
| Precedence table: every layer pair, `--why` golden | §2, rule 4 |
| `TestConfig_ProjectCannotWidenTrust` for each trust-widening key | §2, rule 5 |
| `TestReload_InvalidKeepsLastGood` (syntax error, unknown key, bad reference) | §5, spec §19.2 |
| `TestReload_RunKeepsSnapshot`: agent registry edited mid-run | invariant 10 |
| `TestReload_ShrinkPoolNoKill` | §5 |
| Fuzz: random edits to a valid `init.lua`; daemon never stops, snapshot always valid or last-good | rule 2 |
| Golden `settings.lua` writer output | §6 |

## 10. Sources

| Decision | Follows |
|---|---|
| Separate config evaluation from use; immutable snapshot; validate before apply | [SRW ch. 14] |
| One typed registry with defaults, ranges and help | [SRW ch. 15] |
| Precedence defaults < files < env < flags | [TAOUP ch. 10] |
| Project config can't widen trust | [BSRS ch. 5] |
| Last-good on failed reload | [SRW ch. 14] [RI ch. 10] |
| Aliases and deprecation for renamed settings | [SEG ch. 15] |
