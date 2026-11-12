# Handoff: start here

For the next coding agent on mill. Read this first, then follow the onboarding order. Last updated 2026-11-11, at the spec stage (no code), after the decisions were applied (spec v2.1).

## 1. Onboarding order

1. [../../README.md](../../README.md): what mill is, status.
2. [../spec/01-summary.md](../spec/01-summary.md) and [../spec/02-principles.md](../spec/02-principles.md): what mill is for and the 16 principles that decide arguments. Read the principles twice; they override everything below them.
3. [../../CONTRIBUTING.md](../../CONTRIBUTING.md): the gate, commits, PR checklist, independent review.
4. [AGENTS.md](AGENTS.md): authority order, safe vs careful edits, what never to do.
5. [README.md](README.md): module map and "where to start" per task.
6. [../STRUCTURE.md](../STRUCTURE.md): where docs and code go.
7. [../plans/README.md](../plans/README.md): how plans are named and written; then the plan for your task.
8. The spec section and module doc (`modules/*.md`) for the area you touch.

The spec is split: one file per section in `docs/spec/` (start at [../spec/index.md](../spec/index.md)). `spec.md` is only a pointer.

## 2. Current state

| Thing | State |
|-------|-------|
| Spec | **v2.1**, split into `docs/spec/NN-*.md` (start at [../spec/index.md](../spec/index.md)). The 2026-11-11 decisions are applied; see "v2.1" in the spec index |
| Decisions | All open decisions are decided: backend D1–D30, editor 1–30, spec issues S1–S31 applied. Recorded as ADR-024 to ADR-044 in [DECISIONS.md](DECISIONS.md) |
| Mockups | `mockups/v2/index.html` is the current TUI prototype; v1 is frozen. v2 lags v2.1 in two spots (footer says "provisional"; `M3201` on manual pipelines shows as an error) |
| Backend plan | Accepted: [../plans/backend/00-overview.md](../plans/backend/00-overview.md), milestones M0–M11 |
| Editor plan | Accepted: [../plans/editor.md](../plans/editor.md), phases E0–E7 |
| Overviews | [../design.md](../design.md) (product) and [../tech.md](../tech.md) (stack), one page each |
| Code | None. No `go.mod`, `Makefile`, CI or `examples/` |
| Git | Repo `github.com/TristanBietsch/mill` (public), branch `main`, remote `origin`. All mill work is committed there |
| License | MIT ([../../LICENSE](../../LICENSE)) |
| `TODO.md` | The user's list. Do not tick or edit lines unless asked |

## 3. The gate

```sh
make check   # vet, race tests, lint (host + windows), coverage floors, govulncheck, errors-check, layering
```

Run it before every PR. It does not exist yet: the first code PR creates `go.mod`, the `Makefile`, `.golangci.yml` and CI, copied from rex and adapted ([../plans/backend/tech-stack.md](../plans/backend/tech-stack.md) §4, §6). Until then, a docs-only PR has no gate beyond the link check in its own description.

If several worktrees lint at once, golangci-lint can fail on its lock. rex's fix applies: run it by hand with `--allow-parallel-runners`.

## 4. Conventions that matter

