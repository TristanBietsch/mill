# tui

**Path:** `internal/surface/tui`  
**Status:** planned · first built in v0.2 (board), v0.3 (editor)  
**Backend plan path:** `internal/surface/tui` (outside the backend plan's scope)  
**Depends on:** wire/client, canvas, tui-settings, sound, settings registry  
**Depended on by:** cmd

## Purpose

The Bubble Tea client: boot splash, the board (FAILED, RUNNING, READY), the `λ` prompt, the run popup with timeline and log, view-only agent screens, the editor, wizards, `:plugins`, `:stats`, settings, screensaver.

## Owns

- Boot splash steps and failure screen.
- Board rows: five things per row, conditional columns, narrow-terminal rules.
- Focus modes and keymaps.
- Error display: statusline, `:messages`, lint quickfix list.

## Invariants

- Talks to the daemon only; never reads run directories.
- Never modal, never steals focus on errors.
- Color is state only; every row fills the width exactly and never wraps.
- Every screen maps to a CLI command.

## Spec

- [§18 TUI reference](../../spec/18-tui.md#18-tui-reference)
- [§18.1 Design rules](../../spec/18-tui.md#181-design-rules)
- [§29.8 Errors in the TUI](../../spec/29-errors.md#298-errors-in-the-tui)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
