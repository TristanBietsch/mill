# Changelog

All notable changes to mill are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

No code yet; mill is at the spec stage. The first code entries arrive with the weekend prototype (`mill run` over `sh` and headless `agent` nodes).

### Changed

- docs: spec v2.1 applies the 2026-11-11 decisions (backend D1–D30, editor 1–30, spec issues S1–S31; ADR-024 to ADR-044): primary-upstream edges (`M3021`, unreachable moved to `M3024`), fresh clones disarmed until `mill trust .`, built-in part ids win, allowlisted step environment, default retries as new attempts, dry runs that run `sh`/`exec` and stub agents, crash semantics for `mill daemon stop`, GitHub registry index with SHA-256 hashes, board placement for crashed/cancelled/untrusted pipelines, backend package layout, Go 1.27, module `github.com/TristanBietsch/mill`, Homebrew tap `tristanbietsch/tap/mill`, MIT `LICENSE`, and one-page `docs/design.md` and `docs/tech.md`.
