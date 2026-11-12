# Security policy

## Reporting a vulnerability

Report vulnerabilities privately through GitHub Security Advisories: open the repository's **Security** tab and choose **Report a vulnerability**.

Do not open a public issue, pull request or discussion for a suspected vulnerability. Include the mill version (`mill version`, once it exists), your OS, and steps to reproduce.

## Supported versions

mill has no release yet. It has one maintainer. Fixes will land on the default branch and ship in the next release. Older versions get no backports.

| Version | Supported |
|---------|-----------|
| default branch (latest) | Yes |
| Anything older | No |

## Threat model

This is the design from [spec §22](docs/spec/22-security.md#22-security-and-trust) and [the backend security plan](docs/plans/backend/security.md). It becomes a description of shipped behavior as code lands. Where the two disagree, the code wins and this file gets fixed.

mill runs on one machine, as one user. `milld` runs pipelines: it starts agent CLIs, shell commands and plugin processes, and writes run history to disk. `mill` (CLI and TUI) talks to it over a local socket. An optional HTTP gateway is off by default and binds to localhost when on.

### In scope

- **The daemon socket.** A client that can connect can start any pipeline, so it can run commands as you. The socket is `0600` in a `0700` directory, and the daemon checks the peer's uid. Reports of another user or process reaching it are in scope.
- **Secrets.** Pipelines must declare the secrets they use. A part gets only the secrets its node names, only through environment variables or the parts protocol, never on a command line. Every resolved value is redacted before anything is written to run logs, items, artifacts, events or client output. A secret value appearing in any of those, or on a process's argv, is in scope.
- **Trust.** Pipelines and project config from elsewhere (a cloned repo, an import, the registry) run nothing until you run `mill trust`. Trust is recorded by file hash; a changed file needs trust again. Anything that makes untrusted Lua run hooks, commands or inline parts, arms an untrusted trigger, or runs an untrusted pipeline is in scope.
- **Destructive actions.** Parts flag capabilities such as push, merge, send, delete or pay as `destructive`. A pipeline can use them only if `policy.allow_destructive` lists them. A destructive part running without that, or during `--dry-run`, is in scope.
- **Paths.** Workspace roots, file parts and `trusted_dirs` refuse the filesystem root, the home folder itself, relative paths and `..`. A way to read or write outside the allowed roots (path traversal, symlinks) is in scope.
- **Re-running side effects.** After a daemon crash, a node that was live and is not idempotent is marked `crashed` and never re-run on its own. Anything that silently runs it again is in scope.
- **Plugins.** `mill.lock` pins plugin versions and hashes. A changed binary refuses to start until re-approved. A way around the lockfile check is in scope.
- **Sandboxes.** CEL expressions have no side effects and are cost-limited. Lua transforms and Lua parts run sandboxed with deadlines. An escape, or a way to hang or crash the daemon from an expression or Lua part, is in scope.
- **The gateway.** Webhooks need a per-trigger token or signature. Forged or replayed deliveries that start runs are in scope.
- **Untrusted bytes reaching your terminal.** Step and agent output is shown in the TUI and CLI. Escape sequences that act on your terminal are in scope.
- **Exports and bug bundles.** `mill export` blocks secret-looking strings and flags machine-specific paths. `mill doctor --bundle` redacts and lists what it includes. A secret leaving through either is in scope.
- **Malformed protocol input** on the socket or a part's stdio that crashes or hangs the daemon.

### Out of scope

- **`~/.config/mill/init.lua` and your `lua/` modules are trusted user code**, like an editor config. Anyone who can edit them can already run code as you.
- What agent CLIs and the commands in your pipelines do inside their workspaces. Without the container runner, a step can reach the network and your home folder.
- Native executable plugins' declared permissions are shown but not enforced on the host unless `plugins.sandbox = "container"`.
- Pipelines and plugins you chose to trust.
- An attacker who already runs code as your user, or as root.
- Vulnerabilities in third-party dependencies that mill does not reach. Check reachability with `make vuln` (govulncheck) first.