- **Conventional Commits:** `type(scope): summary` (see CONTRIBUTING.md).
- **Signed commits:** retry once if signing fails, then stop and ask. Never bypass it (no `--no-gpg-sign`, no config change).
- **Independent review:** every PR gets a review from a separate agent or person before merge. Do not review your own PR and merge it.
- **TDD:** a failing test first, then the code. Bug fixes come with a test that fails without the fix.
- **CHANGELOG:** user-visible changes get an entry under `[Unreleased]`.
- **Errors are interface:** every new error has an M-code, a class, a `help:` line, an explain page and a golden test ([../spec/29-errors.md](../spec/29-errors.md#291-rules-for-every-error)).
- **Hands-off:** nothing in mill waits on a person. No approve, reply or input intents, ever ([AGENTS.md](AGENTS.md#never)).
- **Synthetic data only:** never paste real run output, names, emails, tokens or personal paths into commits, PRs, fixtures or docs.
- **Windows:** expect test-only path bugs (`HOME` vs `USERPROFILE`, illegal filename characters, `t.TempDir()` cleanup while a child still runs). rex hit all three.

## 5. In flight

| Work | Owner | Status | Rule for you |
|------|-------|--------|--------------|
| Spec split into `docs/spec/*.md` | done | 2026-11-11 | Edit the section files; `spec.md` is a pointer |
| Decisions applied to spec (v2.1) | done | 2026-11-11 | The spec is the contract again; plans follow it |
| Editor plan `docs/plans/editor.md` | accepted | decisions decided | Follow it for editor work (E0–E7) |
| Backend plan `docs/plans/backend/` | accepted | decisions decided | Follow it for backend work (M0–M11) |
| Mockups `mockups/` | another agent | v2 current, small lag behind v2.1 | Read only; v1 is frozen |

## 6. Next milestone: M0, the weekend prototype

From [spec §28](../spec/28-roadmap.md#28-roadmap), detailed as M0 in [../plans/backend/milestones.md](../plans/backend/milestones.md):

- No daemon. A single `mill run <file.mill.yaml> [--input k=v]... [--stdin]`.
- Parse a YAML subset (`mill`, `name`, `inputs`, `nodes` with `uses`, `with`, `in`, `outputs`), infer edges (every reference orders; input from `in:` or the primary upstream, spec §7.4), topological sort, run nodes in order.
- Parts: `sh` (`io: plain|text`), headless `agent` (`claude`, `codex`, `echo`), `stdout`.
- `${{ }}` CEL for `inputs`, `item`, `items`, `nodes.x.item(s)`.
- JSONL items between steps; a run dir with `events.jsonl`, `items/`, `logs/`.
- Outputs as JSONL on stdout, progress on stderr, exit 0 or 1.

**Done when** `mill run hello.mill.yaml --json | jq .greeting` works and the first dogfood pipeline, **`nightly-vuln-sweep`** (lite: `sh` scanners → headless agent triage → markdown report), runs nightly from the user's OS crontab. `morning-brief` follows in M1, `issue-to-pr` around M4. M0 writes small versions of the real packages (`pipeline`, `expr`, `engine`, `store`), not throwaway code.

The three pre-M0 confirmations are answered (2026-11-11):

| Question | Answer |
|----------|--------|
| Package layout | The backend plan's ([../plans/backend/architecture.md](../plans/backend/architecture.md) §2), now spec §27.4 and [../STRUCTURE.md](../STRUCTURE.md). Module docs give it as the primary path |
| Go version | Go 1.27, `toolchain go1.27.4` (D28) |
| Module path | `github.com/TristanBietsch/mill` |

Code copied from rex gets a header naming the rex commit and path it came from (ADR-026).

## 7. How to pick up work

1. Ask the user which task, or take the next unstarted task in the current plan.
2. Read the plan, its Decisions table and the spec sections it cites. If the plan and spec disagree, the spec wins unless the plan's decision was accepted.
3. Follow [WORKFLOWS.md](WORKFLOWS.md): worktree → TDD → gate → independent review → PR.
4. If a task has no plan and is bigger than one small commit, write the plan first ([../plans/README.md](../plans/README.md)) and get it reviewed.
5. Update this file's §2 and §5 when you finish something that changes the state.

## 8. Doc debt

- User docs under `docs/*.md` are stubs that point to the spec. Write each when its feature ships.
- Module doc file names still use the spec v2 package names (`sched.md` is `engine`); rename them and add docs for the new packages (`core`, `errs`, `partproto`, `schema` …) as their code lands.
- `mockups/v2` still marks the backend-plan states as provisional and shows `M3201` as an error on manual pipelines; sync it on the next mockup pass.
- Module docs under `modules/` describe planned packages. Once code exists, rewrite each from the source (rex's module doc shape: Path, Depends on, Entry points, Purpose, Public surface, Invariants, Gotchas).
