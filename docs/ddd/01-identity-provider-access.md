# Bounded Context 01: Identity & Provider Access

## Purpose

Owns the entire lifecycle of connecting a user's mailbox (Gmail or Microsoft 365/Outlook) to the system: OAuth consent, credential storage, token refresh, scope management, and the mapping between an internal tenant/user identity and one or more provider mailbox connections. Every other context that must call Gmail or Microsoft Graph obtains a live access token through this context — no other context talks to an OAuth endpoint directly.

---

## 1. Ubiquitous Language Glossary

| Term | Definition |
|---|---|
| **Provider** | An external mail platform this system integrates with: `GOOGLE` or `MICROSOFT`. |
| **MailboxConnection** | The aggregate root representing one authorized link between an internal user and one provider mailbox (one Gmail address or one M365 mailbox). |
| **CredentialVault Entry** | The encrypted-at-rest storage record holding a refresh token (and current access token) for a `MailboxConnection`. Never leaves this context in decrypted form except as a short-lived in-memory token handed to an authorized caller. |
| **ScopeGrant** | A value object recording which OAuth scopes were actually granted by the user/admin at consent time (may be a subset of requested scopes). |
| **TokenLease** | A short-lived, in-memory-only value object representing a valid access token issued to a calling context for a bounded duration (never persisted). |
| **ConsentSession** | An entity tracking an in-progress OAuth authorization-code flow (state nonce, PKCE verifier, redirect target) between redirect-out and callback-in. |
| **TenantMailboxLink** | A value object binding a `MailboxConnection` to a tenant (for Microsoft, the Entra tenant ID; for Google, the Workspace customer ID or "consumer" marker) — required because Microsoft's `MailboxSettings.ReadWrite` application permission is tenant-scoped. |
| **Application Access Policy Reference** | A value object recording the Entra application access policy ID that scopes this app's tenant-wide application permissions down to specific onboarded mailboxes (Microsoft-only concept, §8 of research). |
| **CASA Review Status** | A value object tracking whether the OAuth client's Google `gmail.modify` restricted-scope grant has passed Google's CASA security assessment (required for production use of that scope). |
| **Revocation** | The event/state of a user or admin withdrawing consent, either detected via a failed refresh (`invalid_grant`) or an explicit disconnect action. |

---

## 2. Aggregate Root: `MailboxConnection`

### Identity
- `connectionId: ConnectionId` (UUID, internally generated, immutable)

### Fields (Entities & Value Objects)
- `tenantId: TenantId` — internal tenant this connection belongs to (see Context 08: Tenant & Account Management, which owns tenant lifecycle; this context only references the ID)
- `provider: Provider` (`GOOGLE` | `MICROSOFT`)
- `providerAccountId: string` — Google's stable `sub`/account ID, or Microsoft's `oid` (object ID)
- `mailboxAddress: EmailAddress` (value object: validated RFC 5322 address string)
- `tenantMailboxLink: TenantMailboxLink | null` (null for Google consumer accounts; populated for Google Workspace and all Microsoft accounts)
- `scopeGrant: ScopeGrant` — value object: `{ requested: Scope[], granted: Scope[], grantedAt: DateTime }`
- `credentialRef: CredentialVaultEntryId` — pointer to the encrypted credential record (the actual secret never lives on the aggregate itself, to keep the aggregate safely loggable/serializable)
- `status: ConnectionStatus` (`PENDING_CONSENT` | `ACTIVE` | `TOKEN_EXPIRED` | `REVOKED` | `SUSPENDED_BY_POLICY`)
- `casaReviewStatus: CasaReviewStatus | null` (Google-only; `NOT_REQUIRED` | `PENDING` | `APPROVED` | `REJECTED`)
- `applicationAccessPolicyRef: string | null` (Microsoft-only; the Entra application access policy ID scoping this connection)
- `lastSuccessfulRefreshAt: DateTime | null`
- `consecutiveRefreshFailures: number`
- `createdAt: DateTime`, `updatedAt: DateTime`

### Entities within the aggregate
- **`CredentialVaultEntry`** (child entity, referenced by ID from the aggregate root but lifecycle-managed together):
  - `credentialId: CredentialVaultEntryId`
  - `encryptedRefreshToken: EncryptedBlob` (encrypted via KMS/Secret Manager envelope encryption, never decrypted outside a scoped token-issuance operation)
  - `encryptedAccessToken: EncryptedBlob | null`
  - `accessTokenExpiresAt: DateTime | null`
  - `encryptionKeyVersion: string` (supports key rotation without re-encrypting every row atomically)

