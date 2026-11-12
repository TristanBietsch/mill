# Agent instructions

Coding agents working in this repository: start at [docs/agents/HANDOFF.md](docs/agents/HANDOFF.md). It has the onboarding order, current state, the gate and open work.

Short version:

- mill is at the spec stage. The spec ([docs/spec/index.md](docs/spec/index.md)) is the reference until code exists; then Go source wins and the spec gets fixed.
- Gate: `make check` before every PR. Conventional Commits, signed, never bypassed. Every PR gets an independent review.
- Never add a step that waits on a person. Never print or log secret values. Never re-run a non-idempotent step on a guess.
- Rules and authority: [docs/agents/AGENTS.md](docs/agents/AGENTS.md).
