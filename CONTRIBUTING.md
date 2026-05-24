# Contributing to the AI Forensics Audit Trail Specification

Thanks for considering a contribution. This is an open reference standard — its credibility depends on multi-vendor and auditor participation.

## What we welcome

| Contribution type | How to start |
|---|---|
| **Provider matrix updates** (§9.1) | PR updating `SPEC-v1.0.md`. Cite the provider's public docs in the PR body. |
| **Bug reports** — ambiguous wording, wrong claim about a profiled standard, schema typo | File an issue. Include the §-reference. |
| **Feature proposals** for v1.1+ | File an issue with the `proposal` label. Brief problem statement first, design after triage. |
| **Implementation reports** — "we implemented Level 2 in $stack" | Open a discussion (Discussions tab). We'll track conforming implementations in v1.1. |
| **Reference test vectors** | PR adding to a future `tests/` directory. We'll seed this in v1.0.1. |

## What we don't accept

- Vendor-specific extensions in the core spec. Profile them in a separate document.
- Changes that break Level 1 conformance without a clear migration path.
- "I think we should also capture X" without an auditor use case attached.

## Issue templates

When filing a bug or proposal, include:

1. **Section reference** (e.g., §4.1, §5.3) — what part of the spec is in scope.
2. **The auditor question being served** — every spec primitive must answer one. If none, it probably doesn't belong here.
3. **Current behavior** — what the spec says today (or doesn't say).
4. **Proposed behavior** — what it should say.
5. **Impact** — does this break Level 1 / 2 / 3 conformance?

## PR conventions

- One spec change per PR (don't bundle editorial fixes with normative changes).
- Reference the issue: `Closes #N`.
- For normative changes, summarize the diff in the PR body — "before / after" snippets help reviewers.
- Provider matrix updates: include the public-docs URL you sourced from, and the date you read it.

## Versioning impact

When proposing a change, label it:

- `editorial` — typo, clarification, no implementation impact.
- `minor` — additive (new optional field, new standards profile, new conformance level). Goes into next minor.
- `major` — breaks an existing conforming implementation. Requires a deprecation window.

See [`SPEC-v1.0.md §11`](./SPEC-v1.0.md#11-versioning) for the versioning model.

## Coordination with profiled standards

This spec profiles OCSF, OpenTelemetry GenAI, MITRE ATLAS, SPIFFE/SPIRE, and NIST AI RMF. If a profiled standard changes upstream, this spec is expected to track:

- **OCSF** — proposed AI Agent Activity event class is upstream; track via the linked OCSF discussion (filed shortly).
- **OpenTelemetry GenAI semconv** — track via the OTel community SIG.
- **MITRE ATLAS** — annual taxonomy update; the `request_metadata.atlas_tags[]` pattern absorbs new technique IDs without spec change.
- **SPIFFE/SPIRE** — Level 3 dependency.
- **NIST AI RMF** — mapping table in §7 only; no schema impact.

Upstream tracking issues live as `tracking:<standard>` labels in this repo.

## Code of conduct

Be civil, technical, and specific. We're trying to build infrastructure regulators and auditors can trust — that requires precision and good faith. Bad-faith participation (vendor sniping, manufactured urgency, etc.) will be removed.

## Licensing of contributions

By submitting a contribution, you agree that:

- **Prose / documentation contributions** are licensed under [CC-BY-4.0](./LICENSE-DOCS).
- **JSON schema / reference code contributions** are licensed under [Apache-2.0](./LICENSE).

You retain copyright; you grant the project a non-exclusive license under the above terms.

## Contact

- Spec questions: file an issue.
- Sensitive coordination (e.g., responsible disclosure for spec ambiguity that enables evasion): security@ai-identity.co
