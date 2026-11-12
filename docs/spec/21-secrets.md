## 21. Secrets

- Pipelines refer to secrets by name (`secrets.GITHUB_TOKEN`) and must declare them in `secrets:`.
- Values come from secret-backend parts, tried in configured order: `env`, OS keychain (default), 1Password (`op://vault/item/field`), HashiCorp Vault, and an age-encrypted file for headless servers.
- A part receives only the secrets named in its node's config, and only when it opens. Secrets are never passed as command-line arguments (they'd be visible in process lists); they go through environment variables or the parts protocol.
- Every resolved value is registered with the redactor. Exact matches (and common encodings: base64, URL-encoded) are replaced with `•••` in logs, items, artifacts written by mill, and client events.
- A file a step writes into `MILL_ARTIFACTS` gets the same treatment: text files are redacted on the way into the artifact store (and marked `redacted`); a binary file that contains a secret fails the node with `M9205` instead of being stored.
- In expressions a secret is an opaque handle: it can go whole into a field the part marks as secret or into an `env` entry, never into argv or a larger string (§8).
- `mill secrets test <name>` checks a secret resolves, without printing it.


---

[← spec index](index.md)
