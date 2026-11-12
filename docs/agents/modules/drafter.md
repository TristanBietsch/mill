# drafter

**Path:** `internal/drafter` (layer 2, services; added 2026-11-12, ADR-047)  
**Status:** planned · first built in v0.3 (with the wizard)  
**Spec v2 path (superseded):** none (new)  
**Depends on:** `pipeline` (parse, lint), `schema`, `errs`, `clock` | external: stdlib HTTP client. Settings arrive as a `Config` (`BaseURL`, `Model`, `Timeout`, `Backend`, `Agent`) built by `daemon/boot` from `drafter.*`, as in rex  
**Depended on by:** `daemon/server` (`DraftPipeline` intent), `daemon/boot` (wiring, health probe), `surface/cli/local` (`mill new --describe` without a daemon)

## Purpose

Turns one sentence into a pipeline proposal for the wizard's *describe it* and `mill new --describe`. Modeled on rex's summarizer (`rex/internal/features/summarizer`): a small Ollama HTTP client, a model resolver, a health probe, an untrusted-input prompt builder and an output guard. It runs in milld (principle 4); the TUI asks for a draft with `DraftPipeline` and shows the reply.

## Backends (`drafter.backend`)

| Backend | What runs | Cost | Fallback |
|---------|-----------|------|----------|
| `ollama` (default) | `POST {drafter.base_url}/api/generate` with `drafter.model`, JSON output, `drafter.timeout` per attempt | free, local | keyword rules + `M2212` |
| `agent` (opt-in) | a headless registry agent (`drafter.agent`) as a normal one-node run with a fixed output schema | tokens, shown and counted | keyword rules + `M2212` |
| `off` | keyword rules only | free | none needed |

`agent` needs the engine, which sits above this layer: `daemon/boot` injects an `AgentDrafter` (an interface defined here) backed by the engine, so `drafter` never imports `engine`.

## Owns

- `Client`: `/api/generate` and `/api/tags`, `transient` (unreachable or 5xx), one retry for transient errors only, never for a deadline or a 4xx.
- `ResolveModel(configured string, pulled []string) (string, bool)`: the configured model if pulled, else the first pulled model from `llama3.2:3b`, `qwen2.5:3b`, `gemma2:2b`, `llama3.2:1b`, `phi3:mini`; never an arbitrary (large) model. Substitution is logged.
- Health: probe at start and every 30 s (`/api/tags`, then a warm-up generate that loads the model); three failed calls in a row mark it unavailable until the probe passes. Flips go out as `DrafterHealth` events and into the `Snapshot`.
- `BuildPrompt(sentence, base)`: the step catalog (fetch, ws, work, scope, check, review, pr, notify), triggers, agents from the registry, and the sentence inside a delimited `<sentence>` block whose delimiter is defused (rex's `defuse`). The sentence is never logged.
- `Validate(reply) (Proposal, error)`: parse into the fixed `Proposal{Steps, Trigger, Agent, Effort, Name, Description, Why}` shape, keep only catalog steps and known triggers, build the document, run `pipeline` lint and the schema check. Anything that fails → `M2213` and the keyword drafter.
- `Keywords(sentence, base) Proposal`: the local rule drafter (the prototype's `wizPropose`): offline fallback and the test double.

## Invariants

- Optional never blocks (principle 10): every failure path ends in a keyword proposal plus a warning (`M2212` unavailable, `M2213` invalid), never an error that stops the wizard.
- The reply is untrusted: never executed, never shown before it validates; a reason string is display text only.
- One request in flight per daemon (a slot, as in rex); a new draft cancels the previous one.
- A `drafter.base_url` off this machine receives the sentence; the settings page and `mill doctor` say so.

## Errors

`M2212` drafter unavailable (unreachable, no usable model, timeout), warning. `M2213` the proposal failed parsing, schema or lint, warning. `M6101` only for the `agent` backend's CLI.

## Tests

HTTP fakes for `/api/generate` and `/api/tags`; golden prompts (`testdata/prompts`, `-update`); table tests for `ResolveModel`, `Validate` (hostile replies: extra steps, shell in a reason, a delimiter in the sentence) and `Keywords`; a live eval behind an `ollama_eval` build tag, as in rex.

## Spec

- [§18.8 Wizards](../../spec/18-tui.md#188-wizards) (Drafter)
- [§19.6 Settings registry](../../spec/19-config-lua.md#196-settings-registry) (`drafter.*`)
- [§16.3 Intents](../../spec/16-wire-protocol.md#163-intents) (`DraftPipeline`)
- [§29.10 `mill doctor`](../../spec/29-errors.md#2910-mill-doctor)
- Plan: [../../plans/wizard.md](../../plans/wizard.md) §6

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
