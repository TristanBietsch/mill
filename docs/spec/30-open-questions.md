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



**From rex v2's pages (v2.3, ADR-055).** These rex v2 choices would change something mill already decided, or add something new, so they wait for a decision. Each has a recommended default (first) and the alternatives. Sources: `rex/docs/plans/r2/R04-fullscreen-settings-help.md`, `rex/docs/plans/r2/R16-new-agent-flow.md`.

13. **One compose screen instead of four steps?** rex's new-agent page has at most two screens and lands on one where the common path is `n`, type, `enter`. **Default:** keep the four steps (ADR-045), shown as the step rail (§18.8); `enter` already walks all the way through. Alternatives: fold shape and run into one scrolling pane after start; or rex's layout, with start, shape, run and review as sections of one pane.
14. **Remember the last choices?** rex starts from the agent, model and effort you launched last (R16 D3). **Default:** remember the agent, effort, budget preset and on-failure from the last create, in memory while the TUI runs, falling back to the ADR-050 defaults. Alternatives: always the ADR-050 defaults (today); remember them in the state folder across restarts.
15. **Create from any step?** rex launches with `ctrl+s` from any section. **Default:** no; `enter` walks to review and `enter` or `:w` creates there (review shows the files, estimate and lint first). Alternatives: `ctrl+s` and `:w` jump to review from any step; or create from any step with the defaults.
16. **Pastes and files as context for *describe it*?** rex turns large pastes and `@path` into context items for the agent, with caps and a sanitizer. **Default:** later; the sentence stays one field and files come in through import. Alternative: large pastes and `@path` become context items sent to the drafter (capped, sanitized, workspace files only, never through a shell).
17. **Keep the draft across restarts?** **Default:** no, memory only, as rex (R16 D4: a sentence may hold secrets). Alternative: save it under `~/.local/state/mill/` with mode `0600` and drop it after create.
18. **Accent color and bundled themes?** rex v2 adds `accent_color` (named colors, `#hex`, `rgb()`, `oklch()`, adjusted for contrast, with a swatch and live preview on the settings page) and bundled `dracula`, `tokyo_night`, `pantone_safe` themes. **Default:** don't adopt; color in mill means state only (rule 8), so Appearance keeps `theme` and `density`, and `mill.theme { accent = … }` (§19.3) never paints state. Alternatives: an `accent_color` setting with rex's parser and swatch, limited to the cursor and page titles; or read rex's accent so the pair matches.
19. **A flag for the old overlays?** rex keeps its overlays behind `fullscreen_pages` and `new_agent_page`. **Default:** no flag; mill has no overlay code to keep. Alternative: a `fullscreen_pages` setting, on by default.
20. **Plugins and stats as pages too?** **Default:** they stay popups for now (rex kept `:usage` a popup, R04 D10). Alternative: `:plugins` becomes a page (it already has views and a detail page); stats stays a popup.
21. **Help groups for the settings and new-pipeline pages?** **Default:** their keys sit under Modes with their own headings, so help keeps eight groups on `1`–`8` (§18.5.1). Alternative: two more groups, ten in all, with `0` for the tenth.

---

[← spec index](index.md)
