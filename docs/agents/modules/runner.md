# runner

**Path:** `internal/runner` + `runner/proc`, `runner/local`, `runner/pty`, `runner/container`, `runner/ssh` (backend layout, accepted 2026-11-11)   · D33 (open, [layers.md](../../plans/backend/layers.md)) proposes moving `runner/proc` to `internal/proc` (L0) and the runner interface into `partproto`  
**Status:** planned · first built in M0 (local), M6 (pty), M9 (container, ssh), M10 (remote, external)  
**Spec v2 path (superseded):** `internal/daemon/runner`  
**Depends on:** workspace, secrets, paths  
**Depended on by:** builtin parts (through the `Open` context)

## Purpose

Executes node-runs in the right place: local process groups, PTYs with a screen mirror, containers, SSH, a remote `milld`, an external session manager.

## Owns

- Process groups (Unix) and Job Objects (Windows); kill and drain.
- PTY with view-only attach and repaint.
- Runner selection and per-runner caps.

## Invariants

- Every child is in its own process group and is killed on teardown.
- No input is ever sent to a PTY except `auto_answer` rules.

## Spec

- [§12 Runners](../../spec/12-runners.md#12-runners)
- [§11.4 Terminal agents](../../spec/11-agents.md#114-agents-that-need-a-terminal)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
