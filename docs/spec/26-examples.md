## 26. Example pipelines

These ship in `examples/` and double as the integration test suite (run against the `echo` agent and recorded HTTP fixtures).

**Code**
1. **issue-to-pr:** label → worktree → plan → scope check → build until `make check` passes → independent review → draft PR → Slack. Out-of-scope plans become an issue comment (§7.5).
2. **best-of-3:** the same task to three agents in three worktrees → checks → judge compares diffs → draft PR from the winner, the others cleaned up.
3. **nightly-vuln-sweep:** cron → `govulncheck` / `npm audit` → if anything is found, an agent fixes it in a worktree → checks → draft PR. Otherwise nothing.
4. **multi-repo-refactor:** `map` over a list of repos → one agent each with a shared instruction → checks → one draft PR per repo that passes; repos that fail get a handoff issue.
5. **pr-fixer:** new review comment on your PR → agent writes a fix → checks → push a fixup commit and reply to the comment with what changed.
6. **release-notes:** tag pushed → merged PRs since the last tag → agent drafts notes in the changelog's style → judge checks them against the PR list → publish as a draft release.
7. **docs-drift:** merge to main → agent compares `docs/` against the source → docs PR if they've drifted.

**Automation**

8. **morning-brief:** weekday cron → RSS, market APIs and calendar (MCP) → headless summary → markdown file and Slack DM.
9. **inbox-triage:** poll email (MCP) every 15 minutes → classify → label and archive the routine ones → draft replies for the rest into the drafts folder (a handoff: you send them).
10. **form-intake:** webhook → enrich via HTTP → CRM record via MCP → notify.
11. **notes-to-post:** a new file in `~/notes/drafts` (fswatch) → agent edits it into a post → lint and link checks → commit to the blog repo as a draft PR.
12. **failure-digest:** `run` trigger on any pipeline's `failed` → group by cause → one daily message with each failure and its `mill rerun` command.

**Pipe one-liners**

```sh
mill pipe 'sh "git log --since=1.week --pretty=%s" | agent claude/haiku "Write a weekly update from these commits" | stdout'
mill pipe 'file.glob path="logs/*.log" | each sh "grep -c ERROR" | reduce ${{ items.sum() }}'
```


---

[← spec index](index.md)
