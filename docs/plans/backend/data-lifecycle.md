# Data lifecycle

Status: accepted with the 2026-11-12 refinement; D37 (backups) and D38 (old log versions) in [open-decisions.md](open-decisions.md). Refines [storage.md](storage.md) §7 (retention), spec [§20](../../spec/20-paths.md#20-files-and-paths).

Every kind of data mill keeps: whether it's truth or derived, who owns it, how long it lives, how it's collected, how it's backed up and how its format changes. Layers per [layers.md](layers.md).

## 1. Data classes

Files are the truth; every database is a cache (principle 5) [DDIA ch. 11] [DDIA ch. 12].

| Data | Where | Truth or derived | Owner (layer) | Lifetime | Rebuildable from |
|---|---|---|---|---|---|
| Pipelines, prompts, project config | `~/.config/mill/pipelines/`, `<repo>/.mill/` | truth (user's, in git) | user; edits via `pipeline/edit` (L2) | user's | — |
| User config | `init.lua`, `settings.lua`, `lua/` | truth | user; `config` (L5) writes `settings.lua` | user's | — |
| Run event log | `runs/<id>/events.jsonl` | **truth** | run actor (L4) via `store` (L3) | `retention` | — |
| Frozen pipeline bundle, inputs, trigger event, outputs, checkpoints | `runs/<id>/pipeline/`, `inputs.json`, `event.json`, side files | truth | `store` (L3) | with the run | — |
| Items | `runs/<id>/items/` | truth (committed prefix) | `store` (L3) | with the run | — |
| Raw logs, transcripts | `runs/<id>/logs/`, `transcripts/` | truth (best effort, no fsync) | worker (L4) | with the run | — |
| Artifacts | `artifacts/sha256/` | truth, shared | `store/cas` (L3) | while referenced | — |
| `run.json` | `runs/<id>/run.json` | derived | `store` (L3) | with the run | its `events.jsonl` |
| Index | `index.db` (+ `-wal`, `-shm`) | derived (cache) | `store/index` (L3) | forever | all event logs |
| Cache entries | `cache/` | derived, reusable | `engine/cache` (L4) | TTL + LRU | re-running the node |
| Trigger state | `triggers/<pipeline>/<trigger>.json` | truth (fires, seen keys, queue) | `trigger` (L5) | bounded windows | — (lost state means a possible duplicate or missed fire, logged) |
| Trust records, lockfiles | `trust.json`, `mill.lock` | truth | `parthost`, `plugins` (L5) | until revoked or updated | — |
| Workspaces | `workspaces/` | scratch with value (agent edits) | `workspace` (L4) | per policy | — |
| Scratch, tmp | `runs/<id>/scratch/`, `artifacts/tmp/` | scratch | L3/L4 | node-run, then 24 h sweep | — |
| Daemon and client logs | `~/.local/state/mill/` | diagnostic | `mlog` (L0) | rotated | — |
| Status file, TUI state, editor swap | `~/.local/state/mill/` | derived / scratch | L6, L7 | overwritten | — |

## 2. Retention

| What | Policy | Mechanism |
|---|---|---|
| Finished runs | `retention` (30 d) by `finished_at`; `mill prune --older 30d`; never prune a run that's live, holds `run.lock`, or is the `parent` of a live run | daily job in `milld` (03:15 local + jitter) and on demand; deletes through the teardown path, then the dir; index rows cascade |
| Cache entries | `cache.ttl` (30 d since last hit), `cache.max_bytes` (5 GiB, LRU) | same job |
| Artifacts | mark-and-sweep (§3) | same job, after runs and cache |
| Workspaces | node policy (`on_success`, `always`, `never`, `keep: 7d`) + `keep_workspaces`; orphans whose run is terminal and past keep | workspace manager at run end + daily |
| Trigger state | seen keys bounded by `dedupe_window` and 10k keys; queued fires by `trigger.max_queued` | trigger host on every write |
| Logs per node-run | 256 MiB cap: first 16 MiB + rolling last 64 MiB, gap recorded as an event | log writer |
| `milld.log`, `tui.log`, `cli-errors.log` | 10 MiB × 5 files | `mlog` |
| `tmp/`, `scratch/` | anything older than 24 h | boot |
| Compressed logs (M8) | finished runs' logs older than 7 d recompressed with zstd (`*.log.zst`), read transparently | daily job |

Pruning a run whose items a newer run reused is safe: `rerun --from` copied (hardlinked) what it reused.

## 3. Garbage collection

Order: runs → cache → artifacts → workspaces, so each sweep sees the roots left by the previous one [DDIA ch. 3] [DBI ch. 7].

1. **Mark:** roots are `artifact_refs` of every remaining run, every cache entry's artifacts, and every artifact registered by a live run (read from the live run's actor, not the index, to avoid a stale-index race).
2. **Sweep:** delete blobs not marked **and** older than 1 h. The grace covers artifacts written by a node-run whose `ArtifactRegistered` event hasn't reached the index yet [SRE ch. 26].
3. **Dry run:** `mill prune --dry-run` prints runs, cache entries, blobs and workspaces that would go, with sizes.
4. **Never concurrent with itself:** one GC at a time (in-process mutex in `milld`; `mill prune` without a daemon takes `daemon.lock`).
5. **`VACUUM`** on `index.db` after a prune that removed > 10 % of rows.

## 4. Disk pressure

| Free space in the data dir | Behavior |
|---|---|
| < 5 GiB or < 10 % | `mill doctor` warns; board header shows `disk low` |
| < 1 GiB | the scheduler stops granting new node-runs (they stay `queued` with text `disk low`); retention runs at once with half the configured age; `M2002` warning on `ConfigStatus` |
| ENOSPC on a log write | rex's pausing transcript: drop bytes for 30 s, retry, record the gap |
| ENOSPC on an event append | the run actor stops appending and parks the run without a terminal event; recovery handles it on the next start or when space returns (`mill doctor` offers `mill daemon restart`) |

The rule follows [RI ch. 5, Steady State] and [BSRS ch. 8]: run out of space slowly and visibly, never corrupt the truth.

## 5. Backups and restore

Decision D37: no `mill backup` command in v0; mill makes the data dir safe for ordinary file backups (Time Machine, restic, rsync) and documents what to copy [SRE ch. 26].

- **Safe to copy while running:** every truth file is append-only (`events.jsonl`, item files) or written by atomic rename (everything else), and CAS blobs are immutable. A file-level copy taken mid-run restores as a crashed run, which recovery handles.
- **What to back up:** `~/.config/mill/`, `~/.local/share/mill/` except `index.db*`, `workspaces/`, `artifacts/tmp/` and `cache/` (all derived or scratch).
- **Restore:** copy back, start `milld`; a missing index is rebuilt (`mill doctor --reindex` is automatic when `index.db` is absent). Restore is tested, not assumed: an integration test copies a data dir mid-run, restores it elsewhere and checks recovery and rebuild.
- **Moving machines:** same procedure; absolute paths in run headers (`project_dir`) are informational and don't need to exist.

## 6. Format changes

| Data | Version marker | How it changes |
|---|---|---|
| Event log | `log_v` in `RunCreated` | additive fields need no bump; a meaning change bumps `log_v`. Logs are never rewritten (invariant 4); readers keep a decoder for every `log_v` ever shipped (D38). A log with a newer `log_v` than this mill knows is refused for that run only (M7007) |
| Index | `meta.schema_version` | **migration = rebuild** [BEA ch. 6]: a different schema version means rename aside and fold all logs into a new file. No `ALTER` |
| `run.json` | derived | rewritten from the log on any mismatch |
| Item files | `core.Item` envelope | additive only; `meta` gains fields |
| Trigger state, trust records, lockfile | `"state_v"`, `"trust_v"`, `"lock_v"` | additive; a meaning change ships a one-way upgrader that writes a new file atomically and keeps the old as `.bak` |
| `settings.lua` | registry aliases | renames handled by aliases ([configuration.md](configuration.md) §8) |
| Cache entries | key prefix `"v": 1` | a key-format change makes old entries unreachable; TTL collects them |

Full compatibility rules: [versioning.md](versioning.md).

## 7. Deletion and privacy

- `mill rm <run>` and prune go through teardown first (invariant 8), then remove the dir; artifacts are released to GC.
- Nothing mill persists contains a registered secret (invariant 9, [security.md](security.md) §2).
- `mill doctor --bundle` and `mill export` list what they include before writing (spec §29.10, §22).

## 8. Tests

| Test | Guards |
|---|---|
| `TestPrune_SkipsLiveAndParents` | §2 |
| `TestGC_GraceProtectsInFlight`: artifact written, index delayed, GC runs | §3 |
| `TestGC_RootsFromLiveActors` | §3 |
| `TestDiskLow_StopsGrants` with a fake free-space probe | §4 |
| `TestENOSPC_EventAppend_ParksRun` with a failing writer | §4 |
| `TestBackup_CopyMidRun_Restore` | §5 |
| `TestIndex_SchemaMismatch_Rebuilds` | §6 |
| `TestEventLog_AllVersionsDecode` (golden logs per `log_v`) | §6, D38 |

## 9. Sources

| Decision | Follows |
|---|---|
| Log as truth, everything else derived | [DDIA ch. 11] [DDIA ch. 12] |
| Append plus sweep, mark-and-sweep with grace | [DDIA ch. 3] [DBI ch. 7] [SRE ch. 26] |
| Migration = rebuild for derived data | [BEA ch. 6] |
| Backups by plain file copy; restore tested | [SRE ch. 26] |
| Bounded growth and visible disk pressure | [RI ch. 5, Steady State] [BSRS ch. 8] |
| Versioned file formats | [DBI ch. 3] [DDIA ch. 4] |
