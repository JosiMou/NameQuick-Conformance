# Conformance vectors

Language-neutral scenarios that both NameQuick clients (macOS/Swift, Windows/Rust+TS) must satisfy against `contract_version: 1` (Phase A) of the managed-proxy client contract. The authoritative prose is [`../spec/managed-proxy-client-spec.md`](../spec/managed-proxy-client-spec.md).

**Vectors are pure data.** No code in this repo decides behavior. Each client implements its own runner that:

1. reads a vector's `given` fixtures,
2. drives its own client code (either as a real HTTP exchange against a mock server seeded with the fixtures, or as a unit test that feeds the fixtures directly into the client's request-builder / response-handler),
3. asserts every field in the vector's `expect` block.

A client is conformant for v1 when it passes every `contract_version: 1` vector here.

## Schema

Each `*.json` file validates against [`vector.schema.json`](./vector.schema.json). Required keys:

- `id` — stable id, equals the filename stem.
- `contract_version` — `1` for Phase A.
- `title` — one-line human summary.
- `kind` — `wire` (fixed request/response bytes and shapes at the HTTP boundary) or `behavior` (a required client decision/output given a fixture).
- `given` — input fixtures (see conventions below).
- `expect` — required client behavior/output.

Optional: `description`, `spec_refs` (spec section anchors).

## `given` / `expect` conventions

These are conventions, not a rigid schema — `given` and `expect` are intentionally open so each scenario can describe itself. Runners key off the fields relevant to the `kind`.

Common `given` fields:

- `request` — the request the client is expected to build: `{ method, path, headers, body }`. For header-derivation vectors, `request.inputs` carries the raw inputs (e.g. `customerId`) and `request.headers` carries the expected sanitized output.
- `response` — a single server response `{ status, headers, body }`.
- `responses` — an ordered list of responses for a multi-step exchange (e.g. poll sequences, chunked upload). The runner replays them in order.
- `context` — non-wire inputs (e.g. `bundle_patch_deployed: false`, prior session state).

Common `expect` fields:

- `outcome` — a stable string naming the required client outcome (e.g. `proceed_to_generate`, `resume_from_expected_offset`, `honest_provider_outage`, `insufficient_credits`).
- `client_action` — the concrete action the client must take next (`send_next_chunk`, `retry_after_seconds`, `abort`, `surface_error`).
- `error_state` — for failure scenarios, the user-facing state class the client must surface (never a silent reroute).
- `assert` — a list of specific field-level assertions the runner checks (e.g. `honor_retry_after: true`, `parsed_reason: free_quota_exceeded`).
- `must_not` — behaviors that fail the vector if observed (e.g. `silent_provider_reroute`, `billed_retry`, `restart_whole_upload`).

Runners MUST treat an assertion they do not understand as a failure, not a pass (fail-closed), so that a client cannot silently skip a requirement it has not implemented.

## Fixture realism

- Customer ids are always obviously fake (`cus_test_0000000000`). No real customer ids, secrets, or tokens appear anywhere.
- The only cryptographically real value is the client-token in `client-token-derivation.json`, which is a SHA-256 of a fake input and is verifiable offline (`printf '%s' "namequick-proxy-client:cus_test_0000000000" | shasum -a 256`).
- Error bodies are copied byte-for-byte from the audited proxy source so a `wire` runner can assert exact JSON.

## Index

| id | kind | proves |
|---|---|---|
| `happy-two-step-upload-generate` | behavior | Full start -> chunk -> poll ACTIVE -> generate happy path, parts ordering. |
| `chunk-308-continue` | wire | 308 on a non-final chunk means accept-and-continue. |
| `chunk-offset-mismatch-resume` | wire | 400 expected_offset -> resume from anchor, not restart. |
| `upload-finalize-size-mismatch` | wire | 400 size mismatch is a fatal session. |
| `upload-session-expired` | wire | 400 invalid/expired session -> restart from /upload/start. |
| `upload-already-finalized` | wire | 400 already-finalized -> stop chunking. |
| `poll-processing-then-active` | behavior | PROCESSING repeats then ACTIVE proceeds. |
| `poll-failed-abort` | behavior | FAILED (non-ACTIVE) aborts. |
| `poll-exhaustion-timeout` | behavior | 24 PROCESSING attempts -> timeout. |
| `windows-bundle-id-accepted` | wire | app.namequick.windows accepted post-patch. |
| `bundle-id-missing-403` | wire | Missing X-Bundle-Id -> 403 exact body. |
| `bundle-id-unlisted-403` | wire | Unlisted bundle -> 403 exact body. |
| `quota-429-rate-limited` | wire | 429 reason rate_limited parsed. |
| `quota-429-free-quota-exceeded` | wire | 429 reason free_quota_exceeded distinct state. |
| `shaper-429-traffic-rejected-retry-after` | wire | 429 traffic_rejected -> honor Retry-After. |
| `shaper-503-traffic-circuit-open` | wire | 503 circuit_open -> honor Retry-After. |
| `generate-502-attempted-models` | wire | 502 -> honest provider-outage state. |
| `generate-504-timeout` | wire | 504 -> transient temporary-failure. |
| `malformed-model-output-tolerant-ladder` | behavior | Invalid JSON -> recovery ladder, non-debited. |
| `empty-candidates-empty-response` | behavior | Empty candidates -> emptyResponse error. |
| `model-refusal-detection` | behavior | Refusal detected as failure, not a filename. |
| `entitlement-expired-still-succeeds` | behavior | Expired entitlement token -> request still succeeds (observe-only). |
| `header-sanitization` | behavior | Illegal chars -> '-', 128 truncation, empty -> omit, denied fragment -> omit. |
| `client-token-derivation` | behavior | Known customerId -> exact 32-hex token. |
| `network-error-one-transport-retry` | behavior | Proxy down -> one transport retry, honest error, no reroute. |
