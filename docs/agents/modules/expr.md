# expr

**Path:** `internal/expr`  
**Status:** planned · first built in M0  
**Backend plan path:** `internal/expr` (+ `internal/schema` for output shorthand)  
**Depends on:** cel-go  
**Depended on by:** pipeline, sched, builtin

## Purpose

The CEL environment for `${{ }}`: variables, helper functions, the template scanner, cost limits, and the opaque secret handle type.

## Owns

- Variables: `inputs`, `event`, `item`, `items`, `nodes.<id>`, `run`, `env`, `secrets`, `now`.
- Helpers: `json`, `yaml`, `text`, `lines`, `trim`, `slug`, `hash`, `glob`, `duration`, `template`, `default`.

## Invariants

- No side effects; always terminates (cost-limited).
- Secrets are opaque handles; values reach only parts.

## Spec

- [§8 Expressions](../../spec/08-expressions.md#8-expressions-and-templating)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
