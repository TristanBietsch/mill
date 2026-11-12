## 14. Piping and Unix composability

### 14.1 mill as a filter

```sh
# one run per input line, outputs as JSONL on stdout
gh issue list --json number,title,body | jq -c '.[]' | mill run triage --each

# a pipeline's outputs feed the next tool
mill run release-notes --input tag=v1.4.0 --json | jq -r .outputs.markdown > NOTES.md

# chain pipelines
mill run scrape --json | mill run summarize --stdin | mill run post-to-slack --stdin
```

Rules:
- `--stdin` (or `-` as the input) reads items from stdin: JSONL by default, `--text` for plain lines.
- `--each` starts one run per input item; without it, all items go to one run.
- Run **outputs** go to stdout as JSONL (`--json` adds the full run record). Progress and logs go to **stderr**, so pipes stay clean.
- Exit code reflects the run: 0 done, 1 failed, 9 budget exceeded (§17.2). A run never blocks on a person, so a piped `mill run` always finishes.
- `--wait` (the default when stdout isn't a terminal) blocks until the run finishes. `--detach` prints the run id and returns immediately.

### 14.2 Pipe expressions

A one-line pipeline in the shell. Stages are separated by `|`, exactly like a shell pipeline:

```sh
mill pipe 'rss url=https://example.com/feed.xml
         | filter ${{ item.published > now - duration("24h") }}
         | agent claude/haiku "Summarize in one line: ${{ item.title }}"
         | file.append path=~/notes/brief.md'
```

Grammar (informal):
- `stage := part-id [key=value | "positional"]...`
- `sh '…'` runs a shell command as a stage. Plain-text stdin/stdout, so any Unix tool works mid-pipeline.
- `agent <id>[/<model>] "<prompt template>"` is shorthand for an agent node.
- `each` / `batch` before a stage sets its mode.
- `assert <expr>` stops the pipeline (or drops items with `assert drop <expr>`) when a check fails.
- `as <name>` names a stage so later stages can reference `${{ nodes.<name>.item }}`.

Pipe expressions compile to the same execution graph as YAML. `--save <name>` writes the equivalent `*.mill.yaml`. `--dry-run` prints the graph. They're linear by design; for branches and joins, save to YAML and open it in the canvas.

### 14.3 Steps are filters

Inside a pipeline, any program is a step:

```yaml
nodes:
  clean:
    uses: sh
    with: { run: "jq -c 'select(.score > 0.8)'" }
    io: plain
```

### 14.4 Scripting the daemon

Everything is reachable without the TUI:
- `mill events --json` streams every daemon event as JSONL (for your own dashboards and bots).
- `mill wait <run> --until done|failed|any --timeout 30m`.
- `mill rerun <run> --from <node>` after fixing what failed.
- The raw socket protocol (§16) for full control from any language.


---

[← spec index](index.md)
