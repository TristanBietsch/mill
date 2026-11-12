# CLI (agents)

Contract summary. Reference: **[spec §17](../../spec/17-cli.md#17-cli-reference)**. User doc (when shipped): [../../cli.md](../../cli.md). Status: planned.

## Entry

`cmd/mill/main.go` → `internal/surface/cli` ([../modules/cli.md](../modules/cli.md)). No arguments → TUI.

## Command families

| Family | Commands | First milestone |
|--------|----------|-----------------|
| Run and inspect | `run`, `runs`, `logs`, `items`, `wait`, `rerun`, `replay`, `cancel`, `rm`, `prune` | M0–M2 |
| Status | `status`, `watch`, `events`, `prompt`, `stats`, `digest` | M2–M5 |
| Build and edit | `new`, `ls`, `show`, `render`, `lint`, `edit`, `node add/set/rm/mv`, `edge add/rm`, `pipe` | M1 (`lint`), M5 |
| Triggers | `trigger ls/enable/disable/fire`, `hooks install/uninstall` | M4–M7 |
| Parts and plugins | `plugin …`, `part ls/info`, `import`, `export`, `agent ls/test`, `mcp ls/serve`, `trust`, `lua-types` | M9 |
| Config and secrets | `config list/get/set/reset/edit/check`, `secrets set/ls/rm/test`, `cache ls/clear` | M1–M3 |
| Daemon and setup | `daemon start/stop/status/restart/logs`, `setup`, `doctor`, `update`, `uninstall`, `completion`, `explain` | M1–M5 |
| View-only | `attach <run>/<node>` | M6 |

Full synopsis: [spec §17.1](../../spec/17-cli.md#171-commands).

## Rules every command follows

| Rule | Source |
|------|--------|
| Data on stdout, diagnostics on stderr | [§17.11](../../spec/17-cli.md#1711-shell-and-terminal-integration) |
| Every listing takes `--json`; streaming commands emit JSONL | [§17.6](../../spec/17-cli.md#176-output-modes) |
| `--json` failure: error object on stdout, non-zero exit | [§29.3](../../spec/29-errors.md#293-json-output) |
| Destructive commands confirm in a terminal, need `--yes` otherwise (exit 6) | [§17.4](../../spec/17-cli.md#174-conventions) |
| `--dry-run` on every command that changes state | [§17.4](../../spec/17-cli.md#174-conventions) |
| Pretty only on a terminal; plain when piped or `NO_COLOR`; `MILL_ASCII=1` for words instead of glyphs | [§17.6](../../spec/17-cli.md#176-output-modes) |
| Columns stable, new ones only on the right; `--fields` picks | [§17.6](../../spec/17-cli.md#176-output-modes) |
| Edits go through the daemon with the file's hash; stale edit → `conflict` (exit 6) | [§17.8](../../spec/17-cli.md#178-building-and-editing-pipelines-from-the-cli) |
| `--local` runs one pipeline in-process, no daemon, no triggers | [§17.10](../../spec/17-cli.md#1710-no-daemon---local) |
| No approve or reply commands | [§17.9](../../spec/17-cli.md#179-running-and-inspecting-from-the-cli) |

## Piping

`--stdin` (or `-`) reads items: JSONL by default, `--text` for lines. `--each` starts one run per item. Outputs go to stdout as JSONL; `--wait` is the default when stdout is not a terminal; `--detach` prints the run id. See [§14.1](../../spec/14-piping.md#141-mill-as-a-filter).

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic error or the run failed |
| 2 | Selector not found |
| 3 | Ambiguous selector |
| 4 | Daemon unreachable |
| 5 | Invalid arguments or lint failure |
| 6 | Refused or conflict |
| 7 | Timed out |
| 8 | Reserved |
| 9 | Budget exceeded |

Derived from the error class ([§29.6](../../spec/29-errors.md#296-error-classes)). Known spec issues: `mill status` exit 1 wording (backend S3) and budget → 9 vs `cancelled` → 1 (backend S9).

## Selectors

Full id · `pipeline#42` · `pipeline#last` · short hex id (4+ digits, prefix) · `pipeline#42/node` · `…/node[3]` for an `each` item. Ambiguous → exit 3 listing every match. See [§17.3](../../spec/17-cli.md#173-selectors).

## Gotchas

- `mill status` is for prompts and scripts: one line, fast. `mill prompt` reads a status file and never touches the socket.
- `mill attach` is view-only; `Ctrl+]` leaves.
- Plan decision: no CLI framework; rex's output kit and stdlib `flag` (backend D5, open).
