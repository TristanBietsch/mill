# Tech stack

Every dependency mill's backend needs, pinned, with the reason it's there. Versions were checked against `proxy.golang.org` on 2026-11-11. Pin exact versions in `go.mod`; bump on purpose, never as a side effect.

Related: [architecture.md](architecture.md) for where each library is used, [layers.md](layers.md) §8.1 for which layer may import which module (the `ext` allowlists), [open-decisions.md](open-decisions.md) for the choices that still need a yes. Few, pinned, deliberately chosen dependencies [SEG ch. 21]. Citations: [references.md](references.md); sources table in §7.

## 1. Language and toolchain

| Item | Choice | Notes |
|---|---|---|
| Go | `go 1.27` with `toolchain go1.27.4` | Current stable (go1.27.4). rex pins `go 1.26.11`; mill starts on 1.27 because there's no legacy and 1.27 is what `govulncheck` will expect next. Raise the patch floor promptly when a stdlib advisory lands (rex did this for GO-2026-6859). |
| Module path | `github.com/TristanBietsch/mill` | Public repo of the same name, same owner as rex. Name check resolved (spec §30 item 12): the name stays `mill`; Homebrew ships from the own tap `tristanbietsch/tap/mill` because core Homebrew has an unrelated Scala `mill`. |
| CGO | `CGO_ENABLED=0` for every target, enforced in CI | Possible now that both hard cases are pure Go: `modernc.org/sqlite` and `oto/v3` ≥ 3.5 (Linux no longer needs cgo or ALSA headers at build time; it `dlopen`s `libasound.so.2` at runtime and degrades to silence without it). rex's CI still installs `libasound2-dev` because it's on oto 3.4.0; mill skips that. |
| Targets | `darwin/arm64`, `darwin/amd64`, `linux/amd64`, `linux/arm64`, `windows/amd64`, `windows/arm64` | All six built in CI. Test tiers per OS in [testing.md](testing.md). |
| Std features relied on | `os.Root` (traversal-safe file access, Go 1.24+), `log/slog`, `iter`, `context.AfterFunc`, `slices`/`maps`, `testing/synctest` (fake-time tests, stable since Go 1.25) | `os.Root` is the backbone of path safety for `file.*` parts and artifact materialization ([security.md](security.md)). |

## 2. Runtime dependencies

Grouped by area. "Where" names the package from [architecture.md](architecture.md).

### Core engine

| Area | Module | Version | License | Where | Why | Rejected |
|---|---|---|---|---|---|---|
| Expressions | `github.com/google/cel-go` | v0.30.0 | Apache-2.0 | `expr` | Reference CEL: side-effect free, cost-limited, typed. Spec §5.3 decided. | `expr-lang/expr` (no cost model, less portable spec); Lua for expressions (not terminating). |
| Lua | `github.com/yuin/gopher-lua` | v1.1.2 | MIT | `luaconf`, `luaapi`, `parts/builtin/lua` | Pure Go, `SetContext` for deadlines, `SkipOpenLibs` for sandboxes. Same lib and patterns as rex (`internal/features/lua`). | `Shopify/go-lua` (unmaintained), `aarzilli/golua` (cgo). |
| YAML | `go.yaml.in/yaml/v4` | v4.0.0 | Apache-2.0 | `pipeline`, `pipeline/yamledit` | Maintained successor of `gopkg.in/yaml.v3` (archived). v3 is frozen to security fixes; v4 is where fixes land. `yaml.Node` with line/column and comments. | `gopkg.in/yaml.v3` (archived, spec §27.2 names it); `goccy/go-yaml` v1.19.2 (good AST, but a second YAML dialect to learn and re-emits formatting too). See note on lossless edits below. |
| JSON Schema | `github.com/santhosh-tekuri/jsonschema/v6` | v6.0.3 | Apache-2.0 | `schema` | Draft 2020-12, precise error locations (instance + schema pointer) that map cleanly onto §29 diagnostics. Validates part `with`, node `output`, manifests. | `google/jsonschema-go` v0.4.3 (comes in transitively with the MCP SDK; weaker error output). Revisit if one library is wanted. |
| jq | `github.com/itchyny/gojq` | v0.12.20 | MIT | `parts/builtin/data` | The `jq` part with no external jq. | Shelling out to `jq` (not always installed). |
| IDs | `crypto/rand` (stdlib) | n/a | BSD-3 | `core/ids` | Run ids are 128-bit random hex; see [storage.md](storage.md) §2 for why not UUIDv7/ULID. | `google/uuid` v1.6.0, `oklog/ulid/v2` v2.1.2 (time-prefixed ids break short-id prefix matching). |
| Semver | `github.com/Masterminds/semver/v3` | v3.5.0 | MIT | `plugins` | Ranges like `^1.2`, `>=0.5 <2` (spec §9.5). | `golang.org/x/mod/semver` (no ranges). |
| Concurrency helpers | `golang.org/x/sync` | v0.25.0 | BSD-3 | engine, parthost | `errgroup`, `singleflight` (compile cache). | |
| Globs | `github.com/bmatcuk/doublestar/v4` | v4.10.4 | MIT | `file.glob`, `fswatch`, `ignore` rules | `**` globs, works on `fs.FS` and `os.Root`. | `filepath.Match` (no `**`). |

