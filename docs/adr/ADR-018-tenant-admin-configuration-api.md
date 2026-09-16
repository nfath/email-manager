# ADR-018: Tenant / Admin Configuration API Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: api, configuration, multi-tenant, admin

## Context

Several other ADRs in this set define per-tenant configurable behavior that needs a concrete surface: VIP list management ([[adr-011-vip-relationship-graph-scoring]]), explicit user corrections ([[adr-013-user-feedback-correction-loop]]), the `body_content_llm_enabled` DPA gate and export/delete flows ([[adr-014-data-retention-gdpr-ccpa-compliance]]), per-tenant alert threshold tuning and dashboards ([[adr-015-observability-slos]]), and per-tenant LLM spend budgets ([[adr-022-llm-cost-governance-circuit-breaker]]). Per §3 of the research report, category rule overrides must also be tenant-scoped since the internal taxonomy is applied per-mailbox with per-tenant learned overrides (rule weight and sender-override tables per [[adr-013-user-feedback-correction-loop]] and [[adr-008-deterministic-rule-scoring-engine]]).

This ADR consolidates these into a single coherent admin/configuration API surface rather than letting each ADR define its own ad hoc endpoint shape, and defines the authorization model given Microsoft Graph's own permission distinction (delegated vs. application permissions, per §2) that any admin surface must respect when it acts on a tenant's behalf.

## Decision

### API shape
RESTful JSON API, versioned via URL path (`/v1/...`), scoped under tenant:

```
GET    /v1/tenants/{tenant_id}
PATCH  /v1/tenants/{tenant_id}/settings          -- body_content_llm_enabled, region, retention overrides
GET    /v1/tenants/{tenant_id}/mailboxes
GET    /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/status   -- watch health, queue depth, last sync

# VIP management (ADR-011)
GET    /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/vips
POST   /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/vips           { email, note? }
DELETE /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/vips/{email}

# Category rule overrides (ADR-008, ADR-013)
GET    /v1/tenants/{tenant_id}/rule-overrides
PUT    /v1/tenants/{tenant_id}/rule-overrides/{symbol_name}   { weight_multiplier }
POST   /v1/tenants/{tenant_id}/sender-overrides               { sender_domain, category }
DELETE /v1/tenants/{tenant_id}/sender-overrides/{override_id}

# Explicit corrections (ADR-013)
POST   /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/messages/{message_identity_id}/correct
       { corrected_category | null }

# Compliance (ADR-014)
POST   /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/export     -- async job, returns job_id
GET    /v1/tenants/{tenant_id}/jobs/{job_id}
DELETE /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}            -- triggers deletion cascade

# Observability (ADR-015)
GET    /v1/tenants/{tenant_id}/metrics/summary
PATCH  /v1/tenants/{tenant_id}/alert-thresholds

# Cost governance (ADR-022)
GET    /v1/tenants/{tenant_id}/llm-budget
PATCH  /v1/tenants/{tenant_id}/llm-budget   { monthly_cap_usd, alert_thresholds }
```

All mutating endpoints require `Idempotency-Key` header support (mirroring the idempotency design in [[adr-007-message-identity-idempotency]]) since admin actions (e.g., VIP add, override create) must be safely retryable by API clients.

### AuthN/AuthZ model
- API access is via short-lived, tenant-scoped bearer tokens (OAuth2 client-credentials for machine clients, or a session token for the human-facing admin console), never the underlying Gmail/Graph user OAuth tokens themselves — the admin API is a separate trust boundary from the mailbox provider credentials in [[adr-004-oauth-token-storage-credential-vault]].
- Two authorization roles enforced at the API-gateway layer: `tenant_admin` (full read/write on all endpoints above) and `tenant_viewer` (read-only, e.g., for a support/analyst role viewing metrics/status without changing configuration).
- Mirroring Graph's own delegated-vs-application permission distinction (§2): a `tenant_admin` token scoped to act "as an admin managing mailbox settings on behalf of others" is treated internally with the same caution as Graph's `MailboxSettings.ReadWrite` application permission — every write via this role is logged with the acting principal's identity in the immutable audit table ([[adr-014-data-retention-gdpr-ccpa-compliance]]), not just the tenant ID, so a compromised admin credential's actions remain individually attributable.

