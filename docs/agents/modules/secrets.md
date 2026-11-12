# secrets

**Path:** `internal/secrets` + `secrets/redact` (backend layout, accepted 2026-11-11)  
**Status:** planned · first built in M1 (env backend, redactor), M3 (keychain and others)  
**Spec v2 path (superseded):** `internal/daemon/secrets`  
**Depends on:** secrets backend parts  
**Depended on by:** sched, store, runner, server

## Purpose

Resolves secret references through backend parts, injects them only where declared, and registers every value with the redactor.

## Owns

- Backends in configured order: env, keychain, 1Password, Vault, age file.
- Streaming redaction (exact value plus common encodings).

## Invariants

- Values never on argv, never in logs, items, artifacts or events.
- A part gets only the secrets its node names, at `Open`.

## Spec

- [§21 Secrets](../../spec/21-secrets.md#21-secrets)
- [§5.4 invariant 9](../../spec/05-architecture.md#54-invariants)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