### Storage

| Area | Module | Version | License | Where | Why | Rejected |
|---|---|---|---|---|---|---|
| SQLite | `modernc.org/sqlite` | v1.62.0 | BSD-3 | `store/index` | Pure Go (CGO-free), WAL, mature, used widely. The index is a cache, so driver maturity beats speed. | `ncruces/go-sqlite3` v0.37.0 (wazero-based; fine, but adds a WASM runtime and slower cold start); `mattn/go-sqlite3` (cgo). |
| File locks | own code copied from rex `internal/daemon/state/lock_{unix,windows}.go` | n/a | ours | `store` | `flock` / `LockFileEx`, already tested on three OSes. | `gofrs/flock` v0.13.2 (fine; not needed). |
| Compression (later) | `github.com/klauspost/compress` | v1.20.4 | BSD-3 / Apache-2.0 | `store` retention | zstd for archiving old run logs (`retention.compress_after`). Not needed before M8. | gzip (3–5× bigger for logs). |

### Processes, terminals, files

| Area | Module | Version | License | Where | Why | Rejected |
|---|---|---|---|---|---|---|
| PTY (Unix) | `github.com/creack/pty` | v1.1.24 | MIT | `runner/pty` | Same as rex. | |
| PTY (Windows) | `github.com/charmbracelet/x/xpty` | v0.1.4 | MIT | `runner/pty` | ConPTY wrapper rex already uses (`console_windows.go`). | Raw `CreatePseudoConsole` calls (more code). |
| Terminal emulator | rex's fork of `hinshun/vt10x` in `third_party/vt10x` | in-tree | MIT | `runner/pty`, attach | Screen mirror for terminal-only agents: reading prompts, M6304 detection, attach repaint. rex already fixed scrollback, CSI markers and repaint. | `charmbracelet/x/vt` v0.1.0 (newer, maintained; re-evaluate at M6 when the PTY runner lands). |
| OS calls | `golang.org/x/sys` | v0.50.0 | BSD-3 | `proc` | Job Objects, `CTRL_BREAK`, `prctl`, peer creds. | |
| Terminal | `golang.org/x/term` | v0.48.0 | BSD-3 | `surface/cli` | TTY detection, raw mode for `attach`. | |
| File watch | `github.com/fsnotify/fsnotify` | v1.10.1 | BSD-3 | `trigger/fswatch`, config reload, pipeline reload | Standard. Debounce is ours. | Polling (wasteful). |
| Cron | `github.com/adhocore/gronx` | v1.20.6 | MIT | `trigger/cron` | Zero-dep parser plus `NextTickAfter`. mill owns the timer loop (catch-up, jitter, tz, fake clock), so only parsing and next-time math are needed. `@every` is parsed by mill. | `robfig/cron/v3` (last release 2020); `netresearch/go-cron` v0.19.0 (maintained fork, but brings its own scheduler we'd bypass). |
| Shell words | none | | | `pipe` parser | The pipe-expression grammar is small and has `${{ }}` and quoting rules of its own; a hand-written lexer gives better diagnostics. | `mattn/go-shellwords`, `google/shlex` (unmaintained). |

### Network and integrations

| Area | Module | Version | License | Where | Why | Rejected |
|---|---|---|---|---|---|---|
| MCP | `github.com/modelcontextprotocol/go-sdk` | v1.8.0 | MIT/Apache-2.0 mix (GitHub reports NOASSERTION; check the LICENSE file at pin time) | `parts/builtin/mcp`, `surface/mcpserver` | Official SDK, client and server, stdio and streamable HTTP. | `mark3labs/mcp-go` (community). |
| WebSocket | `github.com/coder/websocket` | v1.8.15 | ISC | `gateway` | Small, context-aware, maintained (ex-nhooyr). | `gorilla/websocket` (heavier API). |
| HTML parsing | `github.com/PuerkitoBio/goquery` | v1.13.0 | BSD-3 | `parse` part | CSS selectors (spec §10.2). CSV, XML, JSON are stdlib. | |
| HTTP | `net/http` (stdlib) | | | `http` part, gateway, Vault backend | Spec §27.2. No SDKs for Vault or 1Password: Vault over its HTTP API, 1Password via the `op` CLI. | `hashicorp/vault/api` (dependency tree); `1password/onepassword-sdk-go` (embeds a WASM core). |

### Secrets and signing

| Area | Module | Version | License | Where | Why | Rejected |
|---|---|---|---|---|---|---|
| OS keychain | `github.com/zalando/go-keyring` | v0.2.8 | MIT | `secrets/keychain` | macOS via `/usr/bin/security`, Linux via Secret Service (godbus), Windows via wincred, all CGO-free. **Check before use:** how macOS `Set` passes the value (argv vs stdin). If it's argv, `mill secrets set` must use `security -i` with the command on stdin instead; reading is fine. | `99designs/keyring` (bigger, file backends we don't need). |
| age files | `filippo.io/age` | v1.3.2 | BSD-3 | `secrets/agefile` | The headless-server backend (spec §21). | sops (too broad). |
| Signatures | deferred | | | `plugins` | Not needed until the hosted registry (D27 decided: GitHub index + lockfile hashes in v0.5, Sigstore keyless later). Candidates were: `github.com/sigstore/sigstore-go` v1.4.0 (Apache-2.0, keyless) or `aead.dev/minisign` v0.3.0 (MIT, offline). Spec §30 item 10. | |

