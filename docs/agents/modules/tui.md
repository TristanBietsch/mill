# tui

**Path:** `internal/surface/tui` (outside the backend plan's scope) (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in v0.2 (board), v0.3 (editor)  
**Spec v2 path (superseded):** `internal/surface/tui`  
**Depends on:** wire/client, canvas, tui-settings, sound, settings registry  
**Depended on by:** cmd

## Purpose

The Bubble Tea client: boot splash, the board (FAILED, RUNNING, READY), the `:` command line on the last row, the run popup with timeline and log, view-only agent screens, the editor, the wizards (new pipeline, setup), `:plugins`, `:stats`, settings, screensaver.

## Owns

- Boot splash steps and failure screen.
- Board rows: five things per row, conditional columns, narrow-terminal rules.
- Focus modes and keymaps.
- Error display: statusline, `:messages`, lint quickfix list.
- The command line (`internal/surface/tui/cmdline`, planned): the root model owns the last row (help bar, echo, `:` line, `/` search); a command registry (`Command{Name, Aliases, Context, Arg, CLI, Help, Run}`) feeds completion, `:help commands` and `mill help`; history in `~/.local/state/mill/cmdline_history`. A screen with a focused text field (`TextFocused() bool`) gets `:` as a colon; a pending confirm swallows it. Plan: [command-line.md](../../plans/command-line.md).
- The new-pipeline wizard (`internal/surface/tui/wizard`, planned): four steps over the editor's keymap modes; drafts through milld's `DraftPipeline` intent ([drafter](drafter.md)) and imports through milld. Plan: [wizard.md](../../plans/wizard.md).

## Invariants

- Talks to the daemon only; never reads run directories.
- Never modal, never steals focus on errors.
- Color is state only; every row fills the width exactly and never wraps.
- Every screen maps to a CLI command.
- `:` is never swallowed by a screen, only by a pending y/n confirm; INSERT and text fields type it.
- Never calls Ollama or reads import files itself: milld drafts and imports (principle 4).

## Spec

- [§18 TUI reference](../../spec/18-tui.md#18-tui-reference)
- [§18.1 Design rules](../../spec/18-tui.md#181-design-rules)
- [§18.5 Keys and mouse](../../spec/18-tui.md#185-keys-and-mouse) (command line)
- [§18.8 Wizards](../../spec/18-tui.md#188-wizards)
- [§29.8 Errors in the TUI](../../spec/29-errors.md#298-errors-in-the-tui)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
