# Changelog

All notable changes to mill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

No code yet; mill is at the spec stage. The first code entries arrive with the weekend prototype (`mill run` over `sh` and headless `agent` nodes).

### Changed

- docs: spec v2.3 (2026-11-12; ADR-055): settings, help and the new-pipeline wizard become full-screen pages following rex v2 (shared page frame, two panes, inspector and cross-section search in settings, help groups at full height with a filter, a step rail and graph preview for new pipeline); rules 1 and 10 rewritten; rex proposals that would change decided behavior are open questions 13–21.
- docs: spec v2.2 (2026-11-12; ADR-045 to ADR-053): four-step new-pipeline wizard (start › shape › run › review, name last, dry run then edit, modal `hjkl`/`i`/`esc` keys, file import with type detection, duplicates start manual, n8n waits become handoffs), a local-LLM drafter through Ollama configured in `:settings` (`drafter.*`), and one nvim-style `:` command line on the last row of every screen (`/` is search only; the `λ` prompt is replaced by `:run` and `:pipe`; a pending confirm swallows `:`). New plans: `docs/plans/wizard.md`, `docs/plans/command-line.md`.
- docs: spec v2.1 applies the 2026-11-11 decisions (backend D1–D30, editor 1–30, spec issues S1–S31; ADR-024 to ADR-044): primary-upstream edges (`M3021`, unreachable moved to `M3024`), fresh clones disarmed until `mill trust .`, built-in part ids win, allowlisted step environment, default retries as new attempts, dry runs that run `sh`/`exec` and stub agents, crash semantics for `mill daemon stop`, GitHub registry index with SHA-256 hashes, board placement for crashed/cancelled/untrusted pipelines, backend package layout, Go 1.27, module `github.com/TristanBietsch/mill`, Homebrew tap `tristanbietsch/tap/mill`, MIT `LICENSE`, and one-page `docs/design.md` and `docs/tech.md`.