### Surface (client side, listed so the module is complete)

| Area | Module | Version | License | Notes |
|---|---|---|---|---|
| TUI | `charm.land/bubbletea/v2` | v2.1.1 | MIT | rex is on v1.3.10 (maintenance only). mill starts on v2; the splash and board are ported, not copied verbatim. Decision D2 (default accepted). |
| Styling | `charm.land/lipgloss/v2` | v2.1.0 | MIT | |
| Widgets | `charm.land/bubbles/v2` | v2.2.1 | MIT | |
| Width | `github.com/rivo/uniseg` | v0.4.7 | MIT | East-asian width for the 18-cell splash column, as in rex. |
| Audio | `github.com/ebitengine/oto/v3` | v3.5.1 | Apache-2.0 | Client only. Recipe synth (Appendix A) renders WAVs at `go generate` time; playback from memory. |

## 3. Test and tool dependencies

| Area | Module | Version | License | Use |
|---|---|---|---|---|
| Assertions | `github.com/stretchr/testify` | v1.12.1 | MIT | `require`/`assert` only; no suites. |
| Leaks | `go.uber.org/goleak` | v1.3.0 | MIT | Every package's `TestMain` via `testutil/testenv` (copied from rex). |
| CLI scripts | `github.com/rogpeppe/go-internal` | v1.16.0 | BSD-3 | `testscript` for CLI tests, `txtar` for golden archives. |
| HTTP fixtures | `gopkg.in/dnaeon/go-vcr.v4` | v4.0.7 | BSD-2 | Recorded HTTP for `http`, `gh` (API mode), MCP-over-HTTP tests. |
| Diffs | `github.com/google/go-cmp` | v0.7.0 | BSD-3 | Struct diffs in reducer and IR tests. |
| Lint | `github.com/golangci/golangci-lint/v2` | v2.15.0 | GPL-3.0 (tool only, run with `go run`, never linked) | rex pins v2.14.0; mill pins v2.15.0. |
| Vulns | `golang.org/x/vuln` | v1.9.0 | BSD-3 | `govulncheck` in the gate. rex pins v1.8.0. |
| Release | `github.com/goreleaser/goreleaser/v2` | v2.20.1 | MIT | Run via `go run` in the release workflow only. |
| Bench compare | `golang.org/x/perf/cmd/benchstat` | latest at use | BSD-3 | Developer tool; not in `go.mod`. |

