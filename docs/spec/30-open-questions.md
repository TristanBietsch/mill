## 30. Open questions

1. **Item streams everywhere, or single values by default?** The spec uses streams (best fit for Unix pipes). Check that simple pipelines don't feel heavier because of it (`nodes.x.item` as the shorthand helps).
2. **Built-in handoff targets:** which ship with v1 (GitHub comment and issue, Slack message, email draft, an external session manager adapter), and which are left to plugins?
3. **How far to take keyboard graph editing** before saying "press `E` and edit the YAML"? **Resolved 2026-11-11:** as far as §18.7 goes. The editor plan's open decisions are all decided (`docs/plans/editor.md` §5) and applied in §18.7.
4. **Parts protocol transport:** stdio only, or also a socket for parts that run elsewhere (a part on another host)? Leaning stdio-only, with remote work going through runners.
5. **WASM parts:** worth the complexity before v1, or wait for demand?
6. **Pipe expression branching:** stay strictly linear, or add a small `tee`/`join` syntax?
7. **Startup voice:** record the final "mill" voice (distinct from rex's) and decide whether each built-in soundset gets its own take or all share one.
8. **Live-run view name:** "nilo" (after the Nilometer, the ancient flood gauge) is a candidate.
9. **Plugin registry:** run our own, or start with GitHub as the registry? **Resolved 2026-11-11:** GitHub first. v0.5 uses `gh:` sources plus a curated index repo with archive hashes, and the lockfile pins SHA-256s; a hosted registry comes later (§9.6, §25).
10. **Signatures:** Sigstore or minisign? **Resolved 2026-11-11:** Sigstore (keyless), later, with the hosted registry. Until then integrity comes from the index and lockfile hashes (§9.6, §22).
11. **Lua version and API stability:** freeze the `mill.*` API at v1 with deprecation warnings, the way editor plugin APIs do?
12. **Name check:** confirm `mill` and `milld` are free as a Go module path, Homebrew formula and GitHub org/repo. **Resolved 2026-11-11:** the name stays `mill`. Repo and module `github.com/TristanBietsch/mill`; Homebrew core already has an unrelated `mill`, so mill ships from its own tap, `tristanbietsch/tap/mill` (§27.1).


---

[← spec index](index.md)
