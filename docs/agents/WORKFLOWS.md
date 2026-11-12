# Workflows

How an agent takes a task from plan to merged PR, and the commands involved. No `Makefile` exists yet; the targets are the plan from [../plans/backend/tech-stack.md](../plans/backend/tech-stack.md) §4.

## Taking a task

1. **Read the plan.** Find the task in its plan under [../plans/](../plans/README.md). Read the plan's header, global constraints, Decisions table and the spec sections it cites. If the task has no plan and is bigger than one small commit, write one first (below).
2. **Check for conflicts.** Re-read [HANDOFF.md](HANDOFF.md) §5. If the task touches a file another agent owns, stop and ask.
3. **Make a worktree.** One branch and one worktree per task, from the current default branch:
   ```sh
   git worktree add ../mill-<task-id> -b <type>/<task-id>-<slug>
   ```
   Example branch: `feat/M0-3-sh-part`. Never work in another agent's worktree or the shared main checkout.
4. **TDD.** Write the failing test, watch it fail for the right reason, write the code, watch it pass. One logical change per commit, Conventional Commits, signed.
5. **Gate.** `make check` green. If coverage rose, raise the floor. If you added an error code, its explain page and golden tests are in the same commit.
6. **Docs.** Update the module doc, the user doc (if the feature ships to users), `CHANGELOG.md` `[Unreleased]`, and the plan's task checkbox. If behavior departs from the spec, say so in the PR and propose the spec edit; do not edit `docs/spec/` unless the user asked.
7. **Independent review.** Ask a separate agent (or person) that did not write the change to review the diff against the plan, the spec and [AGENTS.md](AGENTS.md#never). Fix what it finds. Never review and merge your own PR.
8. **PR.** Title in Conventional Commits form. Body: what and why, the plan task id, test evidence, spec deviations, review outcome. Synthetic data only.
9. **After merge.** Remove the worktree. Update [HANDOFF.md](HANDOFF.md) §2/§5 if the project state changed.

## Writing a plan

Full rules: [../plans/README.md](../plans/README.md). In short:

1. Pick the id and file name from the naming table there.
2. Start with the header (goal, architecture, spec links, start condition), then global constraints, review focus, decisions, then tasks.
3. Each task: id, files to create or modify, interfaces produced, tests (named), acceptance, size.
4. Questions for the user go in the Decisions table with a recommended default. Do not build past an unanswered decision that changes the design.
5. Get the plan reviewed by a separate agent (`00-review.md` for a multi-file plan, or a Review section) before tasks start.

## Commands (planned)

| Command | What it does |
|---------|--------------|
| `make build` | `mill` and `milld` into the repo root |
| `make test` / `make test-race` | All tests, without / with `-race` |
| `make test-e2e` | `examples/` end to end with `--local` and the echo agent |
| `make lint` / `make lint-windows` | golangci-lint, host and `GOOS=windows` |
| `make cover` / `make cover-check` | Coverage and floors (`tools/coverage/baseline.txt`) |
| `make vuln` | govulncheck |
| `make errors-check` | Every M-code has a page and golden test, and the reverse |
| `make layering` | Package import rules |
| `make fuzz` | Fuzz targets for `FUZZTIME` |
| `make check` | The gate: all of the above except fuzz and e2e |

Parallel lint across worktrees can fail on golangci-lint's lock; run it with `--allow-parallel-runners` (as in rex).

## Running mill during development (planned)

```sh
make build
./mill run examples/hello.mill.yaml --local --json | jq .
./milld -version
```

Use a throwaway `HOME` (or `MILL_CONFIG_DIR`, `MILL_DATA_DIR`, `MILL_SOCKET`) so dev runs never touch your real mill state.

## Docs-only changes

Docs-only PRs follow the same flow without the Go gate. Before opening one: every relative link resolves (to a file, or to a path in the spec split map), and the voice matches [CONVENTIONS.md](CONVENTIONS.md#documentation-voice).
