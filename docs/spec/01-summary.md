## 1. Summary

mill moves work between machines.

You describe a pipeline as a graph of steps: agent calls, shell commands, HTTP requests, MCP tools, checks, branches, loops. You build it in a terminal UI, as a YAML file, or as a one-line pipe expression. A local daemon runs it by hand, on a schedule, or when something happens (webhook, file change, git event, a message arriving).

Three properties define mill:

- **It's a filter.** mill reads items on stdin and writes items on stdout like any Unix tool. Any program that reads and writes lines can be a step. Pipelines pipe into other pipelines and into `jq`, `grep` and `xargs`.
- **It's made of parts.** Every step type, trigger, runner, agent, secret store and output sink is a plugin ("part") speaking one small protocol. The built-ins use that same protocol. You can swap any of them.
- **It's agent-native but agent-agnostic.** Coding agents are first-class: isolated git workspaces, best-of-N, judges, independent reviews, loops until tests pass. But mill depends on no vendor. Agent CLIs are described in a YAML registry, and anything that runs in a terminal works.

It covers the same ground as n8n and Zapier, but it lives in your terminal and your git repo, and it treats a coding agent as just another machine on the line.

**mill is hands-off.** A pipeline never waits on a person. Every decision inside a run is made by the pipeline itself (checks, judges, reviews, policies), or the run ends with a clear result. When something needs a human, mill hands it off to wherever people work (an issue, a message, a ticket, another tool) and finishes. mill is about creating, changing and running pipelines, not about waiting for input.

It's meant to sit next to **rex**: agent sessions that need a person live on rex's board, and everything that should run unattended lives in mill. The two connect through an optional plugin, and every connection point has a generic equivalent, so mill works just as well with GitHub, Slack, Linear, any other agent supervisor, or nothing at all (§23.6).


---

[← spec index](index.md)
