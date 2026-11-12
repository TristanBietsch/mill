## 11. Agents

### 11.1 Agent registry (`mill.agent` in Lua)

Agent CLIs are described, not coded. Built-in entries ship embedded. You add or override entries with `mill.agent "id" { … }` in Lua config (§19), and plugins can ship agent entries too. Entries merge by `id` (scalars replace, `models` merge by model id).

| Field | Meaning |
|---|---|
| `id`, `name` | Stable id and display name. |
| `command` | Base argv. |
| `models[]` | `id`, `name`, `args`, optional `price` (per million input/output tokens) for cost tracking. |
| `effort` | How effort tiers map to arguments (`low`, `medium`, `high`, `max`). |
| `headless` | Args for non-interactive mode, plus how output comes back (`stdout-json`, `stdout-text`, `file`). |
| `terminal` | Args for agents that only run in a terminal (PTY), plus `auto_answer` rules for the prompts they show (§11.4). |
| `prompt` | How the prompt is delivered: `argv` (with position), `stdin`, `file`, or `paste` (typed into the PTY once `ready_regex` matches). |
| `detect` | How state is read: `exit` (exit code only), `heuristic` (`busy_regex`, `ready_regex`, `idle_ms`, `done_regex`), or `hooks` (the agent writes state words to a file mill provides). |
| `usage` | How token counts are read: a regex over output, a JSON path in headless output, or hooks. |
| `trust_prompt` | Optional: how to recognize and answer the agent's "trust this folder?" dialog, only in folders the user has listed as trusted. |
| `env` | Extra environment. |

Built-in entries at launch: Claude Code, Codex CLI, Gemini CLI, Aider, OpenCode, Ollama (`ollama run`), and `echo` (a fake agent for tests).

### 11.2 Unattended by design

Every agent node runs without a person. mill never waits for someone to answer an agent, approve a step or take over a session.

| Mode | Used when | How |
|---|---|---|
| **Headless** (default) | The agent CLI has a non-interactive mode | Plain child process. Structured output on stdout. |
| **Terminal** | The agent only works inside a terminal | Child in a PTY. mill answers the prompts it knows itself (§11.4), and fails the node with `M6304` if the agent asks anything else. |

### 11.3 Getting structured output from an agent

In order of preference:
1. **Headless JSON**: the agent's own structured output mode.
2. **Output file**: mill sets `MILL_OUTPUT` to a path and appends to the prompt: *"When you're finished, write your result as JSON matching this schema to $MILL_OUTPUT."* The schema is the node's `output`.
3. **Extraction pass**: if the file is missing or invalid, `agent.extract` reads the transcript tail and produces the JSON.
4. **Validation**: if the node declares `output`, the result is validated. On failure, retry once with the validation error appended, then fail.

Every agent node also gets: `MILL_RUN_ID`, `MILL_NODE_ID`, `MILL_WORKSPACE`, `MILL_ARTIFACTS` (a directory whose files are registered as artifacts at the end).

### 11.4 Agents that need a terminal

- Some agent CLIs only work in a terminal. mill runs them in a PTY and reads their screen like any other output.
- Known prompts are answered by `auto_answer` rules in the agent registry: a pattern, the keys to send, and where it applies. Folder-trust dialogs, for example, are only answered for folders in `trusted_dirs`.
- If the agent stops and asks anything else (its ready prompt reappears mid-task, or the screen goes idle on a question), the node fails with `M6304 agent asked for input`. The question is included in the error, so the pipeline can be fixed with a clearer prompt, a flag or a new `auto_answer` rule. It never waits.
- You can watch a running agent without touching it: `enter` on the node in the TUI, or `mill attach <run>/<node>`. Both are view-only; no keystrokes are sent.

### 11.5 Patterns built into parts

- **Best-of-N:** `agent.fanout` (N workspaces) → `check` on each → `agent.judge` → keep the winner, clean up the rest.
- **Independent review:** `agent.review` always starts a fresh session with only the diff and the task, never the producer's transcript.
- **Fix until green:** `agent.loop` with `until: { sh: "make test" }`, feeding failures back, bounded by `max_iters` and the budget.
- **Plan, check, execute:** a planning agent, an `assert` on the plan (risk, size, files touched), then an executing agent. Plans that fail the check route to a handoff.
- **Handoff:** when a run reaches something a machine shouldn't decide, a `handoff` node puts it where a person will see it (an issue comment, a message, a ticket, another tool) and that branch ends. If the person's answer should continue the work, it arrives as a trigger and starts a new run.


---

[← spec index](index.md)
