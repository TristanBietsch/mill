# Contributing to mill

Docs start at [docs/index.md](docs/index.md). Coding agents start at [docs/agents/HANDOFF.md](docs/agents/HANDOFF.md).

mill is at the spec stage. There is no `go.mod` or `Makefile` yet. Everything below is the rule from the first commit of code on. The targets are planned in [docs/plans/backend/tech-stack.md](docs/plans/backend/tech-stack.md) §4 and [docs/plans/backend/testing.md](docs/plans/backend/testing.md) §9.

## Prerequisites

- Go 1.27 (`go.mod` pins `go 1.27` with `toolchain go1.27.4`; backend decision D28).
- `make`, a POSIX shell and `git`. Lint and vulnerability tools run through `go run` at pinned versions, so there is nothing else to install.
- `CGO_ENABLED=0` everywhere. A change that needs cgo needs a decision first.

## Make targets (planned)

| Target | What it does |
|--------|--------------|
| `make build` | Builds `mill` and `milld` into the repo root |
| `make test` | Unit, component and integration tests |
| `make test-race` | The same with `-race` |
| `make test-e2e` | Every pipeline in `examples/` end to end with `--local` against the echo agent |
| `make lint` | golangci-lint for the host OS |
| `make lint-windows` | golangci-lint for `GOOS=windows` |
| `make cover` / `make cover-check` | Coverage report / fail below a package's floor in `tools/coverage/baseline.txt` |
| `make fuzz` | Every `Fuzz*` target for `FUZZTIME` |
| `make vuln` | govulncheck |
| `make errors-check` | Every `M` code has an explain page and a golden test, and every page has a code |
| `make layering` | Import rules between packages hold |
| `make check` | **The gate:** vet, race tests, both lints, coverage floors, govulncheck, errors-check, layering |

Do not install mill over your own binaries while working on it unless you mean to.

## Test tiers (planned)

1. **Unit and component** (`make test-race`): pure functions, and one subsystem with fakes and a fake clock.
2. **Integration**: real processes, real `git` worktrees in temp repos, the daemon over a real socket.
3. **CLI scripts**: `testscript` over `mill` and `milld`.
4. **Examples** (`make test-e2e`): the pipelines in `examples/`, with the echo agent and recorded HTTP.
5. **Stress and fuzz**: nightly.

Tests must not touch your real mill state. Use `t.TempDir()` and set `HOME`, `XDG_*` and `MILL_*` for anything that resolves paths. Fixtures use synthetic data only: no real names, emails, paths or tokens.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/) subjects: `type(scope): summary`, for example `feat(engine): retry rate-limited agent nodes` or `docs(spec): fix status exit code`. Common types are `feat`, `fix`, `refactor`, `test`, `docs`, `chore` and `perf`. Keep the subject under about 72 characters. Use the body to explain why.

Commits are signed. If signing fails, retry once, then stop and ask. Never bypass it: no `--no-gpg-sign`, no `-c commit.gpgsign=false`, no config change.

Keep commits small and logical. Each one should build and pass tests.

## Pull requests

Before you open a PR:

- [ ] `make check` passes locally.
- [ ] New behavior has tests; bug fixes have a test that fails without the fix.
- [ ] Coverage floors still pass. If coverage rose, raise the floor.
- [ ] New or changed errors follow [spec §29.1](docs/spec/29-errors.md#291-rules-for-every-error), with a code, an explain page and a golden test.
- [ ] Docs match the change: the user doc under `docs/`, the module doc under `docs/agents/modules/`, and the spec if the change differs from it.
- [ ] `CHANGELOG.md` has an entry under `[Unreleased]` for user-visible changes.
- [ ] Changes to the socket, the parts protocol, secrets, trust, path checks or Lua sandboxes were checked against [SECURITY.md](SECURITY.md).

Every PR gets an independent review before merge: a separate reviewer (person or agent) that did not write the change. Do not review your own PR and merge it.

Report security issues privately as described in [SECURITY.md](SECURITY.md), not in a PR.

## License

mill is MIT licensed ([LICENSE](LICENSE)). By contributing you agree that your contribution is licensed under the same terms. Code copied from rex keeps a header naming the rex commit and path it came from ([spec §27.1](docs/spec/27-tech-stack.md#271-language-go)).
