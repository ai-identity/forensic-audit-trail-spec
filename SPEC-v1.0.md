# AI Forensics Audit Trail Specification

**Status:** Reference Specification — Draft v1.0
**Published:** 2026-05-15
**License:** CC-BY-4.0 (this document) · Apache-2.0 (reference implementation + schemas)
**Editor:** [AI Identity](https://www.ai-identity.co/)

> An open reference standard for tamper-evident, cryptographically-signed audit trails of autonomous AI agents. Profiles OCSF, OpenTelemetry GenAI, MITRE ATLAS, SPIFFE, and NIST AI RMF as a single coherent schema for forensic reconstruction — verifiable offline by auditors with no vendor dependency.

---

## §1. Scope and Goals

This specification defines a minimum coherent schema for capturing, hash-chaining, signing, and replaying the activity of autonomous AI agents in a way that supports **post-incident forensic reconstruction**. It is a *profile* over existing open standards (OCSF, OpenTelemetry GenAI, MITRE ATLAS, SPIFFE/SPIRE, NIST AI RMF), not a competing schema.

### Goals

1. **Tamper-evidence** — any post-hoc modification of a captured event must break the chain in a way an offline verifier detects.
2. **Cryptographic non-repudiation** — a session's audit range is signed by a key held in hardware (KMS/HSM), so the operator cannot silently rewrite history.
3. **Offline verification** — an auditor with only the envelope + the public JWKS can verify the chain and its signature without calling the operator's servers.
4. **Vendor neutrality** — the schema is implementable by anyone; the reference implementation (AI Identity) is one of many possible conforming implementations.
5. **Forensic replay** — a verifier with the chain can reconstruct the agent's decision path (request → policy evaluation → upstream call → response) in order.

### Non-goals (v1.0)

- Runtime threat prevention (covered by AI firewall / prompt-injection-defense vendors)
- Real-time policy evaluation (covered by gateway / proxy products)
- Multimodal context capture (deferred to v1.1 — see §9.2)
- Cross-organization agent identity federation (deferred to v1.2 — see §9.3)

## §2. Terminology

- **Agent** — an autonomous software identity that issues requests on behalf of a user or process. Identified by a per-agent credential (`aid_sk_…` style key or X.509-SVID).
- **Session** — a contiguous time window of agent activity, terminated by an explicit close, timeout, or operator-triggered seal.
- **Audit event** — a single record of one observable agent action (request, policy evaluation, upstream call, response).
- **Chain entry** — an audit event plus its hash linkage to the previous entry.
- **Attestation** — a DSSE envelope signed at session close over the range of chain entries the session covers.
- **Verifier** — any party (auditor, regulator, customer) holding the attestation and the operator's public JWKS, performing offline integrity checks.

## §3. The Four Pillars

The spec organizes forensic primitives into four pillars. Each pillar maps to a specific question an auditor will ask after an incident.

| Pillar | The auditor's question | Spec capability |
|---|---|---|
| **Identity** | Who is this agent? | Per-agent API keys, lifecycle management, scoped credentials. |
| **Policy** | What is it allowed to do? | Fail-closed gateway, deny-by-default policy evaluation. |
| **Compliance** | Can we prove rules were followed? | DSSE-signed session attestations, HMAC-verifiable audit logs, automated compliance assessments. |
| **Forensics** | What happened, provably? | Hash-chained logs, incident replay, export, offline cryptographic verification. |

The chain entry schema (§4) carries the data for all four pillars in one row; the attestation (§5) commits to a contiguous range of those rows.

## §4. Chain Entry Schema

Each captured event is one chain entry. Fields are defined in `schemas/forensic-agent-event/v1.json` (JSON Schema 2020-12). Required fields:

| Field | Type | Description |
|---|---|---|
| `id` | integer (monotonic, per-operator) | Global sequence number. |
| `org_id` | UUID | Tenant/organization isolation key. |
| `agent_id` | UUID | The acting agent identity. |
| `agent_name` | string \| null | Human-readable label at capture time (may diverge from current agent record). |
| `user_id` | UUID \| null | Initiating principal, if applicable. |
| `correlation_id` | string \| null | Cross-system trace correlator (matches OTEL trace ID convention where possible). |
| `endpoint` | string | Upstream URL the agent called. |
| `method` | string | HTTP method or RPC verb. |
| `decision` | enum (`allow` \| `deny` \| `error`) | Gateway decision at evaluation time. |
| `cost_estimate_usd` | decimal \| null | Inferred or reported cost of the call. |
| `latency_ms` | integer \| null | End-to-end latency the gateway observed. |
| `request_metadata` | object | Provider-specific request envelope: model ID, system fingerprint, request ID, policy version, ATLAS tags. **MUST NOT contain raw secrets or PII payloads** — only structured metadata. |
| `entry_hash` | hex string (SHA-256) | HMAC-SHA256 hash over the canonical serialization of all other fields plus `prev_hash`. |
| `prev_hash` | hex string (SHA-256) | The `entry_hash` of the previous chain entry. The first entry of a per-operator chain uses a zero-byte seed. |
| `prev_hash_org` | hex string (SHA-256) \| null | Optional per-org chain link (see §4.2). |
| `entry_hash_org` | hex string (SHA-256) \| null | Optional per-org chain hash. |
| `org_chain_seq` | integer \| null | Per-org sequence number, monotonic within `org_id`. |
| `created_at` | timestamp (RFC 3339, UTC) | Capture time as observed by the gateway, not the agent's clock. |

### §4.1 Canonical serialization

`entry_hash` is computed over the JSON serialization of all fields **except** `entry_hash` itself and `entry_hash_org`, with:

- Keys sorted lexicographically
- UTF-8 encoding
- No leading/trailing whitespace
- `null` fields included explicitly
- The string `prev_hash` field included verbatim

This is RFC 8785 (JCS) canonicalization. Implementations MUST use a canonicalization library to avoid subtle hash drift across stacks.

### §4.2 Per-org chains (optional, recommended for multi-tenant operators)

Multi-tenant operators SHOULD maintain a parallel per-org chain (`prev_hash_org`, `entry_hash_org`, `org_chain_seq`) in addition to the global chain. This allows an auditor representing one tenant to verify their slice of activity without seeing other tenants' rows.

The per-org chain uses the same canonical hashing rules as the global chain, scoped to entries with matching `org_id`.

## §5. Session Attestation

At session close, the operator MUST produce a **session attestation** — a DSSE (Dead Simple Signing Envelope) envelope signed by a key held in hardware (KMS/HSM/Secure Enclave).

### §5.1 Signed payload

The DSSE `payload` is a JSON object with:

| Field | Type | Description |
|---|---|---|
| `spec_version` | string | This document's version (e.g., `"forensic-audit-trail/v1.0"`). |
| `org_id` | UUID | Tenant isolation key. |
| `session_id` | UUID | The session being closed. |
| `first_audit_id` | integer | First chain entry covered. |
| `last_audit_id` | integer | Last chain entry covered. |
| `event_count` | integer | Count of entries the signer walked at signing time. |
| `audit_log_ids` | array of integers | Resolved entry IDs present at sign time. Purge-resilient (see §5.4). |
| `chain_head_hash` | hex string (SHA-256) | `entry_hash` of `last_audit_id`. |
| `signed_at` | timestamp (RFC 3339, UTC) | When the attestation was signed. |

### §5.2 Signing algorithm

- **Algorithm:** ECDSA P-256 with SHA-256 (`SHA256withECDSA` / JWS `ES256`)
- **Envelope:** DSSE v1.0 (`payloadType: "application/vnd.ai-identity.forensic-audit-trail+json"`)
- **Key custody:** Signing key MUST NOT leave hardware (KMS-backed). Public key MUST be retrievable via a stable JWKS URL.

Implementations MAY support additional algorithms (Ed25519, ML-DSA for PQC readiness); ECDSA P-256 is the **mandatory baseline** for v1.0 interoperability.

### §5.3 Public key distribution (JWKS)

Operators MUST publish a stable JWKS endpoint listing the public keys used for attestation signing. Keys MUST include:

- `kid` — stable key identifier referenced from the DSSE envelope's `keyid`
- `alg` — signing algorithm
- `use: "sig"`
- Validity window (`nbf`, `exp`) if rotated

Recommended JWKS path: `/.well-known/forensic-audit-trail-jwks.json`.

### §5.4 Retention coordination

`audit_log_ids` resolves the chain entry IDs present at signing time. This protects against silent truncation:

> If the operator's retention policy later prunes some of these rows, the attestation still tells the verifier the chain existed at sign time, with N rows missing now — instead of silently accepting a shortened chain as authoritative.

Operators MUST NOT prune a chain entry without preserving the attestations that cover it. If retention requires pruning attested rows, the operator MUST record the prune event as a new chain entry (`decision: "error"`, `endpoint: "internal://retention/prune"`).

## §6. Verifier Algorithm

A conforming verifier, given (a) the DSSE envelope, (b) the chain entries the envelope claims to cover, and (c) the operator's JWKS:

1. Fetch the public key from JWKS using the envelope's `keyid`.
2. Verify the DSSE signature over the canonicalized payload.
3. For each chain entry in `audit_log_ids`, recompute `entry_hash` using §4.1 canonicalization and the stored `prev_hash`.
4. Verify each computed `entry_hash` matches the stored value.
5. Verify the last entry's `entry_hash` matches `chain_head_hash` in the signed payload.
6. If any chain entry referenced by `audit_log_ids` is absent from the supplied chain, emit a structured warning: `MISSING_ENTRY id=N (chain existed at sign_time)`.
7. If a per-org verification (§4.2), also verify the per-org chain in parallel using the org-scoped fields.

A verifier MUST emit a clear pass/fail and a structured list of any warnings. A reference implementation is provided in the AI Identity `aid verify` CLI (Apache-2.0).

## §7. Standards Profiled

This spec profiles existing standards rather than inventing new ones:

| Standard | Role | Profile note |
|---|---|---|
| [OCSF](https://github.com/ocsf/ocsf-schema) | Event envelope | Forensic chain entries SHOULD also emit as OCSF events for SIEM ingestion. AI Agent Activity event class proposed upstream. |
| [OpenTelemetry GenAI semconv](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai) | Span attributes | Tool calls, model invocations, and policy evaluations SHOULD emit OTEL spans following GenAI semconv; `correlation_id` is the OTEL trace ID. |
| [MITRE ATLAS](https://atlas.mitre.org/) | Threat taxonomy | `request_metadata.atlas_tags[]` carries ATLAS technique IDs (e.g., `T0051`, `T0040`) for SIEM correlation. |
| [SPIFFE / SPIRE](https://spiffe.io/) | Workload identity | Per-agent X.509-SVID or JWT-SVID identity binding is optional in v1.0; required for cross-trust-domain federation (v1.2). |
| [NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework) | Governance mapping | Chain primitives map to MANAGE 4 (incident response) and MEASURE 2.7 (decision traceability). |
| IETF Agent Identity Protocol (AIP) draft | Identity federation | Tracking upstream; informs v1.2 federation profile. |

## §8. Conformance Levels

A conforming implementation declares one of three levels:

- **Level 1 — Chain capture.** All §4 required fields present; `entry_hash` chain valid; canonical serialization (§4.1) implemented.
- **Level 2 — Session attestation.** Level 1 + DSSE attestations per §5; ES256 minimum; public JWKS published per §5.3.
- **Level 3 — Federation-ready.** Level 2 + per-org chains (§4.2) + SPIFFE/SPIRE identity binding for cross-domain agents.

AI Identity's reference implementation ships at Level 2 today. Level 3 is targeted for v1.2.

## §9. Open Questions

A specification that hides its uncertainty is not a standard — it's a marketing document. These are the questions v1.0 leaves open, with v1.1 and v1.2 commitments where applicable.

### §9.1 Per-provider logprob exposure

Token-level log-probabilities are the strongest forensic signal — they let a verifier replay sampling decisions deterministically. Not every provider exposes them. This matrix tracks the state of public APIs as of 2026-05.

| Provider | Logprobs exposed | Detail | Forensic sufficiency |
|---|---|---|---|
| **OpenAI** | Yes | `logprobs` parameter returns top-N log-probabilities per token. Up to 5 alternatives via `top_logprobs`. | Yes |
| **Anthropic** | No (as of 2026-05) | No public logprob exposure on the Messages API. Forensic attestations rely on response provenance (`model_id`, system fingerprint, `request_id`) instead. | Partial |
| **Google Gemini** | Yes | `logprobs` and `responseLogprobs` available on the GenerateContent API. Up to 5 alternatives. | Yes |
| **AWS Bedrock** | Varies by model | Claude on Bedrock: no logprobs (parity with Anthropic direct). Other models: provider-dependent. | Partial |
| **Self-hosted (vLLM, TGI, llama.cpp)** | Yes | Full logprob distribution exposed. Strongest forensic signal — verifier can replay token-by-token sampling decisions. | Full |

**Fallback when logprobs unavailable.** Verifiers SHOULD treat `request_metadata.provider_response_id` + `model_id` + `system_fingerprint` (when present) as the integrity anchor. This is weaker than logprob replay — it proves *the operator captured what the provider returned*, but not *the provider's sampling was deterministic*.

We track upstream changes here. Contributions welcome (file a PR updating this matrix with a citation to the provider's public docs).

### §9.2 Multimodal context capture

Image, audio, and video inputs are out of scope for v1.0. v1.1 plan in progress — likely capture content-hash + MIME + size rather than raw bytes, with optional vendor-specific perceptual hashes.

### §9.3 Cross-organization federation

Agent identity assertions across trust domains depend on IETF AIP draft maturity. v1.0 assumes single-org trust domain; federation profile deferred to v1.2.

## §10. Reference Implementation

[AI Identity](https://www.ai-identity.co/) ships an open-source reference implementation at Level 2 of §8 conformance. Source code lives at https://github.com/Levaj2000/AI-Identity. Components:

- **Chain capture** — `common/audit/writer.py`
- **Per-org chain support** — `common/audit/correlation.py`
- **Attestation signer** — `common/forensic/signer.py`
- **JWKS publisher** — `common/forensic/jwks.py`
- **Offline verifier CLI** — `aid verify` (see CLI docs)

## §11. Versioning

This spec uses semantic versioning at the schema level:

- **Major (`vN.0`)** — breaking changes to chain entry schema, attestation envelope, or verifier algorithm.
- **Minor (`vN.M`)** — additive fields, new optional standards profiles, new conformance level definitions.
- **Patch (`vN.M.P`)** — editorial corrections, clarifications.

Implementations declare the spec version they conform to via the `spec_version` field in the attestation payload (§5.1).

## §12. Acknowledgements

This specification draws on prior work from OCSF, OpenTelemetry, SPIFFE, NIST, and the W3C Verifiable Credentials community. It is published as an open standard so that auditing AI agent behavior is not a vendor-lock-in problem.

---

**Contact:** Issues and proposals → [github.com/ai-identity/forensic-audit-trail-spec/issues](https://github.com/ai-identity/forensic-audit-trail-spec/issues)
**Spec home:** [www.ai-identity.co/spec](https://www.ai-identity.co/spec)
