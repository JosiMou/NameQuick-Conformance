# Managed Proxy Client Contract

`contract_version: 1` (v1 = Phase A)

Status: authored 2026-07-17 for W5 (CAR-2169). This is the language-neutral contract that every NameQuick client (macOS/Swift and Windows/Rust+TS) must satisfy when talking to the managed Gemini proxy and the Polar credit-state service. It is written so a stranger can implement a conformant client from scratch without reading either client's source.

Two backends are in scope:

- **gemini-proxy** — the managed inference proxy (`namequick-gemini-proxy`). Owns the model chain, holds the Gemini/Vertex credentials, enforces caller quotas, and shapes traffic. Clients never see an API key.
- **polar-proxy** — the licensing-side credit-state service. Owns credit balances and entitlement-token issuance. The gemini-proxy has no credit endpoint; credit read happens entirely against polar-proxy.

Everything below is traceable to the audited ground truth. Where the ground truth is silent, the text says **unspecified** rather than inventing a value.

## Contents

- [1. Client identification](#1-client-identification)
- [2. Request headers](#2-request-headers)
- [3. Entitlement token (forward-compatible, observe-only in v1)](#3-entitlement-token)
- [4. Two-step upload contract](#4-two-step-upload-contract)
- [5. Generate contract](#5-generate-contract)
- [6. Error taxonomy](#6-error-taxonomy)
- [7. Credit read and exhaustion](#7-credit-read-and-exhaustion)
- [8. Versioning and breaking-change rule](#8-versioning-and-breaking-change-rule)
- [9. Divergence register](#9-divergence-register)
- [Appendix A. Endpoint summary](#appendix-a-endpoint-summary)

Keyword convention: **MUST / MUST NOT / SHOULD / MAY** carry their RFC 2119 meaning.

---

## 1. Client identification

The proxy's only hard authentication gate in v1 is the `X-Bundle-Id` header. The proxy runs `--allow-unauthenticated` and injects the Gemini/Vertex credentials server-side; the bundle-id allowlist is what separates a NameQuick client from arbitrary internet traffic.

- Every request to gemini-proxy (`/generate`, `/upload/start`, `/upload/chunk`, `/file/{id}`) MUST send `X-Bundle-Id`.
- The value MUST be the client's registered bundle identifier:
  - macOS client: `app.namequick.NameQuick`
  - Windows client: `app.namequick.windows` (matches the Tauri app identifier already scaffolded)
- The proxy admits a request only when the value is a member of its `ALLOWED_BUNDLE_IDS` allowlist. A missing, empty, or unlisted value is rejected with HTTP `403` and body `{"error":"Unauthorized","details":"Invalid bundle ID: <value>"}`.

**Allowlist change is a server patch, not a client assumption.** The Windows identifier `app.namequick.windows` is not yet in `ALLOWED_BUNDLE_IDS` (server `main.py:46`, which today reads `["app.namequick.NameQuick"]`). Adding it ships as a one-line patch for Josef's review and manual deploy. A client MUST NOT assume the patch is live: until it is deployed, a Windows build receives `403` on every call and MUST surface that as an honest "managed service not available for this build yet" state, not a silent retry loop.

The bundle id is advisory-for-identity only in the sense that the proxy trusts it structurally: forging it just moves the caller into the NameQuick bucket, where the per-IP quota caps still bound abuse. It is not a secret and MUST NOT be treated as one.

---

## 2. Request headers

Headers fall into three classes. Only `X-Bundle-Id` is enforced. Everything else is advisory: the proxy reads these for quota bucketing, operation classification, and telemetry, but MUST never reject a request over them, and a client MUST NOT depend on any advisory header changing the response.

### 2.1 Header set

| Header | Class | Value | Notes |
|---|---|---|---|
| `X-Bundle-Id` | auth (enforced) | bundle identifier (§1) | Only hard gate. |
| `X-App-Version` | telemetry | client short version string | e.g. `2.11.59`. |
| `X-Build-Number` | telemetry | client build number | |
| `X-Request-Id` | telemetry | per-operation correlation id | Same value as `X-Correlation-Id`. |
| `X-Correlation-Id` | telemetry | per-operation correlation id | Derived from the operation id. |
| `X-NameQuick-Client-Token` | advisory (quota principal) | derived token, §2.2 | Proxy's per-customer quota key. Opaque, never the raw customer id. |
| `X-NameQuick-Operation` | advisory | operation classifier | Proxy classifies against its op registry. Not set on the rename path by the Mac client. |
| `X-NameQuick-Billing` | advisory | `free:<reason>` \| `debit:<units>` \| `per_request` | Never trusted for billing decisions; client-side metering is authoritative. |
| `X-NameQuick-Flow` | telemetry | flow id | e.g. `analyze_with_metadata`, `schema_extraction`. |
| `X-NameQuick-Template` | telemetry | preset/template id | |
| `X-NameQuick-Output-Language` | telemetry | BCP-ish language code | e.g. `de`, `en`. |
| `X-NameQuick-Output-Language-Source` | telemetry | resolution source | `preferred_language` \| `detected_<source>` \| `fallback_<reason>`. |
| `X-NameQuick-Entitlement` | forward-compat (optional) | `nqe1.<b64url>.<hexsig>`, §3 | Observe-only in v1. See §3. |

Upload-transport headers (`X-Goog-Upload-*`, `X-Proxy-Session-Id`) are specified in §4.

### 2.2 Client-token derivation

The client-token is a pseudonymous, stable-per-customer quota key. It MUST be computed as:

```
token = lowercase_hex( SHA256( "namequick-proxy-client:" + polarCustomerId ) )[0:32]
```

That is: UTF-8 encode the ASCII string `namequick-proxy-client:` concatenated with the Polar customer id, take SHA-256, hex-encode lowercase, and keep the **first 32 hex characters** (the first 16 bytes of the digest). Trim surrounding whitespace from the customer id first; if the id is empty after trimming, omit the header entirely.

Worked vector (also in `vectors/client-token-derivation.json`):

```
polarCustomerId = "cus_test_0000000000"
SHA256("namequick-proxy-client:cus_test_0000000000")
  = f51aa29bd3e8a7be0cc758674975965237f5d85ee531689dca6540762979bd95
token (first 32 hex) = f51aa29bd3e8a7be0cc7586749759652
```

Reproduce with: `printf '%s' "namequick-proxy-client:cus_test_0000000000" | shasum -a 256`.

The raw customer id MUST NOT be sent in any header. The hash namespace prefix is fixed and MUST NOT be changed without a contract-version bump.

### 2.3 Sanitizer contract

Every advisory/telemetry header value MUST pass through the same sanitizer before being sent. A client that cannot reproduce this sanitizer MUST omit the value rather than send an unsanitized one. The sanitizer is:

1. Trim leading/trailing whitespace.
2. Reject (→ omit the header) if the value contains `/`, `\`, or `@`, or contains any denied substring (case-insensitive): `api_key`, `apikey`, `access_token`, `authorization`, `bearer`, `client_secret`, `cookie`, `credential`, `cus_`, `cust_`, `license_key`, `password`, `private_key`, `refresh_token`, `secret`, `sk-`. Values that look like a raw customer id, a filename, or a model identifier are also rejected.
3. Replace every character outside the allowed set with `-`. Allowed set (charset): `^[A-Za-z0-9._:-]*$` (ASCII letters, digits, `.`, `_`, `:`, `-`).
4. Truncate to **128 characters** (applied after replacement).
5. If the result is empty, omit the header entirely (never send an empty-valued advisory header).

The denied-substring rule is why the client-token is a hash and not the raw id: `cus_...` would be rejected outright, but a hex digest passes cleanly.

---

## 3. Entitlement token

**State it honestly: in v1 = Phase A the wire truth is that no client sends an entitlement token, and the proxy never rejects on its absence.**

The proxy contains complete `nqe1` verification code (Phase B, CAR-2094), but that code is observe-only and fail-open, self-gated on signing-secret provisioning. Until the secret is provisioned and polar-proxy issuance is wired, the header does nothing on the wire.

### 3.1 What the header is (when present)

- Header name: `X-NameQuick-Entitlement`.
- Format: `nqe1.<b64url-payload>.<hexsig>` — a version tag `nqe1`, a base64url-encoded JSON payload, and a lowercase hex signature.
- Signature: HMAC-SHA256 over the ASCII bytes `nqe1.` + `<b64url-payload>`, using a `kid`-selected secret. The signing secret lives in the proxy's Secret Manager (`NQ_ENTITLEMENT_SIGNING_SECRET`, a multi-line `kid:hexsecret` map, 5-minute cache).
- Claims (JSON payload): `{kid, sub, knd, env, iat, exp}`.
  - `kid` — key id, selects the verifying secret.
  - `sub` — subject; when a token verifies, `sub` replaces the advisory client-token as the quota principal.
  - `knd` — kind; `knd == "trial"` moves the caller onto trial quota caps.
  - `env` — must equal the proxy's expected Polar environment (`NQ_EXPECTED_POLAR_ENV`, default `production`); mismatch is treated as an expired/invalid token.
  - `iat`, `exp` — issued-at / expiry, unix seconds. Clock skew tolerance is **300 seconds**: `exp < now - 300` is expired; `iat > now + 300` is expired.
- Failure reasons the proxy distinguishes internally: `missing`, `malformed`, `secret_unavailable`, `bad_signature`, `expired`, `env_mismatch`. None of these produce a rejection in v1 — they are attribution/telemetry outcomes only.

### 3.2 Client requirements in v1

- A client **MUST NOT** fail, retry, or degrade a request because the entitlement header is absent, expired, or rejected. The header is not a precondition for any call in v1.
- A client **MUST** tolerate the header's total absence (its own and the server's — the server returns no entitlement header).
- A client **SHOULD** begin sending a valid `X-NameQuick-Entitlement` once polar-proxy token issuance is wired and the proxy secret is provisioned (Phase B). Sending a well-formed token early is safe: the proxy observes it and, if it verifies, attributes quota to `sub` instead of the client-token.

### 3.3 Phase B migration

Token lifecycle is **polar-proxy-owned**. Issuance, refresh, and revocation are all the licensing service's responsibility:

- Tokens are minted by polar-proxy on license validation. There is no token-issue, refresh, or revocation endpoint on gemini-proxy, and none in the polar-proxy contract audited here.
- **Expiry (`exp`) is the only lifecycle control that exists today.** There is no refresh endpoint and no revocation/`jti` denylist in the proxy. A client's only "renew" path is to obtain a fresh token the next time it validates its license with polar-proxy.
- The migration is additive: v1 clients that never send the header remain conformant; Phase B is reached when (a) the proxy secret is provisioned and (b) clients send a token. No breaking change to the wire is required to cross into Phase B, because the header was reserved as optional from v1.

---

## 4. Two-step upload contract

Files are uploaded to the proxy in a resumable, chunked flow, then referenced by URI in `/generate`. **Nothing is prompt-inlined and no inline base64 is used on the managed rename path** — the file is always uploaded first and passed as a `file_data` part.

The full sequence is: **start → chunk (repeat) → poll `/file/{id}` until ACTIVE → generate**.

### 4.1 Start — `POST /upload/start`

Request headers (in addition to §2):

- `X-Goog-Upload-Protocol: resumable`
- `X-Goog-Upload-Command: start`
- `X-Goog-Upload-Header-Content-Length: <total-bytes>`
- `X-Goog-Upload-Header-Content-Type: <mime-type>`

Request body: `{"file":{"display_name":"<name>"}}`.

Response:

- On success, the proxy returns `X-Proxy-Session-Id` (a proxy-issued session handle) plus `X-Goog-Upload-Status` and `X-Goog-Upload-Size-Received`. In Vertex mode the body is `{}` and `X-Goog-Upload-Status: active`.
- **`X-Proxy-Session-Id` is REQUIRED in the response.** A client MUST treat its absence as a fatal upload error and abort — it is the handle every subsequent chunk carries.

Session TTL is **3600 seconds** (`UPLOAD_SESSION_TTL_SECONDS`). A session that outlives its TTL is invalid; see §4.2.

### 4.2 Chunk — `POST /upload/chunk`

- Chunk size: **8 MB** (`8 * 1024 * 1024`). Chunks are sent strictly in order.
- Each chunk request carries: `X-Proxy-Session-Id` (from start), `Content-Length`, `X-Goog-Upload-Offset` (the byte offset of this chunk within the file), and `X-Goog-Upload-Command`. The command is `upload` for a non-final chunk and `upload, finalize` for the last chunk.
- Offsets are **strictly sequential**. The proxy compares the sent offset to the bytes it has received:
  - Match, non-final → HTTP **308** with `X-Goog-Upload-Status: active`. **308 means "accepted, continue"** — the client sends the next chunk. A client MUST treat 308 (and any 2xx) as success for a non-final chunk; treating 308 as an error is a conformance failure.
  - Offset mismatch → HTTP `400` with body `{"error":"Unexpected upload offset","expected_offset":N}`. `expected_offset` is the **resume anchor**: the client MUST resume by re-sending from byte `N`, not restart the whole upload.
  - Final chunk, match → HTTP `200`, `X-Goog-Upload-Status: final`, body `{"file":{"uri":"gs://<bucket>/uploads/<session_id>"}}` (Vertex) / Files API `uri` (developer_api). The client decodes `file.uri` for the generate step.
  - Final chunk, size mismatch → HTTP `400` with body `{"error":"Final upload size does not match session metadata","expected_size":M,"received_bytes":K}`. Fatal for this session.
  - Session invalid or expired (developer_api) → HTTP `400` `{"error":"Invalid or expired upload session"}`.
  - Chunk sent after the session was already finalized → HTTP `400` `{"error":"Upload session already finalized"}`.
- **Client resume policy in v1:** the reference Mac client performs **no internal retry or resume** — any chunk failure aborts the whole upload. The `expected_offset` resume anchor is defined by the server and a conformant client MAY implement resume-from-anchor, but MUST NOT silently loop; at minimum it MUST surface the failure honestly. (Whether Windows implements active resume or matches the Mac abort-on-failure behavior is an alignment decision, not a wire requirement.)
- Server-side GCS compose fan-in limit is **32** parts. For the 8 MB chunk size this bounds a single composed object; larger files are handled server-side and are not a client concern beyond sending chunks in order.

### 4.3 Poll — `GET /file/{fileId}`

After finalize, the file may still be processing. The client polls file state until it is usable.

- Response body carries a `state`: `PROCESSING`, `ACTIVE`, or `FAILED` (developer_api passes the Files API body verbatim; Vertex returns `{"name","uri","state"}`).
- **Client poll policy (v1):** **24 attempts, 2.0 seconds apart** (~48 s ceiling).
  - `ACTIVE` → proceed to generate.
  - `PROCESSING` → wait `checkInterval` and poll again.
  - Any other state (including `FAILED`) → abort with an honest failure.
  - HTTP `403` → authentication-failed (bundle-id gate).
  - `404` → `{"error":"File not found"}`.
  - Attempts exhausted without reaching `ACTIVE` → timeout error.
- The reference Mac client **swallows** a poll failure on the plain analyze path (proceeds to generate and lets generate surface any problem) but **requires** a successful poll on the schema-extraction path. A conformant client SHOULD require `ACTIVE` before generate; the swallow-on-analyze behavior is a Mac-specific leniency, not a contract requirement.

### 4.4 Then generate

Once the file is `ACTIVE`, call `POST /generate` (§5) with the file referenced as a `file_data` part. In Vertex mode the `file_uri` MUST be a `gs://` URI; a developer_api `files/...` reference in Vertex mode is rejected `400 invalid_vertex_file_reference`.

---

## 5. Generate contract

`POST /generate` proxies a Gemini `generateContent` call. The proxy owns the model and credentials; the client owns the prompt and the file reference.

### 5.1 Request

- Body is JSON. An empty body → `400 {"error":"Invalid JSON body"}`.
- **Only these top-level keys are forwarded:** `contents`, `tools`, `toolConfig`, `safetySettings`, `systemInstruction`, `generationConfig`, `cachedContent` (plus Vertex `labels`). Any other top-level key is dropped.
- **The `model` field is advisory and ignored.** The proxy runs its own chain (`MANAGED_GEMINI_MODEL_CHAIN`, default `["gemini-3.1-flash-lite","gemini-3-flash-preview","gemini-2.5-flash"]`) and retries within it. A client MAY send `model` for readability but MUST NOT depend on it selecting the model.
- Any `proxyTelemetry` envelope is **not forwarded**.
- `generationConfig` keys are accepted in both camelCase and snake_case on read; the proxy normalizes to what the backend needs and, in Vertex mode, **strips `generationConfig.thinkingConfig`**. Clients targeting the managed rename path SHOULD send snake_case `generationConfig` (`response_mime_type`, `max_output_tokens`, etc.) to match the reference client, but MUST NOT rely on `thinkingConfig` surviving.

### 5.2 Parts ordering (rename path)

`contents[0].parts` MUST be, in this order:

1. the text prompt part: `{"text": "<prompt>"}`
2. then the file reference: `{"file_data": {"mime_type": "<mime>", "file_uri": "<uri>"}}`

**Text part first, file part second.** Nothing is inlined; there is no `inline_data` base64 on the managed rename path. Plain rename sends `generationConfig: null` (one-shot). Metadata rename sends a `generationConfig` with `response_mime_type: application/json`, low temperature (0.2 for metadata, 0.0 for structured/preset extraction), a `max_output_tokens` from the output-token policy, and a `responseSchema`.

### 5.3 Representative rename metadata responseSchema

The metadata rename uses an `OBJECT` schema requiring all three fields, with a fixed property ordering:

```json
{
  "type": "OBJECT",
  "properties": {
    "filename":    {"type": "STRING", "minLength": 1, "maxLength": 160},
    "tags":        {"type": "ARRAY",  "items": {"type": "STRING"}},
    "description": {"type": "STRING"}
  },
  "required": ["filename", "tags", "description"],
  "propertyOrdering": ["filename", "tags", "description"]
}
```

(`minLength`/`maxLength` on `filename` are 1..160. The exact tag/description bounds beyond "present and string/array" are **unspecified** here beyond what the schema states.)

### 5.4 Response

- Success is the **upstream `generateContent` body verbatim** plus proxy headers `X-Managed-Gemini-Model`, `X-Managed-Gemini-Attempts`, `X-Managed-Gemini-Backend`, `X-Managed-Gemini-Endpoint-Family`. Upstream errors also pass through verbatim with the same headers.
- A client MUST read the candidate/parts from the standard `generateContent` shape and MUST NOT assume any proxy-added envelope around the body.

### 5.5 One-shot billing semantics

- `/generate` is **one-shot for billing**: there is no billed retry on a model-output failure. The proxy may retry within its model chain for transport/upstream reasons, but the client MUST NOT itself issue a second billed generate to "fix" a bad model output.
- **Metadata recovery ladder (REQUIRED client behavior).** When the metadata response fails to parse against the contract, the client MUST walk this fixed ladder before giving up, and none of these steps is a new billed generate:
  1. **One non-debited semantic retry** — a single retry that is not counted against credits.
  2. **Filename-only degrade** — fall back to extracting just the filename (also non-debited).
  3. **Partial-JSON recovery** — recover a usable filename from a truncated/partial JSON body.
  4. **Fail** — route to review/error with an honest state; never fabricate a value.

### 5.6 Tolerant ingestion boundary (doctrine, REQUIRED)

Model output MUST be ingested through a tolerant boundary: **parse loose, canonicalize, then convert to strict internal types.** A client MUST NOT decode model output straight into strict domain structs on first contact. Parse into raw/loose objects first, canonicalize weak types and shape drift at the boundary, then validate into strict types. Prompt-only tightening is not a substitute for a tolerant ingestion boundary.

---

## 6. Error taxonomy

The proxy returns structured errors. **A conformant client MUST parse the structured `reason` / `error_class` fields and MUST honor `Retry-After` where present.** Mapping on numeric status alone is not conformant (the Mac client does exactly this today — recorded in §9 as an alignment item, not a spec bug).

| HTTP | Body / discriminator | Meaning | Required client handling |
|---|---|---|---|
| `403` | `{"error":"Unauthorized","details":"Invalid bundle ID: <v>"}` | Bundle-id gate failed (missing/unlisted). | Honest "not authorized for managed service" state. Never retry blindly. For a Windows build pre-patch, surface "not available for this build yet". |
| `400` | `{"error":"Invalid JSON body"}` | Empty/invalid `/generate` body. | Fix request; client bug, not transient. |
| `400` | `{"error":"Unexpected upload offset","expected_offset":N}` | Chunk offset mismatch. | Resume from byte `N` (§4.2). |
| `400` | `{"error":"Final upload size does not match session metadata","expected_size":M,"received_bytes":K}` | Finalize size mismatch. | Fatal session; abort and restart upload. |
| `400` | `{"error":"Invalid or expired upload session"}` | Session unknown/expired. | Restart from `/upload/start`. |
| `400` | `{"error":"Upload session already finalized"}` | Chunk after finalize. | Stop chunking; proceed to poll/generate. |
| `400` | `error_class: invalid_vertex_file_reference` | Vertex given a non-`gs://` `file_uri`. | Client bug; use `gs://` uri. |
| `404` | `{"error":"File not found"}` | `GET /file/{id}` unknown. | Abort poll. |
| `404` | `{"error":"Not found","path":"<p>"}` | Unknown route. | Client bug. |
| `429` | `{"error":"Too many requests. Please try again later.","reason":"rate_limited"}` | Caller quota (per-IP/token rate) exceeded. | Back off. Parse `reason`. No `Retry-After` guaranteed. |
| `429` | `{"error":"Too many requests. Please try again later.","reason":"free_quota_exceeded"}` | Free-tier claim quota exhausted. | Distinct user-facing state (free allotment used), not a generic rate limit. |
| `429` | `{"error":"Managed Gemini is busy...","error_class":"traffic_rejected","retry_after_seconds":N}` + `Retry-After: N` | Traffic shaper rejected admission. | Transient. **Honor `Retry-After`**, retry after the delay. |
| `503` | `{"error":"Managed Gemini is temporarily unavailable...","error_class":"traffic_circuit_open","retry_after_seconds":N}` + `Retry-After: N` | Shaper circuit open. | Transient. Honor `Retry-After`. |
| `502` | `{"error":"Managed Gemini request failed","error_class":"<class>","attempted_models":[...],"details":"..."}` | All model attempts failed (network/upstream). | Honest **provider-outage** state; surface `attempted_models` context internally. |
| `504` | (timeout) | Upstream/generate timeout. | Transient temporary-failure. |
| `500` | internal / canary | Proxy internal error. | Temporary-failure; do not loop. |
| `4xx`/`5xx` | upstream passthrough | Verbatim upstream `generateContent` error + `X-Managed-Gemini-*`. | Map on status + any upstream body. |

`error_class` vocabulary (the full set the proxy emits): `success`, `timeout`, `network_error`, `upstream_transient`, `invalid_request`, `unauthorized`, `unknown`, `vertex_auth_failed`, `backend_auth_failed`, `backend_configuration_error`, `traffic_rejected`, `traffic_circuit_open`, `invalid_vertex_file_reference`, `invalid_upload_session`.

`reason` vocabulary (429 caller-quota body): `rate_limited`, `free_quota_exceeded`.

**Transport failures** (connection refused, DNS, TLS, socket): a client MAY perform **at most one** transport-level retry, then surface an honest network error. A client MUST NOT silently reroute to a different provider or a BYOK key when the managed proxy is unreachable.

---

## 7. Credit read and exhaustion

There is **no credit-balance endpoint on gemini-proxy.** Credit read is entirely a polar-proxy (licensing-side) concern; proxy-side exhaustion signaling is the `429` reason codes in §6 (`rate_limited`, `free_quota_exceeded`). The proxy writes spend counters in Firestore post-response from `totalTokenCount`, but those are not client-readable.

### 7.1 Credit read against polar-proxy

- Endpoint: `GET {polar-proxy}/customers/{customerId}/state`.
- Response: a customer-state object containing `activeMeters`, an array of `CustomerMeter` objects. Each meter carries at least `creditedUnits`, `consumedUnits`, and `balance` (plus other fields).
- **Meter selection:** the client selects the meter that matches the customer's ownership/entitlement — the audited meter ids are `aiTokenUsageMeterId` (managed subscription), `aiTrialUsageMeterId` (managed trial), and `byokTrialMeterId` (BYOK trial). Selection is ownership-gated; the client MUST pick the meter its license entitles it to, not the first meter present.
- **Freshness TTL: 300 seconds.** A balance read within the last 300 s is reused; beyond that it is refreshed.
- **`429` handling:** on a rate-limited state read, the client honors the returned retry hint (`nextAllowedSyncAt`, derived from `Retry-After` plus jitter) and does not hammer the endpoint.
- **Offline: fail closed.** If the balance cannot be read and no fresh cache exists, the client denies the operation rather than assuming credit is available.
- **`past_due` / `incomplete` billing state:** keep the last known local balance and surface a non-destructive `syncError`; do not zero the balance on a billing hiccup.

### 7.2 Per-operation affordability and debit

- Before a managed operation, the client runs an affordability check against the selected meter balance. On an apparent shortfall, it MUST perform **exactly one** stale-balance force-refresh before concluding `insufficientCredits` (a stale cached balance must not falsely block a paid customer).
- **Debit occurs only AFTER model success**, never before. A failed generate is not debited.
- **Exhaustion parks the queue; it never drops files.** When credits run out mid-batch, the orchestrator parks the queue tail (the remaining files stay queued, not discarded) and, for an authoritative paid exhaustion, surfaces an upgrade path. A conformant client MUST preserve queued work across an exhaustion event.

---

## 8. Versioning and breaking-change rule

- Every vector and this spec carry a `contract_version`. **`contract_version: 1` = Phase A.** The field is the single discipline point for compatibility.
- The proxy has **no path or header API versioning.** The `nqe1` prefix versions the token format only; `X-Managed-Gemini-Backend` / `-Endpoint-Family` are the closest wire discriminators; client version headers are telemetry-only.
- **Breaking-change gate:** no proxy endpoint change may be deployed unless **both** clients' conformance suites pass against it. This is the mechanism that keeps Mac and Windows in lockstep.
- **Additive vs breaking:**
  - Additive changes (new optional header, new advisory field, a new `error_class` value that clients already treat as "unknown → temporary/honest failure") are a **minor** contract bump.
  - Breaking changes (removing/renaming a forwarded key, changing an error body shape a client parses, changing the client-token derivation, making the entitlement header mandatory) are a **major** contract bump and MUST ship with a migration note in this spec.
- Crossing from Phase A to Phase B (entitlement enforcement) is additive by construction (§3.3): the header was reserved optional in v1, so enabling it does not require a major bump unless/until the proxy begins *rejecting* on it — which would be a major change requiring both suites to pass first.

---

## 9. Divergence register

These are places where the **current Mac client diverges from this spec.** They are alignment work items for the Swift client (executed under #2510 against this spec), **not spec bugs.** The spec states the target behavior both clients must reach; the Windows client should implement the spec directly rather than copy the Mac divergences.

| # | Divergence | Spec requirement (§) | Mac client today | Disposition |
|---|---|---|---|---|
| D1 | **No reason-body parsing.** | §6 — parse `reason`/`error_class`. | `.managedProxy` policy maps on numeric status only; `429` is a generic `rateLimited`, no `rate_limited` vs `free_quota_exceeded` vs `traffic_rejected` distinction. | Swift alignment (#2510). Windows: implement per spec. |
| D2 | **`Retry-After` ignored.** | §6 — honor `Retry-After` on 429/503. | `.managedProxy` sets `honorRetryAfter=false`, `rateLimitMaxRetries=0`. | Swift alignment (#2510). |
| D3 | **No entitlement header.** | §3 — optional, SHOULD send in Phase B. | Mac sends no `X-NameQuick-Entitlement` anywhere; trust = bundle-id + client-token. | Matches v1 wire truth (conformant now). Both clients send once Phase B issuance is wired. |
| D4 | **Retry posture is transport-only.** | §6 — at most one transport retry, honest failure otherwise (conformant); but §6 also requires honoring shaper `Retry-After`. | `maxRetries=1` transport only, `retryableStatusCodes=[]`. The transport-retry posture is conformant; the missing `Retry-After` honoring is D2. | Transport posture conformant; pair with D2 fix. |
| D5 | **Poll failure swallowed on analyze path.** | §4.3 — SHOULD require `ACTIVE` before generate. | Mac proceeds to generate if the poll fails on the plain analyze path (required only on schema extraction). | Mac-specific leniency; Windows SHOULD require `ACTIVE`. |
| D6 | **No internal upload resume.** | §4.2 — `expected_offset` resume anchor exists; resume is MAY. | Mac aborts the whole upload on any chunk failure; never resumes from `expected_offset`. | Conformant (resume is optional). Windows MAY implement resume. |

---

## Appendix A. Endpoint summary

gemini-proxy (bundle-id gated):

| Method | Path | Purpose | Quota scope |
|---|---|---|---|
| `POST` | `/generate` | Managed `generateContent`. | `generate` |
| `POST` | `/upload/start` | Begin resumable upload. | `upload` |
| `POST` | `/upload/chunk` | Send a chunk / finalize. | `upload_chunk` (exempt from daily/spend caps) |
| `GET` | `/file/{fileId}` | Poll file processing state. | not quota'd |
| `OPTIONS` | `*` | CORS preflight → `204`. | n/a |

There is **no** `/health`, no token-issue/refresh, and no credit-balance endpoint on gemini-proxy. Unknown routes → `404 {"error":"Not found","path":"<p>"}`.

polar-proxy (licensing side):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/customers/{customerId}/state` | Read customer state incl. `activeMeters` credit balances. |

Quota reference (Phase A, env-tunable, fail-open, kill switch `CALLER_QUOTAS_ENABLED`): per-IP/min 240 (upload 720); per-IP/day 5000; per-token/day 2500 (trial 300); per-token/day LLM budget 8M tokens (trial 1.5M, peek-only); tokenless-billed per-IP/hr 400; free claims per-IP/hr 900, unknown-op 150/day, per-token free daily 1500. These are server-side and not client-configurable; they surface to clients only as `429` reason codes.
