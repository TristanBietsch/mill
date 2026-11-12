# plugins

**Path:** `internal/plugins`  
**Status:** planned · first built in v0.5 (M9)  
**Backend plan path:** same  
**Depends on:** luaapi (data-only manifest), paths  
**Depended on by:** parthost, cli

## Purpose

Plugin resolution, install, permissions, lockfile, pack, and pipeline import/export.

## Owns

- `plugin.lua` manifest parsing in a data-only sandbox.
- Semver resolution, checksums, signatures.
- `mill.lock` read and write; `--frozen`.
- Export scanning for secrets and machine-specific paths.

## Invariants

- A changed binary refuses to start until re-approved.
- Reading a manifest never runs code.

## Spec

- [§9.5 Package format](../../spec/09-parts-and-plugins.md#95-plugin-package-format)
- [§9.6 Installing](../../spec/09-parts-and-plugins.md#96-installing-plugins)
- [§9.11 Lockfile](../../spec/09-parts-and-plugins.md#911-lockfile)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
