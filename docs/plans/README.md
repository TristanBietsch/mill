# Plans

A plan says how to build part of the spec: scope, order, decisions, tasks. The spec says what mill is; a plan never overrides it. A plan may propose a spec change, which applies only once the user accepts it and the spec is edited.

## Current plans

| Plan | Scope | Status | Owner |
|------|-------|--------|-------|
| [backend/00-overview.md](backend/00-overview.md) | `milld`, engine, storage, protocols, execution, security, testing; milestones M0–M11 | Accepted 2026-11-11; all 30 decisions decided, spec issues applied (spec v2.1) | another agent |
| [editor.md](editor.md) | TUI pipeline editor | Accepted 2026-11-11; all 30 decisions decided, applied in spec §18.7 | another agent |

Backend plan files: [milestones](backend/milestones.md) · [architecture](backend/architecture.md) · [engine](backend/engine.md) · [storage](backend/storage.md) · [protocols](backend/protocols.md) · [execution](backend/execution.md) · [security](backend/security.md) · [testing](backend/testing.md) · [tech-stack](backend/tech-stack.md) · [open-decisions](backend/open-decisions.md).

Update this table when you add a plan or its status changes.

## Naming and numbering

Adapted from rex's `docs/plans/` (one id per plan, `00-` files that rule a folder, a Decisions table per plan).

| Kind | File | Example |
|------|------|---------|
| Area plan, one file | `<area>.md` | `editor.md` |
| Area plan, several files | `<area>/00-overview.md` plus one file per topic | `backend/00-overview.md`, `backend/storage.md` |
| Milestone task plan | `<area>/<ID>-<slug>.md` | `backend/M0-weekend-prototype.md` |
| Review of a plan or folder | `<area>/00-review.md`, or `<ID>-review.md` for one plan | `backend/00-review.md` |
| Findings after a spike or milestone | `<area>/<id>-findings.md` | `backend/m0-findings.md` (named by the backend plan) |
| Spike (research with a go/no-go) | `<area>/<ID>-<slug>-spike.md` | `backend/M9-wasm-parts-spike.md` |

**Ids.**

| Prefix | Used for | Defined in |
|--------|----------|------------|
| `M0`–`M11` | Backend milestones | [backend/milestones.md](backend/milestones.md) |
| `E<n>` | Editor work | [editor.md](editor.md) (if it numbers tasks, its own scheme wins) |
| `T<n>` | Other TUI work (board, splash, settings, sound) | first TUI plan |
| `<ID>-<n>` | A task inside a plan | `M0-3`, `E2-1` |

Reserved, never used as plan ids: `D<n>` (decisions inside a plan), `S<n>` (spec issues in the backend plan), `ADR-NNN` ([../agents/DECISIONS.md](../agents/DECISIONS.md)), `M` + four digits (error codes). Cite a decision with its plan outside that plan: "backend D9".

Slugs are lowercase kebab-case, short. Ids are never reused; a dropped plan keeps its id and says `Status: dropped`.

## Authority inside plans

1. The spec wins over every plan.
2. Inside a folder, `00-overview.md` wins over its topic files, and topic files win over task plans.
3. A newer plan does not silently override an older one. It says `supersedes <file>` in its header, and the older one gets `Status: superseded by <file>`.
4. Once code exists, the code wins; fix the plan or mark it done.

## Plan format

Every plan starts with:

```md
# <ID or area>: <title>

Status: draft | review | accepted | in progress | done | dropped | superseded. Date: YYYY-MM-DD.
Spec: links to the sections this plan implements.
```

An area plan then has: goals, files (for a folder), build order, key decisions, spec issues found, open decisions. The backend plan is the model.

A task plan follows rex's task-plan shape:

```md
> **For agentic workers:** work through this plan task by task (subagent-driven or inline). Steps use checkboxes (`- [ ]`).

**Goal:** one paragraph.
**Architecture:** packages touched and how they fit.
**Spec:** links. **Start condition:** what must be merged first.

## Global constraints
Limits, caps, never-dos, test rules (hermetic, synthetic data, fake clock), gate.

## Review focus
3–5 risky cases the independent reviewer must check, each with the test that covers it.

## Decisions
| # | Decision | Default (recommended first) | Alternatives | Answer |

### Task <ID>-1: <title>
- [ ] **Files:** create / modify, with paths.
- [ ] **Interfaces produced:** Go signatures other tasks rely on.
- [ ] **Tests:** named tests, written first.
- [ ] **Acceptance:** what passes when it is done.
Size: S (≤ 3 d) · M (4–8 d) · L (9–15 d).
```

## Decisions

- Every question for the user is a row in the plan's Decisions table: the default the plan assumes (recommended first), one or two alternatives, the trade-off in one line.
- The user's answer goes in the Answer column with a date. Do not build past an unanswered decision that changes the design.
- An answer that changes the spec is applied to the spec by whoever the user asks, in its own commit.

## Reviews

Every plan is reviewed by a separate agent before its tasks start. The review checks claims against the spec (and code, once it exists), lists contradictions with a fix for each, and lists decisions that need the user. rex's `00-review.md` is the model.

## Progress

- Task checkboxes in the task plan are the progress record.
- [../agents/HANDOFF.md](../agents/HANDOFF.md) §2 and §5 summarize state across plans.
- `TODO.md` at the repo root is the user's list. Tick a line there only when the user asks or the whole item is done.
