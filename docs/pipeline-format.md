# Pipeline file format

The `*.mill.yaml` file: top-level keys, node keys, edges, expressions, items and validation.

Status: **not written yet.** Written when the feature ships: v0 ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §7](spec/07-pipeline-format.md#7-pipeline-file-format): where pipelines live, keys, edge inference, full example, layout sidecar, `mill lint`
- [spec §8](spec/08-expressions.md#8-expressions-and-templating): `${{ }}` CEL expressions and helpers
- [spec §6](spec/06-data-model.md#6-data-model-items-artifacts-workspaces): items, node modes, artifacts, workspaces

When written, this page covers: every key with type, default and an example; lint rules with their M-codes.
