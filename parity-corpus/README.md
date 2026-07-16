# Prompt-assembly parity corpus

Purpose: guarantee that the macOS (Swift) and Windows (Rust+TS) clients produce **byte-identical** request bytes for identical inputs on the managed rename path. Two clients that both "look right" can still drift in prompt text, JSON key ordering, whitespace, or number formatting; that drift silently changes model behavior and makes the two editions diverge. The parity corpus is the byte-equality gate that prevents it.

Scope: the assembled `/generate` request body (and the prompt text within it) for a given rename input. Transport headers and the file bytes themselves are out of scope here (headers are covered by the `header-sanitization` and `client-token-derivation` vectors; upload bytes are opaque).

## Status: format defined, population PENDING

The corpus format is defined below and one self-evident structural example is included. **Real Mac-derived entries are not yet populated** because deriving the exact prompt/request bytes the Mac client emits requires executing Mac client code (the prompt assembler, output-language resolver, and JSON serializer), which is out of scope for this authoring pass. Faking entries would defeat the gate, so none are faked.

**Follow-up (Mac-side snapshot task):** add a small harness in the Mac repo that serializes the assembled `/generate` body for a fixed set of rename inputs and writes each as a corpus entry here. Until that lands, the corpus gate is defined but not enforced on real prompts. This is honestly incomplete and tracked as a separate task, not part of W5's deliverable bytes.

## Entry format

Each entry is a directory `entries/<entry-id>/` containing:

- `input.json` — the rename input fixture: file metadata (name, mime, size), resolved output language and its source, preset/template context if any, and any flags that affect assembly. Must be fully self-contained (no reference to machine state).
- `expected-request.json` — the canonical `/generate` request body the client must produce, serialized with the canonical rules below. This is the byte-exact expected output.
- `expected-request.sha256` — the SHA-256 of the exact bytes of `expected-request.json`, lowercase hex. This is the gate value.

## Canonical serialization rules

Both clients MUST serialize the request body identically. The canonical form is:

1. UTF-8 encoding, no BOM.
2. Object keys in the exact order the contract specifies for the rename path (e.g. `model`, then `contents`, then `generationConfig`); within `parts`, the text part precedes the `file_data` part (spec §5.2). Key order is **explicit and load-bearing**, not alphabetical and not language-map-iteration order.
3. No insignificant whitespace: no spaces after `:` or `,`, no trailing newline. (A single canonical pretty-print form MAY be chosen instead, but it MUST be identical across clients; the default here is compact.)
4. Non-ASCII characters in strings emitted as literal UTF-8 (not `\uXXXX` escaped), except the JSON-mandatory escapes (`"`, `\`, control characters).
5. Numbers in their shortest round-trip form; no trailing `.0` on integers, no exponent for values that fit as plain decimals.

## Byte-equality gate rule

For each entry: the client assembles the request from `input.json`, serializes it under the canonical rules, and computes SHA-256 over the resulting bytes. The entry passes only if that digest equals `expected-request.sha256` **and** the bytes equal `expected-request.json` exactly. Any difference — a reordered key, an extra space, a different number format, an escaped vs literal Unicode character — fails the gate. Both clients must pass every entry.

## Self-evident structural example

`entries/example-canonical-serialization/` demonstrates the serialization rules on a minimal, self-contained body (no real Mac prompt). Its `expected-request.sha256` is computed from its own `expected-request.json` bytes and is verifiable offline:

```
shasum -a 256 parity-corpus/entries/example-canonical-serialization/expected-request.json
```

This example exists to test that a client's serializer obeys the canonical rules (key order, compact whitespace, literal UTF-8, integer formatting). It is **not** a real rename prompt and does not stand in for corpus population.
