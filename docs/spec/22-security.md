## 22. Security and trust

| Area | Rule |
|---|---|
| **Pipeline trust** | Pipelines in your user dir (`~/.config/mill/pipelines/`) are trusted. Project pipelines are trusted once the project is: mill can't tell your repo from a fresh clone, so a project's pipelines stay **disarmed** until `mill trust .` (triggers never armed, manual runs refused with `M9101`, the board shows `◌ not trusted · triggers off · mill trust .`). Pipelines from elsewhere (registry, downloaded, imported) need `mill trust <file>`. Trust shows what will run (commands, parts, runners, network, secrets) and records each file's hash in `trust.json` (§20); any change to a file needs trust again. |
| **Plugin pinning** | `mill.lock` pins versions and SHA-256 hashes (§9.11). A changed binary refuses to start until re-approved (`M5301`). In v0.5 the registry is a GitHub index whose entries carry archive hashes (§9.6); signed packages (Sigstore, keyless) come with the hosted registry. |
| **Plugin permissions** | Every plugin declares network hosts, filesystem access and destructive capabilities, shown at install. Enforced for Lua and WASM parts; enforced for executables when `plugins.sandbox = "container"` (§9.6). |
| **Project Lua config** | A repo's `.mill/mill.lua` runs data-only until `mill trust .`. Hooks, commands and inline parts in untrusted project config are ignored (§19.5). |
| **Exports** | `mill export` blocks anything that looks like a secret and flags machine-specific paths. Run history is never included unless asked, and then redacted (§9.9). |
| **Isolation** | Code-changing nodes use a workspace by default. Writing to the main checkout needs `workspace: none` plus a confirmation on the first run. Untrusted pipelines can be forced onto the `container` runner by policy. |
| **Destructive actions** | Parts flag capabilities as `destructive` (push, merge, send, delete, pay), and a node can mark itself `destructive: true` (§7.3). A pipeline can only use them if `policy.allow_destructive` lists them, or `mill lint` fails. `mill run --dry-run` runs `sh` and `exec` steps for real unless they're marked destructive, stubs agents, and has destructive parts report what they would have done (§17.10). |
| **Paths** | Workspace roots, file parts and `trusted_dirs` refuse the filesystem root, the home folder itself, relative paths and `..`. |
| **Network listeners** | The gateway is off by default and binds to localhost when on. Webhooks need a per-trigger token or signature check. Public exposure only through a tunnel or the hosted relay. |
| **Sandboxes** | CEL is side-effect free. Lua is sandboxed by default. Container runner offers `network: none`. WASM parts (later) are sandboxed by design. |
| **Logs and examples** | Example pipelines and test fixtures use synthetic data only. Never real names, emails, paths or tokens. |


---

[← spec index](index.md)
