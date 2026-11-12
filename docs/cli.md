# CLI reference

Every `mill` command, its flags, exit codes, selectors and output modes.

Status: **not written yet.** Written when the feature ships: v0 (run, runs, logs, wait, items, rerun) and v0.1 (the full headless CLI) ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §17](spec/17-cli.md#17-cli-reference): commands, exit codes, selectors, conventions, output modes, `--local`, CI recipe
- [spec §29.6](spec/29-errors.md#296-error-classes): error classes and the exit code each maps to
- [spec §14](spec/14-piping.md#14-piping-and-unix-composability): `--stdin`, `--each`, pipe expressions

When written, this page covers: one section per command family, the exit code table, selectors, `--json` shapes, agent-friendly usage.
