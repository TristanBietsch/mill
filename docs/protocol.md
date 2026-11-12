# Daemon wire protocol

The JSONL protocol between clients and `milld`: envelope, handshake, intents, events, error codes.

Status: **not written yet.** Written when the feature ships: v0 (milestone M2, the daemon) ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §16](spec/16-wire-protocol.md#16-daemon-wire-protocol): envelope, handshake, intents, events, error codes
- [spec §29.3](spec/29-errors.md#293-json-output): the error object every `Error` event carries

When written, this page covers: every intent and event with payload fields and a JSON example; versioning rules.
