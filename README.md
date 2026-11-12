# ≋ mill

mill (Machine Intelligence Logistics Layer) is a terminal-native pipeline engine for agents, code and automations. You describe a pipeline as a graph of steps: agent calls, shell commands, HTTP requests, MCP tools, checks, branches, loops. A local daemon runs it by hand, on a schedule, or when something happens.

Two binaries: `mill` (CLI and TUI) and `milld` (daemon). Pipelines are YAML files in git. Run history is an append-only event log. Every step is a filter: items in, items out, so mill pipes into `jq`, `grep` and other pipelines.

mill is hands-off. A run never waits on a person. Decisions inside a run are made by checks, judges and policies. Work that needs a human is handed off (an issue, a message, a ticket) and that branch ends.

It is built to sit next to [rex](https://github.com/tristanbietsch/rex): rex holds the agent sessions that need you, mill runs the work that needs no one. mill does not depend on rex.

## Status

**Specification stage. No code yet.** The spec and an HTML mockup exist. The next milestone is the weekend prototype: a single `mill run` that parses YAML, runs `sh` and headless `agent` nodes in topological order, and pipes JSONL between them ([roadmap](docs/spec/28-roadmap.md#28-roadmap)).

## What it will look like

Planned, not built:

```sh
mill run release-notes --input tag=v1.4.0 --json | jq -r .outputs.markdown > NOTES.md
mill pipe 'sh "git log --since=1.week --pretty=%s" | agent claude/haiku "Write a weekly update" | stdout'
mill status    # ≋ 1 failed · 1 retrying · 1 working · 14 done today · $6.20 today
```

## Next

- [docs/agents/HANDOFF.md](docs/agents/HANDOFF.md): AI agents start here (current state, open work)
- [docs/index.md](docs/index.md): documentation map
- [docs/spec/index.md](docs/spec/index.md): the specification
- [docs/plans/README.md](docs/plans/README.md): plans and how they are written
- [CONTRIBUTING.md](CONTRIBUTING.md): gate, commits, pull requests

Until code exists, the spec is the reference. Once code exists, Go source wins when docs disagree with behavior, and the spec gets fixed.