### Value Objects
- `EmailAddress` — validated, normalized (lowercase domain) string wrapper
- `Scope` — enum-like value object, one of the minimum-viable scopes identified in research: `GMAIL_MODIFY`, `GMAIL_LABELS`, `GOOGLE_CONTACTS_READONLY`, `GOOGLE_CONTACTS_OTHER_READONLY`, `GRAPH_MAIL_READWRITE`, `GRAPH_MAILBOX_SETTINGS_READWRITE`, `GRAPH_CONTACTS_READ`, `GRAPH_OFFLINE_ACCESS`
- `ScopeGrant { requested: Scope[], granted: Scope[], grantedAt: DateTime }`
- `TenantMailboxLink { tenantExternalId: string, tenantType: 'GOOGLE_WORKSPACE' | 'MICROSOFT_ENTRA' | 'GOOGLE_CONSUMER' }`
- `TokenLease { accessToken: string, expiresAt: DateTime, scopes: Scope[] }` — **never persisted**, constructed only in-memory and returned to a calling context through the domain service below

### Invariants (enforced within the aggregate's consistency boundary)

1. **No token issuance without `ACTIVE` status.** A `TokenLease` can only be minted (via `IssueTokenLease`) when `status == ACTIVE`. Attempting on `REVOKED`, `SUSPENDED_BY_POLICY`, or `TOKEN_EXPIRED` (without a queued refresh) must fail.
2. **Minimum viable scope enforcement.** A `MailboxConnection` cannot transition from `PENDING_CONSENT` to `ACTIVE` unless `scopeGrant.granted` is a superset of the provider's minimum required set (Google: `GMAIL_MODIFY` + `GMAIL_LABELS`; Microsoft: `GRAPH_MAIL_READWRITE` + `GRAPH_OFFLINE_ACCESS`). Contacts scopes are desired but not blocking — their absence instead disables Context 05 (Relationship Graph) contact cross-referencing for this mailbox, recorded as a capability flag, not a hard failure.
3. **`GRAPH_OFFLINE_ACCESS` is mandatory for Microsoft connections.** Per research (§2), without `offline_access` the grant dies after ~1hr with no refresh path. A Microsoft `MailboxConnection` missing this scope must never reach `ACTIVE`; it is rejected at consent time with a re-consent prompt.
4. **CASA gate for Google `gmail.modify`.** A Google `MailboxConnection` requesting `GMAIL_MODIFY` cannot be marked `ACTIVE` for production traffic while `casaReviewStatus != APPROVED` unless the connection is flagged as a sandbox/test account (a separate `isSandboxAccount: boolean` field, default false, settable only by an internal admin operation, not user-facing).
5. **Refresh-failure circuit breaker.** After `consecutiveRefreshFailures >= 5` (configurable), status must transition to `REVOKED` and further automatic refresh attempts must stop, to avoid hammering the provider's token endpoint and to surface the disconnect to the user. A single successful refresh resets the counter to 0.
6. **One active `MailboxConnection` per (tenant, provider, providerAccountId) tuple.** Re-authorizing the same provider account must update the existing aggregate (rotate its credential, reset failure counters) rather than create a duplicate — enforced via a uniqueness constraint at the repository layer, checked before insert.
7. **Encrypted-at-rest, never-in-cleartext-outside-vault invariant.** The `encryptedRefreshToken` and `encryptedAccessToken` fields must never be assigned a plaintext value; the aggregate's only mutation path for these fields is through an application-layer command that calls the encryption port. This is enforced by making the raw setter package-private/internal to the aggregate module — no public API surface accepts plaintext tokens except the consent-callback command handler, which immediately encrypts before constructing the entity.
8. **Application access policy required before tenant-wide Microsoft app permission use.** A Microsoft `TenantMailboxLink` of type `MICROSOFT_ENTRA` cannot be marked usable for `GRAPH_MAILBOX_SETTINGS_READWRITE` application-permission calls unless `applicationAccessPolicyRef` is set — prevents accidentally acting against every mailbox in a tenant when only specific mailboxes were onboarded (research §8).

---

## 3. Domain Events (PascalCase, past-tense)

| Event | Payload |
|---|---|
| `ConsentSessionStarted` | `{ consentSessionId, tenantId, provider, requestedScopes: Scope[], redirectTarget, pkceChallenge, createdAt }` |
| `MailboxConnectionAuthorized` | `{ connectionId, tenantId, provider, providerAccountId, mailboxAddress, scopeGrant, tenantMailboxLink, occurredAt }` |
| `MailboxConnectionActivated` | `{ connectionId, activatedAt }` |
| `AccessTokenRefreshed` | `{ connectionId, accessTokenExpiresAt, refreshedAt }` (never carries the token value itself — consumed internally, not broadcast with secret material) |
| `TokenRefreshFailed` | `{ connectionId, providerErrorCode, consecutiveFailures, failedAt }` |
| `MailboxConnectionRevoked` | `{ connectionId, tenantId, reason: 'USER_INITIATED' \| 'PROVIDER_INVALID_GRANT' \| 'ADMIN_INITIATED' \| 'REFRESH_CIRCUIT_BREAKER', revokedAt }` |
| `MailboxConnectionSuspendedByPolicy` | `{ connectionId, policyReason: string, suspendedAt }` |
| `ScopeGrantChanged` | `{ connectionId, previousGrant: Scope[], newGrant: Scope[], changedAt }` (fires on incremental consent / re-consent) |
| `CasaReviewStatusChanged` | `{ connectionId, previousStatus, newStatus, changedAt }` |
| `TokenLeaseIssued` | *(internal, non-persisted domain notification only — used for audit/rate-metrics, not for cross-context integration; carries no secret material)* `{ connectionId, requestingContext: string, scopes: Scope[], issuedAt }` |

