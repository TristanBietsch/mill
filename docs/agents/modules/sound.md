# sound

**Path:** `internal/surface/sound`  
**Status:** planned · first built in v0.2  
**Backend plan path:** same (synth and silent `Player` copied from rex)  
**Depends on:** settings  
**Depended on by:** tui

## Purpose

Soundsets, the event mixer, debounce, quiet hours and a silent fallback when there is no audio device.

## Owns

- Soundset loading (built-in, plugin, `mill.soundset`).
- Event → sound mapping and debounce.

## Invariants

- Audio failure never fails anything: no device means no sound, silently.

## Spec

- [§18.11 Sound](../../spec/18-tui.md#1811-sound)
- [Appendix A](../../spec/appendix-a-soundsets.md#appendix-a-reference-soundsets)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
