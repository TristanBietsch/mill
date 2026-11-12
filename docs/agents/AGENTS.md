# Agent hints

Router and rules for AI agents working in this repository.

## Where to start

| Task type | Start |
|-----------|--------|
| New agent: current state, open work | [HANDOFF.md](HANDOFF.md) |
| Navigation, "what is this?" | [README.md](README.md) |
| Layout, new file placement | [../STRUCTURE.md](../STRUCTURE.md) |
| Write or follow a plan | [../plans/README.md](../plans/README.md), [WORKFLOWS.md](WORKFLOWS.md) |
| Engine, storage, protocols | [../plans/backend/00-overview.md](../plans/backend/00-overview.md) |
| CLI | [interfaces/cli.md](interfaces/cli.md) |
| Pipeline format, lint | [interfaces/pipeline-format.md](interfaces/pipeline-format.md) |
| Wire protocol | [interfaces/wire.md](interfaces/wire.md), [data/schemas.md](data/schemas.md) |
| Parts protocol | [interfaces/parts-protocol.md](interfaces/parts-protocol.md) |
| Config, paths, env | [interfaces/config.md](interfaces/config.md) |
| Why something is the way it is | [DECISIONS.md](DECISIONS.md) |

## Authority

| Stage | Source | Role |
|-------|--------|------|
| Now | `docs/spec/` | The contract. Wins over plans and every other doc |
| Now | `docs/plans/` | How to build it. May propose spec changes; they apply only once the user accepts them |
| Later | Go source | Wins over everything, including the spec |
| Always | `docs/agents/` | A map. Can lag; check the spec or source before relying on it |
| Always | `docs/*.md` | User reference. Describes shipped behavior only |

When the spec and a plan disagree, follow the spec and raise the conflict with the user. Once code exists and the spec disagrees with it, trust the code, fix the spec in the same PR (or note it in the PR), and do not "fix" code to match the spec without the user's intent.

## Safe vs careful edits

| Safe (usual feature work) | Careful (ask or plan first) |
|---------------------------|-----------------------------|
| A new built-in part behind the public parts protocol | The parts protocol or wire protocol: version and additive rules ([DECISIONS.md](DECISIONS.md#adr-005-one-jsonl-envelope-for-the-daemon-and-parts)) |
| A new CLI subcommand or flag that follows [interfaces/cli.md](interfaces/cli.md) | The pipeline file format: changing a meaning needs a `mill:` version bump |
| A new setting in the settings registry | Exit codes, error codes and classes: never renumber or reuse |
| A new error code with page and golden test | The event log format and the state machine: replay of old runs must still work |
| Tests, docs, user doc stubs | Secrets broker, redactor, trust, path checks, Lua and CEL sandboxes |
| | `go.mod` version bumps, new dependencies (must stay CGO-free) |
| | Anything in `docs/spec/`, `docs/plans/backend/`, `docs/plans/editor.md`, `mockups/` (other owners) |

## Never

These come from the principles ([spec §2](../spec/02-principles.md#2-principles)) and the invariants ([spec §5.4](../spec/05-architecture.md#54-invariants)). Breaking one is a bug, whatever the task says.

1. **Never add anything that waits on a person.** No approval nodes, no "ask the user" parts, no reply or input intents, no prompts inside a run. Work that needs a human is a `handoff` that ends the branch. A terminal agent that asks a question fails with `M6304`.
2. **Never bypass commit signing** or any hook. If signing fails, retry once, then stop and ask.
3. **Never print, log or store a resolved secret value**: not in run logs, items, artifacts, events, errors, test output, PRs or commits. Secrets never go on a command line.
4. **Never re-run a non-idempotent step on a guess.** After a crash it is `crashed`; the user re-runs it with `mill rerun --from`.
5. **Never rewrite a run's event log.** Corrections are new events.
6. **Never change the state of a terminal node-run.** A retry is a new node-run.
7. **Never give built-ins private internals.** If a built-in needs something, the parts protocol needs it.
8. **Never write to the main checkout** from a code-changing node without `workspace: none` and its first-run confirmation.
9. **Never bind a listener beyond localhost by default.**
10. **Never use real data** (names, emails, paths, tokens, live output) in fixtures, examples, docs or PRs.
11. **Never edit files another agent owns** (see [HANDOFF.md](HANDOFF.md) §5).

## Commit and PR conventions

See [../../CONTRIBUTING.md](../../CONTRIBUTING.md): Conventional Commits, signed, `make check` before a PR, `CHANGELOG.md` under `[Unreleased]`, independent review. Step by step: [WORKFLOWS.md](WORKFLOWS.md).

## Known gaps

- No code. Module docs describe planned packages.
- The package layout is not final (spec §27.4 vs backend plan architecture §2).
- Spec issues S1–S31 in the backend plan are recommendations, not yet applied.
