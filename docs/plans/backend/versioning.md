# Versioning and compatibility

Status: review (2026-11-12 refinement, awaiting the user); D38 in [open-decisions.md](open-decisions.md). Spec: [§2](../../spec/02-principles.md#2-principles) principle 12 (additive evolution), [§16](../../spec/16-wire-protocol.md#16-daemon-wire-protocol), [§9.3](../../spec/09-parts-and-plugins.md#93-parts-protocol-v1).

Every surface another program or another mill version reads, how each is versioned, and the rules for changing it. Formats live in L1 (`core/event`, `partproto`, `wire/protocol`) so the published languages are in the most stable layer ([layers.md](layers.md)) [DDD ch. 14].

## 1. Rules

1. **Additive by default** (principle 12) [DDIA ch. 4]: new optional fields, new message types, new event types and new `features` never bump a version. Readers ignore unknown fields and unknown message types (tolerant reader), writers never remove or repurpose a field.
2. **A meaning change bumps the version** of that one surface. Each surface has its own constant; they never move together by accident (tech-stack.md §4).
3. **Backward first:** a newer reader reads everything older writers wrote. Forward compatibility (older reader, newer data) holds for additive changes and fails loudly, with a code, for version bumps [DDIA ch. 4].
4. **Contracts are tested from the consumer side** [SAHP ch. 13]: golden fixtures per version for every format, and the parts conformance suite as a consumer-driven contract for every part.
5. **Anything scripts can see is a contract** [SEG ch. 1, Hyrum's Law]: `--json` output fields, exit codes, error codes and classes, event shapes. They follow the same additive rules.
6. **Deprecate before removing** [SEG ch. 15]: warn with a code and a replacement for at least one minor release, then remove with a CHANGELOG entry.

## 2. Versioned surfaces

| Surface | Marker | Layer | Current | On a meaning change |
|---|---|---|---|---|
| Daemon wire protocol | envelope `v` | L1 `wire/protocol` | 1 | daemon speaks v and v−1 for one minor release, dispatching per line ([protocols.md](protocols.md) §2.7) [RI ch. 14] |
| Parts protocol | envelope `v`; manifest `protocol` | L1 `partproto` | 1 | host speaks v and v−1; `Describe` says which a part speaks; the SDK pins one |
| Event log | `log_v` in `RunCreated` | L1 `core/event` | 1 | new decoder added; old decoders kept forever (D38) |
| Pipeline file | `mill: 1` | L2 `pipeline` | 1 | `mill lint --fix` rewrites; old version accepted with a warning for one minor |
| Item envelope | implicit (fields) | L1 `core` | — | additive only |
| Index schema | `meta.schema_version` | L3 `store/index` | 1 | rebuild ([data-lifecycle.md](data-lifecycle.md) §6) |
| Lockfile, trust records, trigger state | `lock_v`, `trust_v`, `state_v` | L5 | 1 | one-way upgrader with `.bak` |
| Settings | registry ids | L5 `config/settings` | — | aliases ([configuration.md](configuration.md) §8) |
| Lua API | `mill.api_version` | L5 `luaapi` | 1 | old functions kept as deprecated wrappers |
| Error codes | `M` + 4 digits | L0 `errs` | — | never reused or renumbered; a new meaning gets a new code (spec §29.4) |
| CLI `--json` output | field names | L7 | — | additive only |
| Binaries | semver `0.x.y` (`internal/version`) | L0 | 0.0.0 | until 1.0, a minor may change behavior with CHANGELOG notes; formats still follow §1 |

## 3. Compatibility matrix

| Pair | Older ↔ newer | Behavior |
|---|---|---|
| client ↔ daemon | client newer than daemon | `Hello` compares versions; M7003 `daemon is older than this client` with `help: mill daemon restart --drain` (spec §29.9). The client never restarts the daemon by itself |
| client ↔ daemon | client older than daemon | works; the daemon answers in the client's `v` (one minor back) and features the client didn't list are not used |
| daemon ↔ part | part speaks an older `v` | supported for one minor after a bump; then M5102-style refusal naming both versions |
| daemon ↔ part | part newer | refused with M5102 naming the needed mill version |
| daemon ↔ event log | log older | always readable (D38) |
| daemon ↔ event log | log newer (after a downgrade) | that run only: M7007 `run <id> was written by a newer mill`; it's left untouched, not recovered, not pruned |
| daemon ↔ index | any mismatch | rebuild |
| mill ↔ pipeline file | `mill: 2` file on a `mill: 1` binary | M3001-style error `unsupported mill version 2` with the needed version |
| SDK ↔ mill | SDK pins a protocol `v` | conformance suite published per `v` (M9) |

## 4. Upgrades and downgrades

- **Upgrade:** install new binaries, `mill daemon restart --drain` (waits for live node-runs up to `shutdown.drain_timeout`) [RI ch. 13]. Without `--drain`, live node-runs follow the crash rules (ADR-042). The new daemon reads all old logs and rebuilds the index if its schema changed.
- **Downgrade:** supported for runs written by the older version; runs written with a newer `log_v` are refused one by one (M7007) and kept intact for the next upgrade.
- **Mixed `--local` and daemon versions:** a `--local` run writes `mill_version` and `log_v` in `RunCreated`; a daemon reading it applies the matrix above.

## 5. How a change ships

1. Add the field or type in L1 with a golden fixture.
2. Readers ignore it until they need it; writers fill it.
3. If the change alters meaning: bump the surface's constant, add the new decoder or dispatcher, keep the old one, add fixtures for both, and a CHANGELOG entry under `Changed`.
4. Deprecations: a warning code, a replacement in `help:`, a removal milestone in the CHANGELOG [SEG ch. 15].

## 6. Tests

| Test | Guards |
|---|---|
| Golden fixtures per version for events, wire messages, part messages, lockfile, trust records | §2 |
| `TestTolerantReader_*`: every decoder ignores an unknown field and an unknown type | rule 1 |
| `TestWire_VMinusOne` once v2 exists | §2 |
| Conformance suite run against an SDK built for the previous `v` | rule 4 |
| `TestEventLog_NewerLogV_Refused` | §3, M7007 |
| `--json` output goldens for every command | rule 5 |

## 7. Sources

| Decision | Follows |
|---|---|
| Additive evolution, tolerant readers, backward then forward compatibility | [DDIA ch. 4] |
| Each protocol versioned independently; speak v and v−1 | [RI ch. 14] |
| Drain-and-restart upgrades | [RI ch. 13] |
| Consumer-side contract tests | [SAHP ch. 13] |
| Visible outputs are contracts | [SEG ch. 1, Hyrum's Law] |
| Deprecation policy | [SEG ch. 15] |
| Published languages in the most stable layer | [DDD ch. 14] [CA ch. 14] |
| Versioned file formats | [DBI ch. 3] |
