# ADR-004: OAuth Token Storage & Credential Vault Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: security, oauth, secrets-management, encryption

## Context

Per §1 and §2 of the research report, both providers require OAuth 2.0 credentials with specific scope minimization requirements: Gmail's `gmail.modify` scope is a **restricted scope requiring Google CASA security assessment** for production use (§1), and Graph's `offline_access` scope is mandatory for refresh tokens since access tokens expire after ~1 hour without it (§2). Per §8 ("Token Handling Best Practices"), the report is explicit: "Encrypt refresh tokens at rest: Use secrets manager or app-level encryption over a DB column; never in client-side/browser storage," "Rotate on refresh: Treat as short-lived credentials," and cites Google's own OAuth token storage best-practices doc. The report also flags that minimum viable scopes must avoid the broad `mail.google.com` scope and Graph's tenant-wide-by-default `MailboxSettings.ReadWrite` application permission should be narrowed via application access policies (§2, §8) — meaning the vault must track which scopes were actually granted per account, not assume a fixed set.

## Decision

**Storage model**: Refresh tokens and access tokens are never stored in application code, browser/client storage, or logs. They live in a dedicated `credentials` table, tenant-scoped (ADR-003), with envelope encryption:

```sql
CREATE TABLE oauth_credentials (
  credential_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  provider TEXT NOT NULL CHECK (provider IN ('gmail','graph')),
  granted_scopes TEXT[] NOT NULL,          -- actual granted scopes, not assumed
  encrypted_refresh_token BYTEA NOT NULL,  -- envelope-encrypted, see below
  encrypted_access_token BYTEA,            -- optional cache; short TTL
  access_token_expires_at TIMESTAMPTZ,
  dek_key_id TEXT NOT NULL,                -- reference to wrapped DEK, not the DEK itself
  rotated_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,                  -- soft-revocation marker
  UNIQUE (account_id, provider)
);
```

**Envelope encryption**: Each tenant has a Data Encryption Key (DEK) generated at tenant provisioning time. The DEK itself is wrapped ("encrypted") by a tenant-independent Key Encryption Key (KEK) held in the cloud provider's managed KMS (GCP Cloud KMS or Azure Key Vault, per deployment target — ADR-016 batch 012-022). Application servers never hold the KEK directly; they call the KMS's `decrypt(wrapped_dek)` operation to obtain a usable DEK in memory per-request, then encrypt/decrypt token columns with that DEK using AES-256-GCM. This two-tier scheme means a DB backup leak alone is not sufficient to decrypt tokens (KMS access is also required), satisfying §8's "encrypt at rest" directive with an auditable, revocable key hierarchy.

**Rotation policy**:
- Access tokens: never persisted longer than their provider-stated TTL (~1hr for both providers); refreshed proactively at 80% of TTL elapsed, not on-demand-only, to avoid request-path latency spikes.
- Refresh tokens: rotated whenever the provider issues a new one on refresh (both Google and Microsoft may rotate refresh tokens silently); the vault always persists the *latest* refresh token and invalidates the prior encrypted value in the same transaction — no window where both old and new are simultaneously valid-looking in storage.
- Full re-consent flow triggered if a refresh attempt fails with an explicit revocation error (`invalid_grant` for Google, `AADSTS700082`-class errors for Microsoft) — the vault marks `revoked_at` and surfaces a re-auth-required state to the tenant/admin API (ADR-018, batch 012-022), rather than retrying indefinitely.

**Scope tracking and least privilege**: `granted_scopes` is populated from the actual OAuth consent response, not assumed from the requested scope list — Google Workspace admins or individual users may grant a subset. The pipeline (ADR-001) checks `granted_scopes` before attempting an action that requires a scope (e.g., skip `contacts.other.readonly`-dependent VIP signals, ADR-011, if not granted) and surfaces a degraded-capability warning rather than failing the whole account.

**Minimum scope sets enforced at OAuth consent-request time**:
- Gmail: `gmail.modify`, `gmail.labels`, `contacts.readonly`, `contacts.other.readonly` — explicitly never request `mail.google.com`.
- Graph: `Mail.ReadWrite`, `MailboxSettings.ReadWrite`, `Contacts.Read`, `offline_access` — for enterprise tenants, the admin is directed to scope `MailboxSettings.ReadWrite` via an application access policy to onboarded mailboxes only (§2, §8), not tenant-wide; this is a deployment runbook item cross-referenced in ADR-018 (batch 012-022), since Graph admin-console mechanics are noted in §2/§9 (gap #4) as needing re-verification against live docs at implementation time.

**Access auditing**: Every `decrypt(wrapped_dek)` KMS call and every token read from `oauth_credentials` is logged (credential_id, requesting service, timestamp) to a write-once audit log, feeding ADR-015 (observability, batch 012-022) and supporting incident forensics without needing to touch plaintext tokens.

## Consequences

### Positive
- Two-tier envelope encryption means neither a DB-only leak nor a KMS-only compromise alone is sufficient to expose plaintext tokens.
- Explicit `granted_scopes` tracking lets the system degrade gracefully per-account rather than assuming uniform scope grants across all tenants.
- Rotation-on-refresh with same-transaction invalidation eliminates a class of "stale token replay" bugs.

### Negative
- Every token decrypt requires a KMS round-trip (network latency); requires connection pooling/caching of unwrapped DEKs in memory with short TTL to avoid a KMS call per single message classification action. This introduces an in-memory plaintext DEK caching decision that must be scoped carefully (short TTL, process-memory only, never written to disk/swap-vulnerable storage).
- Scope-degradation logic adds conditional branches throughout the signal-extraction layer (ADR-011 VIP scoring, People API dependency) that must be tested for every scope-subset combination.

### Neutral
- The exact KMS provider (GCP KMS vs Azure Key Vault) is deferred to ADR-020 (batch 012-022) / ADR-016 (batch 012-022) deployment-topology decisions; this ADR only fixes the envelope-encryption pattern, not the vendor.

## Links
- Related: ADR-003 (tenant-scoped DEK boundary), ADR-005 (ingestion sync depends on valid access tokens from this vault), ADR-018 (admin API surfaces re-auth-required state, batch 012-022), ADR-020 (secrets management/KMS provider detail, batch 012-022)

## Verification

- Penetration-test scenario: given a raw DB backup with no KMS access, confirm no token plaintext is recoverable.
- Unit tests for rotation-on-refresh transactional atomicity (simulate refresh mid-flight crash, assert no dual-valid-token state).
- Scope-degradation integration tests: revoke `contacts.other.readonly` on a test account and assert VIP scoring (ADR-011) falls back to manual-list-only mode without erroring the whole pipeline.
- Quarterly credential-audit-log review confirming no out-of-band (non-KMS-mediated) decrypt paths exist in the codebase (grep-based CI check for direct AES key material outside the KMS client wrapper).
