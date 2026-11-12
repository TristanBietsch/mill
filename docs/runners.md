# Runners

Where a node-run executes: local, pty, container, ssh, remote, external.

Status: **not written yet.** Written when the feature ships: v0 (`local`), v0.5 (`container`, `ssh`), v0.6 (`remote`, `external`) ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §12](spec/12-runners.md#12-runners): runners, selection order, caps

When written, this page covers: config per runner, selection, `policy.allowed_runners`, isolation trade-offs.
