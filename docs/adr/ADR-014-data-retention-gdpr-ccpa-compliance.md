# ADR-014: Data Retention, Content Minimization & GDPR/CCPA Compliance Posture

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: compliance, privacy, gdpr, ccpa, retention, data-minimization

## Context

Per §5 of the research report, 8 of the 11 taxonomy categories are resolvable using header/metadata signals alone (List-Unsubscribe, List-Id, Precedence, Auto-Submitted, calendar METHOD, schema.org JSON-LD, sender-domain allowlists) — only phishing (residual cases), needs-reply, and ambiguous newsletter/promo splits genuinely require body content sent to an LLM. Per §8, the report frames this explicitly as a **content minimization strategy**: "prefer metadata/header-only processing wherever a category can be resolved... when body content must reach LLM, treat as passing through a data processor and paper with a DPA under GDPR."

The report also flags (§8, confidence "Medium-high") that GDPR/CCPA *mechanics* are well-established but whether this specific design is "DPA-sufficient" is a legal judgment call requiring actual legal review before launch, not just engineering best-effort — this ADR sets the engineering posture that legal review is validated against, not a substitute for it.

Prior art (§6) corroborates the approach: both SaneBox and Clean Email ship metadata-only as a stated privacy feature and reach useful categorization for the majority of their taxonomy without body access, which is direct precedent that the 8/11 split is achievable in practice, not just in theory.

## Decision

### Data classification tiers
Define three tiers of data the system touches, each with different retention/access rules:

| Tier | Contents | Example | Default retention |
|---|---|---|---|
| **T1 — Header/Metadata** | From/To/Subject/Date, `List-*`, `Precedence`, `Auth-Results`, MIME structure, schema.org JSON-LD | Used by Stage 2/3 rule engine for 8/11 categories | Indefinite (needed for ongoing sender-reputation/reciprocity scoring per [[adr-011-vip-relationship-graph-scoring]]), subject to tenant-configurable cap, default 18 months |
| **T2 — Body content (transient)** | Full MIME body, attachments metadata (not attachment binary) | Sent to LLM only for the ambiguous band (needs-reply, residual phishing, promo/newsletter split) per [[adr-009-llm-adjudication-service]] and [[adr-012-phishing-detection-subsystem]] | **Not persisted** beyond the single classification call — passed in-memory to the LLM request and discarded; only the LLM's structured output (category, confidence, reasoning symbols) is persisted, never the raw body |
| **T3 — Classification output & corrections** | Category, confidence, source (rule/llm/native), correction history | Per [[adr-013-user-feedback-correction-loop]] | 24 months default, tenant-configurable, included in export/delete flows |

**Default posture: T1-only processing.** The pipeline only escalates a message into T2 (body sent to LLM) when Stage 3 rule scoring lands the message in the ambiguous band ([[adr-008-deterministic-rule-scoring-engine]], [[adr-012-phishing-detection-subsystem]]). This is enforced structurally, not by policy alone: the LLM adjudication service's API contract ([[adr-009-llm-adjudication-service]]) requires a `stage3_ambiguous: true` flag set by the rule engine before body content is attached to a request — the LLM client library refuses to serialize a body field without it, as a fail-closed guard.

### DPA and processor relationship
- Anthropic (or whichever LLM provider is configured, see [[adr-009-llm-adjudication-service]]) is contractually a **data processor** for the T2 body-content path. A signed DPA is a hard prerequisite for enabling body-content escalation for any tenant operating under GDPR/CCPA scope; the tenant config API ([[adr-018-tenant-admin-configuration-api]]) exposes a `body_content_llm_enabled` flag that defaults to `false` until the tenant's jurisdiction/DPA status is confirmed at onboarding.
- Batch API usage (per §5 cost strategy) is documented separately in the DPA scope since it involves asynchronous provider-side storage of submitted batch jobs for up to 29 days per current Anthropic Batch API terms — this is disclosed explicitly to tenants as part of the T2 data flow, not silently assumed equivalent to synchronous calls.

### Data subject rights (export / delete)
- `GET /tenants/{id}/mailboxes/{id}/export` produces a structured export of all T1 metadata and T3 classification/correction records the system holds for that mailbox (never T2, since it is not persisted).
- `DELETE /tenants/{id}/mailboxes/{id}` cascades: revokes OAuth tokens ([[adr-004-oauth-token-storage-credential-vault]]), purges T1/T3 records within 30 days (GDPR Art. 17 "without undue delay," operationalized as a 30-day hard SLA enforced by a scheduled purge job), and removes any per-tenant LLM few-shot examples derived from that mailbox's corrections ([[adr-013-user-feedback-correction-loop]]).
- Deletion requests and their completion timestamps are logged in an immutable audit table (append-only, per [[adr-021-schema-taxonomy-versioning]] migration constraints) retained for 7 years to demonstrate compliance regardless of the underlying data's own deletion.

### Regional data residency
- State store deployments support **region-pinned tenants** (EU-resident tenant data stored and processed in an EU region) as a configuration at tenant-provisioning time, per [[adr-016-deployment-topology-infrastructure]]. This is a V2+ capability — V1 launches single-region with residency as an explicit, disclosed limitation, not a silent gap.

## Consequences

### Positive
- Structural (not just policy) enforcement of the T1-default/T2-exception boundary means a code review or automated test can verify the invariant, rather than relying on discipline alone.
- Aligns directly with validated prior art (SaneBox/Clean Email) rather than an unproven design, reducing both privacy risk and legal review friction.
- 30-day deletion SLA and immutable audit trail give a concrete, verifiable answer to a DPA/customer security-questionnaire ask, rather than a vague "we delete data when requested."

### Negative
- `body_content_llm_enabled` defaulting to `false` means needs-reply and residual-phishing categories are degraded (rule-engine-only, lower recall) for any tenant until DPA/jurisdiction status is explicitly confirmed — an onboarding friction point that must be communicated to customers, not hidden.
- Batch API's provider-side retention window (up to 29 days) is a data flow outside direct system control and must be re-verified against current Anthropic terms at implementation time and on any provider contract renewal.
- Region-pinning is deferred to V2, meaning V1 cannot serve EU tenants requiring strict data residency — a real go-to-market constraint, not just a technical footnote.

### Neutral
- The "DPA-sufficient" determination is explicitly a legal call per the research report (§8) — this ADR documents the engineering posture legal counsel reviews, and is not itself a substitute for that review. Status remains `proposed` pending legal sign-off, tracked separately from architectural acceptance.

## Links
- Related: ADR-009 (LLM adjudication service — enforces the `stage3_ambiguous` fail-closed guard), ADR-012 (phishing detection — primary T2 consumer), ADR-013 (feedback loop — T3 retention and export/delete cascade), ADR-004 (token vault — revocation on delete), ADR-016 (deployment topology — region pinning), ADR-021 (schema versioning — audit table immutability), ADR-003 (multi-tenant isolation — per-tenant DPA/residency flags)

## Verification

- Automated test: attempt to construct an LLM adjudication request with body content but without `stage3_ambiguous: true` set; assert the client library rejects serialization (fail-closed contract test).
- Scheduled job test: seed a mailbox, issue a delete request, assert all T1/T3 rows are purged and OAuth tokens revoked within the 30-day job window (accelerated in test via time-travel/fast-forward harness).
- Audit: quarterly automated report of `body_content_llm_enabled` tenant count vs. confirmed-DPA tenant count — any mismatch is a release-blocking finding.
- Legal review checkpoint: formal DPA/legal sign-off required before flipping `body_content_llm_enabled` default or launching in any new jurisdiction — tracked as a release gate, not an engineering-only verification step.
