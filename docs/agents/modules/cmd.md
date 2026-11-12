# cmd

**Path:** `cmd/mill`, `cmd/milld`  
**Status:** planned · first built in M0 (`cmd/mill`), M2 (`cmd/milld`)  
**Backend plan path:** `cmd/mill`, `cmd/milld` (milld calls `daemon/boot.Run`)  
**Depends on:** cli, server, config, everything wired at boot  
**Depended on by:** none

## Purpose

Entrypoints. `mill` with no arguments opens the TUI; with a subcommand it dispatches to the CLI. `milld` parses flags, takes the daemon lock, loads config, recovers unfinished runs, arms triggers and serves the socket.

## Owns

- Flag parsing for `milld` and version printing (`-version`).
- Thin `main` functions: no logic beyond dispatch and wiring.

## Invariants

- `main` stays thin; wiring lives in a boot package so it can be tested.
- Binary version, wire protocol version, parts protocol version and pipeline format version are four separate constants.

## Spec

- [§5.1 Processes](../../spec/05-architecture.md#51-processes)
- [§17.1 Commands](../../spec/17-cli.md#171-commands)

When code exists, rewrite this file from the source: entry points, public surface, internal structure, side effects, tests, gotchas.
