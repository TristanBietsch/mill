# Glossary (agents)

Product terms (pipeline, node, part, item, run, node-run, states, …) are in **[../glossary.md](../glossary.md)**. Do not duplicate them here.

This file holds terms agents meet in plans and process, alphabetical.

## ADR-NNN

An entry in [DECISIONS.md](DECISIONS.md). Never renumbered; reversed only by a new entry.

## CAS

Content-addressed store: artifacts stored by SHA-256. The backend plan uses one global CAS with per-run references (backend plan S2, D14).

## conformance suite

Planned tests that run every built-in part through the real JSONL parts protocol and in-process, and require identical results. Keeps [ADR-007](DECISIONS.md#adr-007-built-ins-use-the-public-parts-protocol) honest.

## D-number

A decision inside one plan (`D9` in the backend plan). Cite it with the plan when outside it: "backend D9".

## echo agent

The fake agent built into the agent registry for tests. Every test that needs an agent uses it; no test calls a real agent.

## gate

`make check`. Must pass before every PR. See [../../CONTRIBUTING.md](../../CONTRIBUTING.md).

## handoff (two meanings)

The `handoff` part ends a branch by passing work to a person or tool ([../glossary.md](../glossary.md#handoff)). [HANDOFF.md](HANDOFF.md) is this repo's onboarding note. Say which one you mean.

## M-number (milestone)

A backend milestone, M0 (weekend prototype) to M11 (hardening), in [../plans/backend/milestones.md](../plans/backend/milestones.md). Not to be confused with M-codes (`M6304`, four digits), which are error codes.

## node-run id

`<node>@<attempt>` or `<node>[<i>]@<attempt>` (`build@2`, `label[3]@1`), from the backend storage plan.

## owner

The agent or person who currently edits a file. Files with another owner are read-only for you ([HANDOFF.md](HANDOFF.md) §5).

## primary upstream

Backend plan rule for edge inference: the node a step takes its items from when it references several nodes. Other references only order. Ambiguity is lint error `M3021` (backend D9, spec issue S7).

## reducer

The pure function that folds a run's `events.jsonl` into its current state, live and during recovery.

## run actor

The one goroutine per live run that is the only writer of that run's event log (backend plan).

## S-number

A spec issue found by the backend plan (S1–S31): contradiction, under-specified, or over-scoped. A recommendation until the user accepts it and the spec is edited.

## slot

A unit of capacity a node-run must hold to start: global, per-runner, per-agent, per-run. Released at the terminal state on every path (invariant 7).

## spec split

Moving `docs/spec/spec.md` into one file per section (`docs/spec/01-summary.md` …). Anchors stay the same. Done 2026-11-11.

## weekend prototype

Spec §28's first step and backend milestone M0: `mill run` over `sh` and headless `agent` nodes, no daemon.

## worktree

A git worktree per task, so parallel agents never share a checkout. See [WORKFLOWS.md](WORKFLOWS.md).
