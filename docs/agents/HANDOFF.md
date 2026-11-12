# Handoff: start here

For the next coding agent on mill. Read this first, then follow the onboarding order. Last updated 2026-11-11, at the spec stage (no code).

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
| Spec | v2 draft, one file, being split into `docs/spec/NN-*.md` (see §5) |
| Mockups | HTML mockup of the TUI in `mockups/v1/` |
| Backend plan | Written: [../plans/backend/00-overview.md](../plans/backend/00-overview.md), milestones M0–M11, 30 open decisions |
| Editor plan | In progress: [../plans/editor.md](../plans/editor.md) |
| Code | None. No `go.mod`, `Makefile`, CI or `examples/` |
| Git | Not checked: confirm the repo, default branch and remote with the user before your first commit |
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
| Editor plan `docs/plans/editor.md` | written | decisions open | Read only; its open decisions need the user |
| Backend plan `docs/plans/backend/` | another agent | written, decisions open | Read only; its open decisions need the user |
| Mockups `mockups/` | another agent | v2 sync to spec in progress | Read only; v1 is frozen |

Known spec issues are listed in the backend plan's [Spec issues](../plans/backend/00-overview.md#spec-issues) (S1–S31) with recommendations. They are not applied to the spec until the user accepts them.

## 6. Next milestone: the weekend prototype

From [spec §28](../spec/28-roadmap.md#28-roadmap), detailed as M0 in [../plans/backend/milestones.md](../plans/backend/milestones.md):

- No daemon. A single `mill run <file.mill.yaml> [--input k=v]... [--stdin]`.
- Parse a YAML subset (`mill`, `name`, `inputs`, `nodes` with `uses`, `with`, `in`, `outputs`), infer edges, topological sort, run nodes in order.
- Parts: `sh` (`io: plain|text`), headless `agent` (`claude`, `codex`, `echo`), `stdout`.
- `${{ }}` CEL for `inputs`, `item`, `items`, `nodes.x.item(s)`.
- JSONL items between steps; a run dir with `events.jsonl`, `items/`, `logs/`.
- Outputs as JSONL on stdout, progress on stderr, exit 0 or 1.

Done when `mill run hello.mill.yaml --json | jq .greeting` works and a `morning-brief-lite` pipeline runs from cron. M0 writes small versions of the real packages (`pipeline`, `expr`, `engine`, `store`), not throwaway code.

Before starting M0, confirm with the user: the package layout (spec §27.4 vs backend plan architecture §2), the Go version (D28), and the module path.

## 7. How to pick up work

1. Ask the user which task, or take the next unstarted task in the current plan.
2. Read the plan, its Decisions table and the spec sections it cites. If the plan and spec disagree, the spec wins unless the plan's decision was accepted.
3. Follow [WORKFLOWS.md](WORKFLOWS.md): worktree → TDD → gate → independent review → PR.
4. If a task has no plan and is bigger than one small commit, write the plan first ([../plans/README.md](../plans/README.md)) and get it reviewed.
5. Update this file's §2 and §5 when you finish something that changes the state.

## 8. Doc debt

- User docs under `docs/*.md` are stubs that point to the spec. Write each when its feature ships.
- `docs/design.md` and `docs/tech.md` exist and are empty. Ask the user whether to delete them or what they are for.
- Module docs under `modules/` describe planned packages. Once code exists, rewrite each from the source (rex's module doc shape: Path, Depends on, Entry points, Purpose, Public surface, Invariants, Gotchas).
