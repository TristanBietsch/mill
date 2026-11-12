# workspace

**Path:** `internal/daemon/workspace`  
**Status:** planned · first built in M1 (temp), M3 (git worktrees)  
**Backend plan path:** `internal/workspace`  
**Depends on:** paths  
**Depended on by:** runner, builtin (`git.*`)

## Purpose

Creates and cleans up git worktrees, temp dirs and volumes, and enforces path safety.

## Owns

- Worktree per run or per item, from a base ref.
- Cleanup policies: `on_success`, `always`, `never`, `keep: 7d`.
- Path rules.

## Invariants

- Refuses the filesystem root, the home folder itself, relative paths and `..`.
- Code-changing nodes get a workspace by default.

## Spec

- [§6.4 Workspaces](../../spec/06-data-model.md#64-workspaces)
- [§22 Security](../../spec/22-security.md#22-security-and-trust)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
