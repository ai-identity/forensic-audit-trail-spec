# AI Forensics Audit Trail Specification

[![Spec](https://img.shields.io/badge/spec-v1.0--draft-A6DAFF)](./SPEC-v1.0.md) [![License: CC BY 4.0](https://img.shields.io/badge/spec_license-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/) [![License: Apache 2.0](https://img.shields.io/badge/code_license-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)

> An open reference standard for tamper-evident, cryptographically-signed audit trails of autonomous AI agents.

**Spec home:** [www.ai-identity.co/spec](https://www.ai-identity.co/spec)
**Status:** Draft v1.0 — published 2026-05-15. Public review open.

---

## What this is

A minimum coherent schema for capturing, hash-chaining, signing, and replaying the activity of autonomous AI agents in a way that supports **post-incident forensic reconstruction**.

It is a *profile* over existing open standards (OCSF, OpenTelemetry GenAI, MITRE ATLAS, SPIFFE/SPIRE, NIST AI RMF) — not a competing schema. The goal is that an auditor with only the signed envelope and the operator's public JWKS can verify what an AI agent did, **offline, with no vendor dependency**.

## Why it exists

When an autonomous AI agent does something wrong — leaks data, runs up a bill, takes an action it shouldn't have — the questions a regulator, customer, or court will ask are predictable:

1. **Who** was the agent?
2. **What** did it try to do?
3. **Why** was that allowed?
4. **Can you prove the record hasn't been altered since?**

Today, each AI platform vendor answers these questions in its own format, with its own tooling, often requiring vendor-side cooperation to verify. That isn't auditable infrastructure — that's a press release with logs.

This spec fixes the format so any conforming implementation produces evidence any auditor can verify.

## What's in this repo

| File | Purpose |
|---|---|
| [`SPEC-v1.0.md`](./SPEC-v1.0.md) | The normative specification. |
| [`schemas/forensic-agent-event/v1.json`](./schemas/forensic-agent-event/v1.json) | Machine-readable JSON Schema (2020-12) for one chain entry. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to propose changes, file issues, or contribute provider matrix updates. |
| [`LICENSE`](./LICENSE) | Apache 2.0 — covers the JSON Schema and any reference code. |
| [`LICENSE-DOCS`](./LICENSE-DOCS) | CC-BY-4.0 — covers `SPEC-v1.0.md`, this README, and `CONTRIBUTING.md`. |

## Quick read

The spec organizes forensic primitives into four pillars. Each maps to a specific question an auditor asks after an incident:

| Pillar | The auditor's question | Spec capability |
|---|---|---|
| **Identity** | Who is this agent? | Per-agent API keys, lifecycle management, scoped credentials. |
| **Policy** | What is it allowed to do? | Fail-closed gateway, deny-by-default policy evaluation. |
| **Compliance** | Can we prove rules were followed? | DSSE-signed session attestations, HMAC-verifiable audit logs. |
| **Forensics** | What happened, provably? | Hash-chained logs, incident replay, export, offline cryptographic verification. |

See [SPEC-v1.0.md §3](./SPEC-v1.0.md#3-the-four-pillars) for the full table; [§4](./SPEC-v1.0.md#4-chain-entry-schema) for the chain entry schema; [§5](./SPEC-v1.0.md#5-session-attestation) for the DSSE attestation format.

## Conformance levels

A conforming implementation declares one of three levels:

- **Level 1** — Chain capture. All required fields present; HMAC chain valid; RFC 8785 canonical serialization.
- **Level 2** — Session attestation. Level 1 + DSSE-signed session attestations; ES256 minimum; public JWKS published.
- **Level 3** — Federation-ready. Level 2 + per-org chains + SPIFFE/SPIRE identity binding for cross-domain agents.

See [§8](./SPEC-v1.0.md#8-conformance-levels).

## Reference implementation

[AI Identity](https://www.ai-identity.co/) ships an open-source reference implementation at Level 2. Source: [github.com/Levaj2000/AI-Identity](https://github.com/Levaj2000/AI-Identity). The reference implementation is one of many possible conforming implementations — this spec is intentionally vendor-neutral.

## Standards profiled

This spec profiles existing standards rather than inventing new ones:

- [OCSF](https://github.com/ocsf/ocsf-schema) — event envelope; AI Agent Activity event class proposed upstream
- [OpenTelemetry GenAI semconv](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai) — span attributes
- [MITRE ATLAS](https://atlas.mitre.org/) — threat taxonomy tags
- [SPIFFE / SPIRE](https://spiffe.io/) — workload identity (optional v1.0, required for Level 3)
- [NIST AI RMF 1.0](https://www.nist.gov/itl/ai-risk-management-framework) — governance mapping
- IETF Agent Identity Protocol (AIP) draft — federation track (v1.2)

## Roadmap

| Version | Theme | Status |
|---|---|---|
| **v1.0** | Single-org chain capture + session attestation | **Draft, public review open** |
| v1.1 | Multimodal context capture; expanded provider matrix | Planning — see [§9.2](./SPEC-v1.0.md#92-multimodal-context-capture) |
| v1.2 | Cross-organization federation profile (SPIFFE + IETF AIP) | Tracking upstream — see [§9.3](./SPEC-v1.0.md#93-cross-organization-federation) |

## Get involved

- **File a spec issue:** [github.com/ai-identity/forensic-audit-trail-spec/issues](https://github.com/ai-identity/forensic-audit-trail-spec/issues)
- **Discuss on OCSF:** AI Agent Activity event class proposal (link pending)
- **Become a design partner:** [www.ai-identity.co/contact?intent=design-partner](https://www.ai-identity.co/contact?intent=design-partner)

## License

- **Specification text** (`SPEC-v1.0.md`, `README.md`, `CONTRIBUTING.md`): [Creative Commons Attribution 4.0](./LICENSE-DOCS)
- **JSON Schema + reference code** (`schemas/`, any future `examples/`): [Apache License 2.0](./LICENSE)

This split lets you (a) implement the spec in any product without copyleft concerns, (b) quote and adapt the specification text freely with attribution.