Tools run through `go run <pkg>@<version>` from the `Makefile`, exactly like rex, so nobody needs a global install and the linter is built with the repo's toolchain.

## 4. Build, CI and release

| Piece | Plan | From rex |
|---|---|---|
| `Makefile` | Targets `build`, `test`, `test-race`, `test-e2e`, `vet`, `lint`, `lint-windows`, `vuln`, `cover`, `cover-check`, `fuzz`, `bench`, `bench-check`, `check`, `errors-check`, `layering`. `check` = vet + test-race + lint + lint-windows + cover-check + vuln + errors-check + layering. | Copy and rename. Add `errors-check` (every `M` code has an explain page and a golden test, spec §29.11) and `layering` (import rules, [architecture.md](architecture.md) §3). |
| `.golangci.yml` | rex's config as the base: same linter set (`bodyclose contextcheck copyloopvar durationcheck errcheck errorlint exhaustive gocritic gosec govet ineffassign misspell nilerr noctx revive staticcheck unparam unused whitespace`, gofmt + goimports). Add `depguard` for layering and `forbidigo` banning (exact rules: [layers.md](layers.md) §8.2–§8.3) `fmt.Print*`/`os.Stdout` outside `surface/cli/core` (rex enforces this with a test; a linter rule is cheaper). Keep gosec G204/G304 excluded with the same rationale. | Copy, drop rex-specific path exclusions. |
| CI | GitHub Actions: Linux runs `make check`; macOS runs `vet test-race lint vuln`; Windows runs `vet test` plus a smoke (`mill run examples/hello --local`, then daemon start, run, stop). `CGO_ENABLED=0` set at workflow level. Separate `fuzz.yml` (nightly, 5 min per target) and `stress.yml` (nightly, `-count=50 -race` on engine and store). | Copy `ci.yml`, `fuzz.yml`, `stress.yml`; drop the ALSA step. |
| Coverage floors | `tools/coverage/baseline.txt` + `check.sh` | Copy as is. |
| Bench budgets | `bench/budgets.txt` + `check_budgets.sh` | Copy; scenarios in [testing.md](testing.md) §7. |
| Release | GoReleaser v2: six targets, `CGO_ENABLED=0`, `-trimpath`, `-ldflags "-s -w -X …/internal/version.Version={{.Version}}"`, checksums, SBOM (syft via goreleaser), GitHub release, Homebrew tap `tristanbietsch/tap` (formula `mill`), Scoop bucket, `.deb`/`.rpm` via nfpm. Archives contain both `mill` and `milld`. Signing: cosign keyless on the checksum file in the release workflow (GitHub OIDC). | rex has no GoReleaser yet (install.sh/install.ps1); mill adds it. Keep an `install.sh` equivalent only as "download and verify checksum", never `curl \| sh` in docs. |
| Version | `internal/version.Version` (ldflags), `wire.ProtocolVersion`, `partproto.Version`, `pipeline.FormatVersion` are four separate constants. | Same separation rex documents in DECISIONS.md ("daemon version string v1"). |

