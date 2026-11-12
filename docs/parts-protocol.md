# Parts protocol

How to write an executable part in any language: the envelope, lifecycle, messages and rules.

Status: **not written yet.** Written when the feature ships: v0.5 (public parts protocol v1). In v0 the protocol exists but is internal ([roadmap](spec/28-roadmap.md#28-roadmap)).

Until then the spec is the reference:

- [spec §9.3](spec/09-parts-and-plugins.md#93-parts-protocol-v1): lifecycle, messages, rules, SDKs
- [spec §9.2](spec/09-parts-and-plugins.md#92-ways-to-extend-from-least-to-most-effort): when a part is the right way to extend mill
- [spec §16.1](spec/16-wire-protocol.md#161-envelope): the shared envelope

When written, this page covers: message catalog with JSON examples, a minimal part in Go and Python, manifest flags, testing a part.
