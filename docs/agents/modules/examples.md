# examples

**Path:** `examples/`  
**Status:** planned · first built in M0 (`hello`), grows each milestone  
**Backend plan path:** same  
**Depends on:** the `echo` agent, recorded HTTP fixtures  
**Depended on by:** `make test-e2e`

## Purpose

Example pipelines that ship with mill and double as the end-to-end test suite.

## Owns

- The twelve pipelines and one-liners from spec §26.
- Fixtures: synthetic only.

## Invariants

- Every example runs against the `echo` agent and recorded HTTP; no real accounts.

## Spec

- [§26 Example pipelines](../../spec/26-examples.md#26-example-pipelines)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
