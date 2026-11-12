## 30. Open questions

1. **Item streams everywhere, or single values by default?** The spec uses streams (best fit for Unix pipes). Check that simple pipelines don't feel heavier because of it (`nodes.x.item` as the shorthand helps).
2. **Built-in handoff targets:** which ship with v1 (GitHub comment and issue, Slack message, email draft, an external session manager adapter), and which are left to plugins?
3. **How far to take keyboard graph editing** before saying "press `E` and edit the YAML"? §18.7 now covers making, connecting, branching, looping, linting and testing from the keyboard; the decisions it still needs are listed in `docs/plans/editor.md` (Open decisions).
4. **Parts protocol transport:** stdio only, or also a socket for parts that run elsewhere (a part on another host)? Leaning stdio-only, with remote work going through runners.
5. **WASM parts:** worth the complexity before v1, or wait for demand?
6. **Pipe expression branching:** stay strictly linear, or add a small `tee`/`join` syntax?
7. **Startup voice:** record the final "mill" voice (distinct from rex's) and decide whether each built-in soundset gets its own take or all share one.
8. **Live-run view name:** "nilo" (after the Nilometer, the ancient flood gauge) is a candidate.
9. **Plugin registry:** run our own, or start with GitHub as the registry (`gh:` sources plus a curated index repo) and add a hosted registry later?
10. **Signatures:** Sigstore (keyless, tied to GitHub identity) or minisign keys? Sigstore is friendlier for publishers; minisign is simpler offline.
11. **Lua version and API stability:** freeze the `mill.*` API at v1 with deprecation warnings, the way editor plugin APIs do?
12. **Name check:** confirm `mill` and `milld` are free as a Go module path, Homebrew formula and GitHub org/repo before the first release.


---

[← spec index](index.md)
