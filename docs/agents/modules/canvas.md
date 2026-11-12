# canvas

**Path:** `internal/surface/tui/canvas`  
**Status:** planned · first built in v0.3  
**Backend plan path:** `internal/surface/tui/canvas`; layout math in `pipeline/layout`  
**Depends on:** pipeline (layout)  
**Depended on by:** tui; `mill render` shares the layout engine

## Purpose

A small cell-grid renderer inside Bubble Tea for the pipeline graph: boxes, edges, zoom levels and mouse hit-testing.

## Owns

- Cell buffer and drawing primitives.
- Zoom levels.
- Hit-testing for mouse selection.

## Invariants

- Deterministic output for a given graph and size (shared with `mill render`).

## Spec

- [§18.7 Editor](../../spec/18-tui.md#187-editor)
- [§27.1 (why a custom canvas)](../../spec/27-tech-stack.md#271-language-go)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
