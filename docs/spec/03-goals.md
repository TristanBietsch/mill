## 3. Goals and non-goals

### Goals
1. Build, view and run pipelines **visually in a terminal**, with live state on the graph.
2. **Compose with Unix**: stdin/stdout, JSONL, exit codes, `--json` everywhere, a one-line pipe syntax.
3. **Run any agent CLI** (Claude Code, Codex, Gemini, Aider, OpenCode, local models through Ollama, or anything else), always unattended.
4. **Run any program as a step**, and let anyone add new step types in any language.
5. **Cover personal and team automation**: schedules, webhooks, HTTP, MCP tools and transforms are enough to replace a typical n8n or Zapier setup.
6. **Durable, inspectable runs** that survive client exits and daemon restarts where it's safe, with a full audit trail.
7. **Interop**: call and be called by n8n and Zapier, import n8n workflows, expose mill as an MCP server, export simple pipelines to other formats.

### Non-goals (v1)
- A browser-based editor. A read-only web viewer comes later (§25).
- Hand-written integrations for hundreds of SaaS apps. MCP, HTTP and OpenAPI cover that ground (§23).
- A general-purpose data-processing engine (Spark, Beam). mill moves work and moderate data, not terabytes.
- Multi-tenant hosted SaaS at launch.
- Re-implementing an agent. mill orchestrates agents; it doesn't contain one.
- Human-in-the-loop work. No approvals, inboxes or interactive sessions inside mill. rex (or any tool built for supervising agents with a person present) handles that, and mill hands work to it (§10.3, §23.6).


---

[← spec index](index.md)