## 5. Note on lossless YAML edits

Spec §17.8 promises "edited in place, keeping comments, key order and formatting". No Go YAML library round-trips formatting exactly: decoding to `yaml.Node` and encoding again keeps comments and order but normalizes indentation, quoting and blank lines. The plan ([architecture.md](architecture.md), package `pipeline/yamledit`):

1. Parse with `yaml/v4` into `yaml.Node` to get line and column for every key and value.
2. Turn an edit (`node set build mode=each`) into a **text splice**: replace the byte range of one scalar, or insert a new `key: value` line at the node's indentation after its last child.
3. Re-parse the result and compare the decoded spec with the intended one. If they differ, fall back to a full re-encode and warn.
4. Golden tests: a corpus of hand-formatted pipelines where each edit must change only the expected lines.

## 6. What mill reuses from rex

rex (`~/Documents/personal/dev/rex`) is the same author's Go client/daemon tool with the same architecture. Recommendation: **copy and diverge**, no shared module for now. Reasons, then the file list.

This is code replication, chosen over a shared library on purpose while both codebases move fast [SAHP ch. 8].

**Why not a shared module (`github.com/tristanbietsch/termkit` or similar):**
- rex is moving fast (19 PRs this round, uncommitted work in its tree). A shared module couples two release trains and makes every rex refactor a mill dependency bump.
- The overlap is small and stable: the JSONL codec is ~200 lines, daemonctl ~1.3k, paths ~0.5k. Copying costs an afternoon; a shared module costs a versioning policy.
- The payload types differ completely (sessions vs runs), so only the envelope and transport would be shared anyway.
- Revisit after mill M5 (v0.1): if the copied files haven't diverged, extract `wire/jsonl`, `daemonctl`, `paths` base, `testenv`, `golden` and `third_party/vt10x` into one module both import. Track drift with a header comment in each copied file: `// Copied from rex <commit> internal/wire/protocol/codec.go; keep fixes in sync.`

**Copy (adapt names, keep structure and tests):**

