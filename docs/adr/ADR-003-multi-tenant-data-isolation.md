# ADR-003: Multi-Tenant Data Isolation Strategy

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: multi-tenancy, security, data-isolation, postgres

## Context

The research report positions this system as a commercially viable product serving many customers' mailboxes (§7 tech stack recommends Postgres as the state store; §8 treats OAuth tokens and mail content as requiring strict least-privilege and DPA-scoped handling). The report does not itself specify a multi-tenancy model — this is a gap the report doesn't address directly — but its data model implications are clear: per-mailbox state (message-identity table, VIP list, sender reputation, watch/subscription renewal schedule — §7 Stage 6) must never leak across accounts, and per-mailbox rate limits are inherently account-scoped (§3 point 5: "Gmail's per-user 250 qu/s and Graph's ~4 req/s effective concurrency per mailbox are mailbox-scoped — horizontal scale-out (many users) is naturally parallel-safe"). This system will serve both individual users and, per §2's discussion of `MailboxSettings.ReadWrite` application permissions and admin-scoped access policies, enterprise tenants with many mailboxes under one Microsoft 365/Google Workspace admin.

Given mail content is among the most sensitive data categories a SaaS product can hold, isolation must be enforced at multiple layers, not solely in application code, to survive a single-layer bug.

## Decision

Adopt a **tenant → account → mailbox** hierarchy, with `tenant_id` as the top-level isolation key, enforced at three layers:

**1. Data model**: Every table that stores per-mailbox or per-tenant data carries a `tenant_id` column (UUID), including denormalized foreign keys, e.g.:

```sql
CREATE TABLE accounts (
  account_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenants(tenant_id),
  provider TEXT NOT NULL CHECK (provider IN ('gmail','graph')),
  provider_account_identifier TEXT NOT NULL, -- e.g. mailbox UPN or Gmail address
  UNIQUE (tenant_id, provider, provider_account_identifier)
);

CREATE TABLE messages (
  internal_message_id TEXT PRIMARY KEY,   -- ADR-007 identity hash
  tenant_id UUID NOT NULL,
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  -- ... classification state, FKs
  FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id)
);
```

**2. Row-Level Security (RLS)**: Postgres RLS policies enforce `tenant_id = current_setting('app.current_tenant_id')::uuid` on every tenant-scoped table. The application sets `app.current_tenant_id` via `SET LOCAL` at the start of every transaction, derived from the authenticated request/worker context — never from a client-supplied header alone. RLS is defense-in-depth: application-layer queries are still expected to filter by `tenant_id` explicitly, but a bug that omits the filter fails closed at the database layer instead of leaking rows.

**3. Worker/queue isolation**: Per §3 point 5 and ADR-006, work queues are partitioned per-mailbox (`account_id`), which naturally also partitions per-tenant — no cross-tenant queue ever exists, so a queue-draining bug cannot cross tenant boundaries. Batch API calls to the LLM adjudication layer (ADR-009) must not mix messages from different tenants in a way that could leak content across a shared prompt/response if a provider-side bug occurred — each Batch API submission is scoped to a single tenant's message set.

**4. Encryption boundary**: OAuth tokens and message content-at-rest are encrypted with tenant-scoped data encryption keys (DEKs), themselves wrapped by a tenant-independent KEK in the secrets manager (see ADR-004, ADR-020 batch 012-022). This ensures a full-database dump does not expose plaintext across tenants even if the key-management layer is later compromised at the KEK level requires separate compromise of per-tenant DEKs is unnecessary; the primary goal is that a leaked DB backup alone is not directly readable.

**5. Tenant tiers**: Support two tenant shapes explicitly, since Graph's admin-application-permission model (§2, §8) differs materially from Gmail's per-user OAuth model:
   - **Individual tenant**: `tenant_id` = 1 account, user-delegated OAuth only.
   - **Enterprise tenant**: `tenant_id` = many accounts under one Google Workspace domain or Microsoft Entra tenant, may use application permissions (Graph `MailboxSettings.ReadWrite` scoped via application access policy, per §2/§8) — the tenant admin's consent record is stored distinctly from individual mailbox owner consent, since revocation semantics differ (admin revocation should cascade to all accounts under that tenant; individual OAuth revocation affects only that one account).

## Consequences

### Positive
- Defense-in-depth (schema + RLS + queue partitioning + encryption) means no single layer's bug is sufficient to leak cross-tenant data.
- Enterprise admin-consent and individual user-consent are modeled distinctly, matching the real difference in Graph's delegated vs application permission model (§2).
- Per-mailbox queue partitioning (needed anyway for rate-limit correctness, ADR-006) is reused for tenant isolation at no extra cost.

### Negative
- RLS adds a `SET LOCAL` requirement to every transaction and a mandatory connection-pooling discipline (session-level settings must not leak across pooled connections — requires `SET LOCAL` scoped to transaction, not `SET`, and pool configuration that resets session state between checkouts).
- Two tenant shapes (individual vs enterprise) add branching in onboarding/consent flows and in revocation-cascade logic.

### Neutral
- This ADR intentionally does not decide the specific cloud KMS provider (GCP KMS vs Azure Key Vault) — see ADR-004 and ADR-020 (batch 012-022) for that decision, since it depends on where the service is deployed (ADR-016, batch 012-022).

## Links
- Related: ADR-004 (token vault uses tenant-scoped DEKs defined here), ADR-006 (per-mailbox queue model reused for tenant isolation), ADR-007 (internal message identity includes tenant scoping), ADR-018 (tenant/admin configuration API, batch 012-022), ADR-020 (secrets management/KMS detail, batch 012-022)

## Verification

- Automated test suite that provisions two tenants with overlapping data shapes and asserts every read path returns zero cross-tenant rows even when RLS is deliberately bypassed at the app layer (regression test for "forgot the WHERE clause").
- Static analysis/lint rule requiring every new migration touching a tenant-scoped table to include a `tenant_id` column and a corresponding RLS policy, enforced in CI.
- Periodic (quarterly) access-review audit: query for any table missing an RLS policy; alert if found.