---

## 4. Commands (PascalCase, imperative)

| Command | Description |
|---|---|
| `StartConsentSession` | Begins an OAuth authorization-code (+PKCE) flow for a given tenant/provider; creates a `ConsentSession`. |
| `CompleteConsentCallback` | Exchanges the authorization code for tokens, encrypts and stores them, creates or updates the `MailboxConnection`. |
| `ActivateMailboxConnection` | Transitions `PENDING_CONSENT` → `ACTIVE` after invariant checks (#2, #3, #4) pass. |
| `IssueTokenLease` | Requests a short-lived access token for an authorized calling context; triggers a refresh internally if the cached access token is expired or near-expiry. |
| `RefreshAccessToken` | Explicitly forces a refresh-token exchange (used by a scheduled renewal job, distinct from Gmail-watch/Graph-subscription renewal which lives in Context 02). |
| `RevokeMailboxConnection` | User- or admin-initiated disconnect; calls the provider's token-revocation endpoint and marks the aggregate `REVOKED`. |
| `RecordTokenRefreshFailure` | Invoked by the refresh operation on failure; increments the failure counter and may trigger the circuit breaker. |
| `UpdateScopeGrant` | Records a scope change after incremental/re-consent. |
| `SetCasaReviewStatus` | Admin operation updating the CASA review status for Google restricted-scope production readiness. |
| `AttachApplicationAccessPolicy` | Admin operation recording the Entra application access policy ID scoping this tenant's mailbox access. |

---

## 5. Repository Interface

```typescript
interface MailboxConnectionRepository {
  findById(connectionId: ConnectionId): Promise<MailboxConnection | null>;

  findByProviderAccount(
    tenantId: TenantId,
    provider: Provider,
    providerAccountId: string
  ): Promise<MailboxConnection | null>;

  findActiveByTenant(tenantId: TenantId): Promise<MailboxConnection[]>;

  findByMailboxAddress(
    tenantId: TenantId,
    mailboxAddress: EmailAddress
  ): Promise<MailboxConnection | null>;

  findDueForRefresh(before: DateTime, limit: number): Promise<MailboxConnection[]>;
  // access tokens expiring within a lookahead window; feeds the refresh scheduler

  findByCircuitBreakerCandidates(minFailures: number): Promise<MailboxConnection[]>;

  save(connection: MailboxConnection): Promise<void>;
  // upserts aggregate + credential vault entry transactionally

  delete(connectionId: ConnectionId): Promise<void>;
  // hard delete only on GDPR/data-subject erasure request (see Context 09: Audit & Compliance),
  // never as a routine disconnect path (routine disconnect = REVOKED status, retained for audit)
}

interface ConsentSessionRepository {
  findByState(stateNonce: string): Promise<ConsentSession | null>;
  save(session: ConsentSession): Promise<void>;
  deleteExpired(before: DateTime): Promise<number>;
}
```

---

## 6. Domain Services

### `TokenLeaseService`
Coordinates issuance of `TokenLease` value objects to calling contexts. Spans the `MailboxConnection` aggregate and the external OAuth token endpoint (via the ACL below), and encapsulates the "refresh-if-needed, then hand back a lease" logic so no other context ever sees a refresh token.

```typescript
interface TokenLeaseService {
  issueLease(connectionId: ConnectionId, requestingContext: string): Promise<TokenLease>;
  // throws MailboxConnectionNotActiveError if status != ACTIVE
}
```

### `ScopeSufficiencyService`
Spans `ScopeGrant` and each downstream context's declared scope requirements (Context 02 needs mail read/watch scopes; Context 05 needs contacts scopes). Used at activation time and whenever a context asks "can I do X against this mailbox."

```typescript
interface ScopeSufficiencyService {
  isSufficientFor(grant: ScopeGrant, requiredCapability: RequiredCapability): boolean;
  missingScopes(grant: ScopeGrant, requiredCapability: RequiredCapability): Scope[];
}
```

### `RefreshCircuitBreakerService`
Spans multiple `MailboxConnection` refresh-failure histories to decide, on each failure, whether to retry with backoff or trip to `REVOKED` (invariant #5).

```typescript
interface RefreshCircuitBreakerService {
  recordFailureAndEvaluate(connectionId: ConnectionId, errorCode: string): Promise<CircuitDecision>;
  // CircuitDecision = { action: 'RETRY_WITH_BACKOFF' | 'TRIP_REVOKED', backoffMs?: number }
}
```

---

## 7. Anti-Corruption Layer / Integration Adapters

Two provider-specific ACLs isolate the domain from Google's and Microsoft's OAuth/identity wire formats. Both implement a shared internal port so application services never branch on provider type outside this context.

```typescript
// Shared internal port — the ONLY interface the rest of the domain depends on
interface IdentityProviderPort {
  buildAuthorizationUrl(params: {
    tenantId: TenantId;
    scopes: Scope[];
    redirectUri: string;
    pkceChallenge: string;
    stateNonce: string;
  }): string;

  exchangeCodeForTokens(params: {
    authorizationCode: string;
    pkceVerifier: string;
    redirectUri: string;
  }): Promise<ProviderTokenResponse>;

  refreshTokens(refreshToken: string): Promise<ProviderTokenResponse>;

  revokeToken(token: string): Promise<void>;

  fetchAccountIdentity(accessToken: string): Promise<{
    providerAccountId: string;
    mailboxAddress: string;
    tenantExternalId: string | null; // Workspace customer ID / Entra tenant ID
  }>;
}

interface ProviderTokenResponse {
  accessToken: string;
  refreshToken: string | null; // Google/Microsoft may omit on refresh (token unchanged) — ACL normalizes to null, never invents a value
  expiresInSeconds: number;
  grantedScopes: string[]; // raw provider scope strings, normalized to internal Scope[] by the ACL
}
```

```typescript
// Google-specific adapter — isolates Google OAuth2 endpoint shapes
// (authorization_code exchange, https://oauth2.googleapis.com/token, tokeninfo scope introspection)
class GoogleIdentityAdapter implements IdentityProviderPort { /* ... */ }

// Microsoft-specific adapter — isolates MSAL/Entra v2.0 endpoint shapes
// (authorization_code + PKCE via MSAL, tenant-specific vs /common authority selection,
// id_token claims parsing for oid/tid)
class MicrosoftIdentityAdapter implements IdentityProviderPort { /* ... */ }
```

The ACL is also responsible for normalizing each provider's distinct scope-string vocabulary (e.g., Google's `https://www.googleapis.com/auth/gmail.modify` vs Microsoft's `Mail.ReadWrite`) into the internal `Scope` enum, so invariant checks never need provider-specific branching.

---

## 8. Relationships to Other Bounded Contexts

- **Mail Ingestion (Context 02) — Open Host Service / Published Language.** This context exposes `IssueTokenLease` as an Open Host Service; Mail Ingestion is a pure consumer of `TokenLease` and never touches OAuth mechanics directly. The published language is the `TokenLease` value object plus the `RequiredCapability` enum used by `ScopeSufficiencyService`.
- **Relationship Graph (Context 05) — Open Host Service / Published Language.** Same pattern as above; Context 05 requests leases scoped to `GOOGLE_CONTACTS_READONLY`/`GRAPH_CONTACTS_READ` capabilities and degrades gracefully (per invariant #2) if absent.
- **Action & Provider Sync (Context 06) — Open Host Service / Published Language.** Consumes leases scoped to label/category-write capabilities for `batchModify`/category-assignment calls.
- **Tenant & Account Management (Context 08) — Customer-Supplier, with Context 08 upstream.** Context 08 owns tenant creation/plan entitlements and is the *supplier* of `TenantId` and `TenantMailboxLink.tenantType`; this context is the *customer*, referencing but never mutating tenant lifecycle state. Mailbox-count entitlement limits are enforced by Context 08 before this context is allowed to create a new `MailboxConnection`.
- **Audit & Compliance (Context 09) — Customer-Supplier, with Context 09 downstream as consumer.** This context is the *supplier* of every credential-lifecycle event (`MailboxConnectionAuthorized`, `MailboxConnectionRevoked`, etc.) which Context 09 consumes as an append-only audit trail and for data-subject-request (right-to-erasure) fulfillment; Context 09 dictates retention requirements this context's `delete()` path must respect.
- **Classification Engine (Context 03) and LLM Adjudication (Context 04) — no direct relationship.** Neither ever calls this context; they operate on normalized messages already produced by Mail Ingestion and never need provider credentials.
- **Notification & Reporting (Context 10) — Conformist.** Consumes `MailboxConnectionRevoked`/`MailboxConnectionSuspendedByPolicy` events as-is (e.g., to alert a user "reconnect your mailbox") without negotiating a custom contract back to this context.
