## Appendix B. Walkthrough: from install to result

The whole loop: open mill, make a pipeline, have an agent use it, and get the result. Each step names the TUI keys and the equivalent command. The browser prototype follows the same keys.

### B.1 Install and load

1. Install: `go install github.com/<org>/mill/cmd/mill@latest` and `…/cmd/milld@latest` (or `brew install mill`).
2. Run `mill setup` once (§18.8). It finds your agents, picks a soundset, adds shell completion and the `≋` prompt, installs starter plugins (and `mill/rex` if rex is there), and starts the daemon.
3. Run `mill`. You hear "mill", and the board opens: failed, running and ready pipelines, one line each. `?` shows every key.

### B.2 Make a pipeline

4. Press `n`. The wizard opens over the board.
5. **describe:** type the name `fix-flaky-test`, then a sentence: *"every night find flaky tests, fix them, run the tests and open a draft PR"*. An agent proposes the trigger, agent and steps. (Or pick a template.)
6. **trigger:** check what it picked (here "nightly 03:00"), or choose **manual** if an agent should start it on demand.
7. **agent:** pick the agent that does the work, for example `claude · sonnet`.
8. **limits:** pick a budget, keep the destructive steps it needs allowed (`gh.pr`), choose what happens on failure (a GitHub issue, or a rex session), keep **dry run first** on.
9. **review:** check the steps (`space` toggles one) next to the YAML, then press `r` to create and dry-run.
10. The editor opens on the new pipeline. Adjust anything with `hjkl` and the inspector, `a` to add a part, `E` for the YAML, `:w` to save, `esc` back to the board.
11. On the board the pipeline sits under READY with `never run`. The dry run finishes and its text becomes `dry run ok · would open 1 draft PR`.

Command-line equivalent:

```sh
mill new fix-flaky-test --from 'describe:every night find flaky tests, fix them and open a draft PR' \
  --agent claude/sonnet --budget medium --allow gh.pr --on-failure gh.issue --dry-run
```

### B.3 Make an agent use it

There are two ways an agent works with a pipeline. Most setups use both.

**Inside the pipeline.** The `work` node *is* an agent: mill starts it unattended with the prompt from `@prompts/fix-flaky-test.md`, gives it a git worktree, and collects its result. Edit the prompt and the output schema in the inspector or the YAML. If the agent asks a question mid-task, the node fails with `M6304` instead of waiting, and you tighten the prompt.

**From outside: an agent starting the pipeline.**

12. **Over MCP (recommended).** Register mill as a tool server for your agent, for example in Claude Code:

    ```sh
    claude mcp add mill -- mill mcp serve
    ```

    The agent now has `list_pipelines`, `describe_pipeline`, `run_pipeline`, `get_run`, `rerun` and `lint_pipeline`. Ask it, for example, *"run fix-flaky-test for TestCache_Expiry and tell me what changed"*. It calls `run_pipeline` with that input, then `get_run` until the run is done, and reads `outputs` (the PR link and a summary).

13. **From a shell.** Any agent that can run commands can call:

    ```sh
    mill run fix-flaky-test --input test=TestCache_Expiry --json
    ```

    It blocks until the run finishes and prints one JSON object with the state, cost and outputs. Progress goes to stderr, so the JSON stays clean. Exit codes say what happened (0 done, 1 failed, 9 over budget).

14. **From rex.** With `mill/rex` installed, a `rex.event` trigger can start the pipeline when a rex session is marked done, and a failing run can hand off to a new rex session with the error and logs.

### B.4 Watch it run

15. On the board, the pipeline moves to RUNNING and its text says which node is working. `enter` opens the run popup: one row per node with a timeline of where the time went and costs so far. `enter` on an agent node shows its screen (view-only).
16. `L` in the run popup shows the log. `/` searches, `za` opens folded agent output, and `f` follows new lines.
17. Nothing needs you during the run. Rate limits and flaky steps retry on their own (amber), and failures end the run with a clear error.

### B.5 Get the result

18. **Where results land:** the pipeline's real outputs, here a draft PR on GitHub and a Slack message, plus the run's `outputs` (`pr_url`, `summary`).
19. **In mill:** the pipeline returns to READY as `✓` with the result as its text (`draft PR #1301 · next 03:00`), and a new green square in its runs. `p` shows the preview line. `mill items fix-flaky-test#1/pr --json` prints exactly what a node produced.
20. **For the agent that started it:** `get_run` over MCP, or the JSON from `mill run --json`, both include the outputs.
21. **If it failed:** the error says what broke and what to type next, for example `help: mill rerun fix-flaky-test#3 --from check`. If you set a failure handoff, a GitHub issue or a rex session is already waiting with the context.
22. **From now on:** the nightly trigger runs it unattended. `mill status`, or the `≋✗1` prompt badge, tells you when something needs a look.

---

[← spec index](index.md)