### Rate limiting and multi-tenancy isolation
- Per-tenant rate limits on the admin API itself (independent of provider rate limits), default 100 req/min per tenant, to prevent one tenant's automation/scripting from degrading the shared control plane — consistent with the multi-tenant isolation principle in [[adr-003-multi-tenant-data-isolation]].

### Versioning and change management
- Config changes that affect classification behavior (rule-override weight changes, sender-override additions) take effect on the **next** classification cycle only, never retroactively reclassifying already-processed mail automatically — a bulk "reclassify historical mail with new settings" is a separate, explicit, rate-limited async job (`POST /v1/tenants/{tenant_id}/mailboxes/{mailbox_id}/reclassify`) requiring confirmation, since it is a potentially expensive and user-visible bulk mutation (re-triggers action-apply layer, §7 Stage 5) and must not happen as a silent side effect of a settings PATCH.

## Consequences

### Positive
- Centralizing configuration under one API prevents each subsystem ADR from inventing incompatible auth/versioning conventions, and gives customers/support one place to reason about tenant state.
- Explicit `reclassify` as a separate, confirmed, rate-limited action prevents an accidental bulk-relabel storm from a routine settings change — a safety property not obviously present if each ADR wired its own "apply immediately" logic.
- Individual-principal audit logging for admin-role actions gives concrete attributability even though authorization is tenant-scoped, addressing a real gap that a naive tenant-only audit trail would leave open.

### Negative
- Centralization means the admin-API service becomes a shared dependency across many subsystems (VIP, corrections, compliance, cost, observability) — a bug or outage here has broad blast radius across otherwise-independent features, requiring it to be built and tested to a higher reliability bar than a single-purpose service.
- Deferring rule/sender-override effects to "next classification cycle only" means customers may be confused if they expect an override to retroactively fix already-labeled mail — requires clear UX/documentation and the explicit reclassify action as the answer, not silent auto-reclassification.

### Neutral
- The specific rate-limit default (100 req/min) and role model (admin/viewer only, no finer-grained per-endpoint RBAC) are a deliberately minimal V1 starting point; finer RBAC (e.g., a role that can manage VIPs but not compliance/export) is a plausible V2+ extension if enterprise customers request it, not built speculatively now.

## Links
- Related: ADR-003 (multi-tenant isolation), ADR-004 (OAuth token vault — separate trust boundary from admin API tokens), ADR-007 (message identity/idempotency — idempotency key pattern reused), ADR-008 (rule/signal scoring engine — override target), ADR-011 (VIP scoring — VIP list CRUD), ADR-013 (feedback loop — correction endpoint and override effects), ADR-014 (compliance — export/delete endpoints and audit logging), ADR-015 (observability — metrics/alert endpoints), ADR-022 (cost governance — budget endpoints)

## Verification

- Contract test suite covering every endpoint's auth-role enforcement (assert `tenant_viewer` tokens are rejected on all mutating endpoints with 403, not silently downgraded to a no-op).
- Integration test: PATCH a rule-override weight, verify an already-classified fixture message's stored category is unchanged until an explicit `reclassify` job is run against it.
- Idempotency test: replay a `POST .../vips` request with the same `Idempotency-Key` and confirm no duplicate VIP row is created.
- Audit test: perform a write as a `tenant_admin`-role token and confirm the audit log entry records the specific acting principal, not merely the tenant ID.
- Load test: exceed the 100 req/min per-tenant admin-API rate limit and confirm the tenant is throttled (429 with `Retry-After`) without affecting other tenants' request budgets.
