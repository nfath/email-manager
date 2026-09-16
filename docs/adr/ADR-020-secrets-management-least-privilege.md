# ADR-020: Secrets Management & Least-Privilege Scope Governance

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: security, secrets, oauth, least-privilege, scope-governance

## Context

Per §1 of the research report, the minimum viable Gmail scopes are `gmail.modify` (restricted, requires Google CASA security assessment), `gmail.labels`, and `contacts.readonly`/`contacts.other.readonly` — explicitly avoiding the overly broad `mail.google.com` scope. Per §2, the Graph-side minimum scopes are `Mail.ReadWrite`, `MailboxSettings.ReadWrite` (a **tenant-wide-by-default application permission** that should be narrowed via application access policies to only onboarded mailboxes), `Contacts.Read`, and `offline_access`. Per §8, the report's security section states plainly: "encrypt refresh tokens at rest... never in client-side/browser storage," "rotate on refresh, treat as short-lived credentials," and flags CASA review timeline and Microsoft's admin-console scoping mechanics as things to "budget time for" / "re-verify... at implementation time" (Confidence: Medium on exact application-access-policy mechanics, since Microsoft evolves Entra controls frequently).

This ADR defines the concrete secrets-management and scope-governance implementation that operationalizes §1/§2/§8's requirements; token *storage/vault schema* is defined in [[adr-004-oauth-token-storage-credential-vault]] — this ADR focuses on the surrounding governance: which secrets exist, who/what can access them, how scope minimization is enforced and audited over time (not just declared once at launch).

## Decision

