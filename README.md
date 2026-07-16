# NameQuick Conformance

The single source of truth for the **managed-proxy client contract** that every NameQuick client must satisfy: the macOS app (Swift) and the new Windows app (Rust + TypeScript). It is a versioned, language-neutral spec plus machine-readable conformance vectors.

`contract_version: 1` = **Phase A** (the currently deployed managed-proxy behavior).

## What is here

| Path | What it is |
|---|---|
| `spec/managed-proxy-client-spec.md` | The versioned prose contract. Written so a stranger can build a conformant client from scratch. Start here. |
| `spec/openapi.yaml` | OpenAPI 3.1 for the gemini-proxy client surface (`/generate`, `/upload/start`, `/upload/chunk`, `/file/{id}`) plus the polar-proxy credit-state endpoint, with every error response shape. |
| `vectors/` | Language-neutral conformance vectors, one scenario per JSON file, with `vectors/README.md` and `vectors/vector.schema.json`. Pure data. |
| `parity-corpus/` | The prompt-assembly byte-equality corpus: format defined, one self-evident example, real population pending a Mac-side snapshot task. |

The two backends the contract covers:

- **gemini-proxy** (`namequick-gemini-proxy`) — managed inference. Owns the model chain and credentials; enforces caller quotas. Bundle-id gated. No credit endpoint.
- **polar-proxy** — licensing side. Owns credit balances (`GET /customers/{id}/state`) and entitlement-token issuance.

## How the two client repos consume this

Both client repos add this repository as a **git submodule** (read-only from the client's side), pinned to a commit:

```
git submodule add https://github.com/JosiMou/NameQuick-Conformance.git third_party/nq-conformance
git submodule update --init --recursive
```

Each client:

1. reads `spec/managed-proxy-client-spec.md` as the behavior it must implement,
2. runs the `vectors/` suite through its own runner (see below),
3. once the Mac-side snapshot task lands, runs the `parity-corpus/` byte-equality gate.

Bumping the pinned submodule commit is how a client adopts a new contract version.

## How a client runs the vectors

Vectors are **data, not code** — this repo decides nothing at runtime. Each client implements its own runner in its own language:

- The runner loads each `vectors/*.json`, validates it against `vectors/vector.schema.json`, then drives the client's own request-builder / response-handler with the vector's `given` fixtures (either via a mock HTTP server seeded with the fixtures, or by feeding fixtures straight into the client's functions).
- It asserts every field in the vector's `expect` block. An assertion the runner does not understand is a **failure**, not a pass (fail-closed), so a client cannot silently skip a requirement.
- A client is conformant for a contract version when it passes every vector carrying that `contract_version`.

`vectors/README.md` documents the `given`/`expect` conventions and indexes all scenarios.

## Versioning and the breaking-change rule

- `contract_version` is the single discipline point. `1` = Phase A. It appears in the spec front matter and in every vector.
- **No proxy endpoint change deploys unless BOTH clients' conformance suites pass against it.** This is what keeps Mac and Windows in lockstep and is the reason this repo exists.
- **Additive** changes (a new optional header, a new advisory field, a new `error_class` value clients already treat as "unknown → honest failure") are a **minor** bump.
- **Breaking** changes (removing/renaming a forwarded key, changing a parsed error body, changing the client-token derivation, making the entitlement header mandatory-to-reject) are a **major** bump and ship with a migration note in the spec.
- Phase A → Phase B (entitlement token enforcement) is additive by construction: the `X-NameQuick-Entitlement` header is reserved optional in v1, observe-only on the server, so enabling issuance does not break v1 clients. See the spec's entitlement section.

## Provenance

Every shape, header, error body, and quota in the spec and vectors is traceable to the audited Phase A ground truth (server `namequick-gemini-proxy` branch `caller-quota-phase-a-car2094`, and the Mac client's proxy + credit-sync services). Where the ground truth is silent, the spec says **unspecified** rather than inventing a value. Error bodies in the vectors are copied byte-for-byte from the proxy source. No secrets or real customer ids appear anywhere; the only cryptographically real value is the client-token in `vectors/client-token-derivation.json`, a SHA-256 of an obviously-fake input, verifiable offline.

## Known divergences (Swift alignment work, not spec bugs)

The current Mac client diverges from this spec in a few places (numeric-status-only error mapping, ignoring `Retry-After`, no reason parsing, no entitlement header). These are recorded in the spec's **Divergence register** as alignment work items for the Swift client (executed under #2510 against this spec). The Windows client should implement the spec directly rather than copy the Mac divergences.
