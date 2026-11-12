## 21. Secrets

- Pipelines refer to secrets by name (`secrets.GITHUB_TOKEN`) and must declare them in `secrets:`.
- Values come from secret-backend parts, tried in configured order: `env`, OS keychain (default), 1Password (`op://vault/item/field`), HashiCorp Vault, and an age-encrypted file for headless servers.
- A part receives only the secrets named in its node's config, and only when it opens. Secrets are never passed as command-line arguments (they'd be visible in process lists); they go through environment variables or the parts protocol.
- Every resolved value is registered with the redactor. Exact matches (and common encodings: base64, URL-encoded) are replaced with `•••` in logs, items, artifacts written by mill, and client events.
- `mill secrets test <name>` checks a secret resolves, without printing it.


---

[← spec index](index.md)