### Secret inventory and storage tiers
| Secret class | Storage | Access pattern |
|---|---|---|
| OAuth app client secrets (Google/Microsoft app registration credentials) | Cloud secrets manager (GCP Secret Manager for Google-side, Azure Key Vault for Microsoft-side, or a single provider's vault if consolidating — decision deferred to implementation, not blocking this ADR) | Read-only at service startup, cached in-memory, never logged, rotated on a 90-day schedule |
| Per-user/per-mailbox OAuth refresh tokens | Encrypted at rest in the Postgres token vault (application-layer envelope encryption, AES-256-GCM, with the data-encryption-key itself stored in the secrets manager and wrapped per-tenant), per [[adr-004-oauth-token-storage-credential-vault]] | Decrypted only in-memory within the `mailbox-worker`/ingestion services at point of use, never returned via any API response, never logged |
| Access tokens (short-lived, ~1hr per §2) | In-memory only, never persisted to disk or DB | Refreshed on-demand from the refresh token; expiry enforced client-side with a safety margin (refresh at 80% of lifetime) |
| Admin API service credentials (client-credentials for machine clients, per [[adr-018-tenant-admin-configuration-api]]) | Secrets manager, tenant-scoped, 90-day rotation | Standard OAuth2 client-credentials flow |
| Batch-processing/LLM API keys | Secrets manager, single shared key per environment (not per-tenant, since the LLM provider relationship is at the platform level, not per-tenant) | Loaded at `llm-adjudicator` service startup, never exposed via any tenant-facing API |

### Least-privilege scope enforcement
- **Google side**: application requests exactly `gmail.modify`, `gmail.labels`, `contacts.readonly`, `contacts.other.readonly` — no broader scope is ever requested in any OAuth consent flow, enforced by a single centralized scope-list constant consumed by the OAuth client library (not duplicated/hand-typed at each call site, to prevent scope drift from an ad hoc addition).
- **Microsoft side**: delegated permissions `Mail.ReadWrite`, `Contacts.Read`, `offline_access` are requested per-user at consent time. The `MailboxSettings.ReadWrite` **application** permission (needed for categories/rules/Focused-Inbox overrides on behalf of a signed-in user acting within their own mailbox context still generally works via delegated permission; application permission is only needed for service-to-service/admin-consented multi-mailbox scenarios) is scoped down via an **Entra application access policy** restricting it to only the specific mailboxes onboarded by each tenant — never tenant-wide by default. This exact admin-console mechanic is flagged (§2, §8) as needing re-verification against current Microsoft documentation at implementation time; this ADR mandates the *goal* (narrowed scope, not tenant-wide), with the specific portal/API steps validated and documented as a runbook at implementation time rather than frozen into this ADR.
- CASA security assessment (required for Google's restricted `gmail.modify` scope, per §1) is tracked as an explicit project milestone with lead time budgeted into the roadmap — flagged here so it isn't discovered as a launch blocker late.

### Token lifecycle governance
- Refresh tokens are rotated on every use where the provider supports rotation (Google issues a new refresh token on some flows; Microsoft's `offline_access` refresh tokens are also rotated per MSAL default behavior) — the vault always stores only the most recent token, previous versions invalidated, per [[adr-004-oauth-token-storage-credential-vault]].
- Revocation is triggered on: explicit tenant/mailbox deletion ([[adr-014-data-retention-gdpr-ccpa-compliance]]), detected persistent auth failure (e.g., user revoked consent externally — detected via repeated `invalid_grant`/401 responses), and manual admin action via the tenant API.
- No refresh token is ever transmitted to or stored in any client-side/browser context — the OAuth consent flow's authorization code exchange happens entirely server-side, consistent with §8's explicit guidance.

### Scope-drift auditing
- A scheduled job (quarterly minimum) enumerates the actual OAuth scopes granted for a sample of active tenant connections (via Google's tokeninfo endpoint / Microsoft Graph's `/me/oauth2PermissionGrants` equivalent) and diffs against the declared minimum scope list — any tenant found with broader-than-declared scope (e.g., from an old app version, or manual admin over-grant) is flagged for remediation, not silently tolerated.
- Any proposed *addition* to the minimum scope list requires explicit architecture-team review and an ADR amendment (or superseding ADR) — scope expansion is never a routine code change.

## Consequences

### Positive
- Centralized scope-list constants prevent silent scope creep from ad hoc call-site additions, directly operationalizing the "avoid broad scopes" guidance from §1/§2.
- Explicit CASA-assessment and Entra-application-access-policy tracking as project milestones (rather than assumed details) surfaces real launch-timeline risk early, addressing the report's own caution (§1, §2) that these have non-trivial lead time / are subject to change.
- Quarterly scope-drift audit catches configuration entropy over the system's life, not just at initial launch — least privilege is treated as an ongoing property to verify, not a one-time design decision.

### Negative
- Narrowing `MailboxSettings.ReadWrite` via application access policy per-tenant adds onboarding operational steps (each new tenant's mailboxes must be explicitly added to the policy) versus a simpler tenant-wide grant — a deliberate cost accepted for security posture.
- Maintaining two separate secrets-manager relationships (GCP + Azure) if not consolidated adds operational surface area; consolidation onto one vault for both is deferred as an implementation-time choice rather than decided here, which is itself a follow-up decision this ADR leaves open.
- CASA review lead time is an external dependency outside engineering's control and can slip the roadmap; must be tracked as a project risk, not just a checklist item.

### Neutral
- The exact current-state mechanics of Entra application access policies are explicitly flagged (§2, §8, and repeated here) as needing re-verification at implementation time rather than treated as settled by this ADR — this is a deliberate acknowledgment of the source material's own confidence caveat, not an oversight.

## Links
- Related: ADR-004 (OAuth token storage/vault schema — this ADR's governance wraps around that storage design), ADR-003 (multi-tenant isolation — per-tenant scope/policy boundaries), ADR-014 (compliance — token revocation on tenant deletion), ADR-016 (deployment topology — secrets manager as regional infrastructure dependency), ADR-018 (tenant admin API — separate credential class from mailbox OAuth tokens)

## Verification

- Automated test: attempt to request an OAuth scope outside the centralized allowlist constant anywhere in the codebase; a static-analysis/lint rule fails the build if a raw scope string is used instead of the shared constant.
- Quarterly audit job output reviewed by security team; any scope-drift finding tracked to remediation with an SLA (e.g., 14 days).
- Penetration-test/security-review checklist item: confirm no refresh token or access token appears in application logs, error messages, or API responses under any code path (automated log-scanning plus manual review).
- CASA assessment milestone tracked in the project roadmap with a named owner and target date, reviewed at each roadmap check-in until closed.
