## 24. Observability and cost

- **Logs:** every node-run's raw output is kept (`mill logs --raw`). `mill logs -f` tails a whole run with node prefixes.
- **Events:** `mill events --json` streams everything as JSONL.
- **Traces:** an `otel` sink part exports runs as traces and node-runs as spans (attributes: part, runner, items in/out, tokens, cost, exit code).
- **Metrics:** a `prometheus` sink exposes counts, durations, queue depth and token totals.
- **Cost:** token usage comes from parts and agent detection. Money is computed from `price` in the agent registry. Runs show a ledger, and `mill stats` totals by pipeline, agent and model.
- **Digest:** `mill digest` summarizes a day: runs, failures, retries, slowest nodes, spend.


---

[← spec index](index.md)
