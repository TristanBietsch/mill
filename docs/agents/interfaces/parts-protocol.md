# Parts protocol (agents)

Contract summary of the protocol between `milld` and a part. Reference: **[spec §9.3](../../spec/09-parts-and-plugins.md#93-parts-protocol-v1)**. Message catalog, flow control and how built-ins honor it in-process: [../../plans/backend/protocols.md](../../plans/backend/protocols.md) §3. User doc (when shipped): [../../parts-protocol.md](../../parts-protocol.md). Code: [../modules/parthost.md](../modules/parthost.md), [../modules/builtin.md](../modules/builtin.md). Status: planned; public from v0.5.

## Transport

An executable part is started by `milld` with stdin and stdout connected. Same envelope as the daemon protocol: one JSON object per line, `{v, kind, type, id, ctx, data}`. Stderr is captured as the part's log. Lua and WASM parts implement the same lifecycle through function calls.

`kind` is `Intent` (milld → part) and `Event` (part → milld); the optional field `ctx` carries the context id for long-lived parts that handle many contexts (backend S10, applied in spec §9.3).

## Lifecycle

| Step | Direction | Message |
|------|-----------|---------|
| 1 | milld → part | `Describe` → part answers `Manifest` (id, version, capabilities; per capability: config schema, ports, output schema, flags) |
| 2 | milld → part | `Open` with context: resolved `with`, workspace, granted secrets, budget remaining, scratch dir |
| 3 | both | `Item` in; `Item` out with a `port`; milld sends `EndOfInput` |
| 4 | part → milld | `Log`, `Progress`, `Usage`, `Artifact`, `Checkpoint` while working |
| 5 | part → milld | `Done` or `Fail` (class, message, retryable) |
| 6 | milld → part | `Cancel` any time; part must `Fail` with class `cancelled` within the grace period or is killed |
| 7 | milld → part | `Close` ends the context; per-call parts exit |

Other messages: `Resume` (with last checkpoint, for `resumable` parts after a restart), `Ping`/`Pong` (health; no `Pong` → restart), `PartCall` (client-driven UI calls, e.g. list Slack channels), `Error` with code `unsupported`.

## Manifest flags

| Flag | Effect |
|------|--------|
| `streaming` | Supports `stream` mode |
| `cancellable` | Handles `Cancel` gracefully |
| `resumable` | Gets `Resume` after a crash |
| `idempotent` | May be re-run from the start after a crash |
| `destructive` | Needs `policy.allow_destructive`; skipped in `--dry-run` |
| `needs_secrets` | Receives granted secrets at `Open` |

## Rules

- No message lets a part ask a person anything. A part that cannot proceed fails with a clear error.
- A part reads only the secrets it was granted in `Open`.
- Unknown message types are ignored.
- Built-ins use this same contract; any built-in can be replaced by a part with the same id ([ADR-007](../DECISIONS.md#adr-007-built-ins-use-the-public-parts-protocol)).

## Resolution order

Built-ins → user-scope plugins → project-scope plugins → inline `mill.part` → `mill-part-<name>` on `PATH` → MCP tools as `mcp:<server>/<tool>`. Built-ins win: `PATH` executables and MCP tools only fill unused ids; overriding needs an explicit plugin or `mill.part` with `override = true` (backend S5, D19; applied in spec v2.1). See [§9.4](../../spec/09-parts-and-plugins.md#94-resolution-order).

## Versioning

`v` is the parts protocol version, separate from the wire version. Additive changes need no bump.