| rex source | mill destination | Changes |
|---|---|---|
| `internal/wire/protocol/codec.go`, `envelope.go`, `codec_test.go`, `codec_fuzz_test.go`, `codec_bench_test.go` | `internal/wire/jsonl` | Make it payload-agnostic; add optional envelope field `ctx` for parts multiplexing ([protocols.md](protocols.md)). Keep `MaxLineBytes = 16 MiB`, sticky `ErrLineTooLong`, resumable reads after deadline. |
| `internal/daemon/server/conn.go` (bounded per-connection queue, single writer, drop slow clients) | `internal/daemon/server/conn.go` | As is. |
| `internal/daemon/server/peercred*.go`, `socket_unix.go`, `socket_windows.go` | `internal/daemon/server` | As is. |
| `internal/daemon/server/server.go` accept loop, drain, `trackConn` | `internal/daemon/server/server.go` | Keep the loop; replace session handlers. |
| `internal/runtime/daemonctl/*` (find binary next to client, detached spawn with setsid / new process group, `Reachable`, `ErrExitedEarly`, `FindPID`) | `internal/daemonctl` | Rename binary, socket default. |
| `internal/runtime/paths/paths.go` | `internal/paths` | New layout (spec §20), honors `XDG_*` and `MILL_*` overrides (rex deliberately doesn't; mill has no legacy). |
| `internal/daemon/state/lock*.go` | `internal/lock` (L0; was `internal/store/lock`, D33) | Used for `daemon.lock` and per-run `run.lock`. |
| `internal/runtime/rexlog/*` | `internal/mlog` | `slog` to file, 0600 perms. |
| `internal/runtime/redact/redact.go` (argv redaction for logs) | `internal/redact/argv.go` (L0; was `internal/secrets/redact`, D33) | Kept for logging command lines; the value redactor is new. |
| `internal/daemon/pty/console_unix.go`, `console_windows.go`, `repaint.go`, `modes.go`, `sanitize.go`, `transcript_pause.go` | `internal/runner/pty` | Supervisor loop is new (node-run lifecycle, not sessions). |
| `third_party/vt10x` (fork with scrollback hook and CSI marker fix) | `third_party/vt10x` | As is, with `replace` in `go.mod`. |
| `internal/daemon/adapter` ideas (regex classification, hooks file) | `internal/agents/detect` | Rewrite against mill's registry fields; reuse test fixtures style. |
| `internal/catalog/registry/builtin.yaml` agent entries (claude, codex, …, echo) | `internal/agents/builtin.lua` | Translate to Lua tables (spec §11.1); keep `trust_prompt` regexes. |
| `internal/catalog/settings/types.go` (`Setting`, `ApplyMode`, `Normalize`, aliases) and `trust_dirs.go` | `internal/config/settings` | Store becomes `settings.lua`, not YAML. `trust_dirs` checks reused for `trusted_dirs`. |
| `internal/features/lua/runtime.go`, `dispatcher.go` (mutex-serialized `LState`, deadlines, bounded coalescing queue) | `internal/luaapi` | Basis for hooks; sandbox options added. |
| `internal/surface/cli/core/*` (output kit: formats, `Printer`, views, `Errorf`, `Confirm`, selectors, `suggest.go`, golden-test harness, `lint_test.go`) | `internal/surface/cli/core` | Error type becomes mill's `errs.Error` (M-codes, classes); exit codes per spec §17.2. |
| `internal/testutil/testenv`, `golden`, `testperm` | `internal/testutil/...` | As is. |
| `internal/features/audio/synth.go`, `audio.go` (silent fallback `Player`) | `internal/surface/sound` | Recipes come from `soundsets.json` (Appendix A), not Go catalogs. |
| `Makefile`, `.golangci.yml`, `.github/workflows/*`, `tools/coverage/*`, `bench/check_budgets.sh` | repo root | See §4. |
| `docs/agents/` structure (HANDOFF, ARCHITECTURE, CONVENTIONS, DECISIONS, modules/) | `docs/agents/` | Same shape, written fresh. |

**New in mill (no rex counterpart):** pipeline parser/lint/compiler, CEL layer, engine (run actor, scheduler, slots, budgets, retries, cache), event-sourced run store and SQLite index, artifact CAS, parts protocol and part host, built-in parts, runners other than PTY, workspace manager (git worktrees), secrets broker and value redactor, trigger host, plugins/lockfile, gateway, MCP client/server, M-code error system.

## 7. Sources

| Decision | Section | Follows |
|---|---|---|
| Few pinned dependencies, bumped on purpose | §2–3 | [SEG ch. 21] |
| Third-party modules allowed per layer | §2 | [CA ch. 14] (stable layers depend on little); [layers.md](layers.md) §8.1 |
| Tools via `go run` pinned versions, linter in CI | §3–4 | [SEG ch. 20] [SEG ch. 23] |
| Independent version constants per format | §4 | [RI ch. 14] [DDIA ch. 4] |
| Copy and diverge from rex, revisit after M5 | §6 | [SAHP ch. 8] |
| Text-splice YAML edits with golden tests | §5 | [TAOUP ch. 5] (text stays the user's) |
