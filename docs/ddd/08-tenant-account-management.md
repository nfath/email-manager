# Bounded Context 8: Tenant & Account Management

## Purpose

Owns multi-tenant onboarding, mailbox connection lifecycle (connect/pause/disconnect), and plan-tier entitlements — including the LLM adjudication call budget per tier and the number of connected mailboxes allowed. This context is the **system-of-record for "who owns which mailboxes and what they're allowed to consume"**; it does not store OAuth credentials itself (that is [[01-identity-provider-access]]'s responsibility) but references credential records via a stable identifier and gates mailbox activation on their validity.

Source layout convention:

```
src/tenant-account-management/
  domain/
    entities/         # Tenant, MailboxConnection, UsageCounter
    value-objects/     # PlanTier, Entitlements, BillingStatus
    events/            # Domain events
    services/          # MailboxConnectionProvisioningService, EntitlementEnforcementService
    repositories/       # Repository interfaces
  application/         # ProvisionTenant use case, ConnectMailbox use case
  infrastructure/       # IdentityProviderAccessAdapter, Postgres repos
  index.ts              # Public API of the context
```

## 1. Ubiquitous Language

| Term | Definition |
|---|---|
| **Tenant** | The billing/organizational entity (an individual or an organization) that owns one or more Mailbox Connections and is billed per Plan Tier. |
| **Mailbox Connection** | A lifecycle-managed link between a Tenant and one connected mailbox on Gmail or Outlook. Independently created, paused, resumed, and disconnected. |
| **Plan Tier** | A named subscription level (`Free`, `Pro`, `Business`, `Enterprise`) that determines a Tenant's Entitlements. |
| **Entitlements** | The concrete quota/capability set derived from Plan Tier: max mailbox connections, LLM adjudication call budget per billing period, and which of the 11 categories are available (e.g., phishing detection may be Business+ only). |
| **Usage Counter** | A per-Tenant, per-billing-period running total of consumption against an entitlement (e.g., LLM adjudication calls used this month). |
| **Billing Status** | `Active`, `PastDue`, or `Suspended` — gates whether new Mailbox Connections may be created or resumed. |
| **Credential Reference** | An opaque pointer (`credentialRefId`) to a credential record owned by [[01-identity-provider-access]]; this context never stores or reads the token itself. |

## 2. Aggregates

### 2.1 Aggregate Root: `Tenant`

**Fields**: `tenantId`, `displayName`, `planTier: PlanTier`, `entitlements: Entitlements` (denormalized snapshot of the tier's limits, allowing per-tenant overrides for negotiated enterprise deals), `billingStatus`, `createdAt`.

**Invariants**:
1. **No provisioning while suspended**: a `Tenant` with `billingStatus = Suspended` cannot have new `MailboxConnection`s requested against it — enforced by `MailboxConnectionProvisioningService` checking `billingStatus` before creation, since the check spans two aggregates.
2. **Downgrade guard**: `ChangePlanTier` to a tier with a lower `maxMailboxConnections` than the Tenant's current active connection count is rejected — the tenant must disconnect down to the new limit first; this context never auto-disconnects mailboxes as a side effect of a plan change.
3. **Entitlement override bounds**: a per-tenant `entitlements` override (e.g., a negotiated enterprise deal) may only *raise* limits relative to the tier default, never lower them below the tier's published floor — prevents an override from accidentally under-provisioning a paying tenant.

### 2.2 Aggregate Root: `MailboxConnection`

**Fields**: `mailboxConnectionId`, `tenantId`, `provider` (`gmail | outlook`), `providerAccountEmail`, `status` (`PendingAuth | Active | Paused | Disconnected | Error`), `credentialRefId`, `connectedAt`, `pausedAt`, `disconnectedAt`, `lastError`.

**Invariants**:
1. **No activation without valid credential**: `status` cannot transition to `Active` until an ACL check against [[01-identity-provider-access]] confirms `credentialRefId` refers to a non-expired, correctly-scoped credential (`gmail.modify`+`gmail.labels` or `Mail.ReadWrite`+`MailboxSettings.ReadWrite` per the research report §8) — this context does not duplicate token validity state, it asks each time activation is attempted.
2. **Uniqueness per tenant**: `(tenantId, provider, providerAccountEmail)` must be unique — the same mailbox cannot be connected twice under one Tenant (prevents duplicate ingestion/sync work).
3. **Disconnection is terminal**: no transition is defined out of `Disconnected` — reconnecting the same mailbox requires creating a brand-new `MailboxConnection` (and a fresh credential grant), never resurrecting the old aggregate. This keeps historical audit trails (see [[09-audit-compliance]]) unambiguous about which connection instance produced which classification.
4. **Pause halts ingestion, not history**: transitioning to `Paused` must be observable by [[02-mail-ingestion]] (via the `MailboxConnectionPaused` event) so its watchers stop renewing push subscriptions — this aggregate does not directly command Mail Ingestion, it only publishes state.
5. **Connection count vs. entitlement**: `RequestMailboxConnection` is rejected if activating it would exceed the owning Tenant's `entitlements.maxMailboxConnections` — checked at request time by `MailboxConnectionProvisioningService`, not deferred to activation.

### 2.3 Aggregate Root: `UsageCounter`

One instance per `(tenantId, billingPeriod, entitlementKind)` — e.g., `(tenant-123, 2026-09, llmAdjudicationCalls)`.

**Fields**: `tenantId`, `billingPeriod`, `entitlementKind`, `used`, `limit` (copied from Tenant's entitlements at period start), `resetAt`.

**Invariants**:
1. **Hard cap enforcement**: `RecordUsageConsumption` that would push `used` beyond `limit` is rejected outright (not merely warned) — [[04-llm-adjudication]] must check-and-reserve budget here *before* spending, so this counter is the actual gate on LLM spend, not a passive tracker.
2. **Period isolation**: a `UsageCounter` never carries `used` forward across `resetAt` boundaries — a new counter instance is created for the new period; unused budget does not roll over unless the Plan Tier explicitly defines rollover (not modeled in MVP/V2).

## 3. Value Objects

- `PlanTier` — enum `Free | Pro | Business | Enterprise`
- `Entitlements { maxMailboxConnections: number, llmAdjudicationCallBudgetPerMonth: number, allowedCategories: InternalCategory[] }`
- `BillingStatus` — enum `Active | PastDue | Suspended`

## 4. Domain Events

| Event | Payload (key fields) |
|---|---|
| `TenantProvisioned` | `tenantId, displayName, planTier, entitlements, createdAt` |
| `TenantPlanChanged` | `tenantId, oldPlanTier, newPlanTier, newEntitlements, changedAt` |
| `TenantSuspended` | `tenantId, reason, suspendedAt` |
| `MailboxConnectionRequested` | `mailboxConnectionId, tenantId, provider, providerAccountEmail, requestedAt` |
| `MailboxConnectionActivated` | `mailboxConnectionId, tenantId, credentialRefId, activatedAt` |
| `MailboxConnectionPaused` | `mailboxConnectionId, tenantId, pausedAt` |
| `MailboxConnectionResumed` | `mailboxConnectionId, tenantId, resumedAt` |
| `MailboxConnectionDisconnected` | `mailboxConnectionId, tenantId, disconnectedAt, reason` |
| `UsageBudgetExceeded` | `tenantId, entitlementKind, billingPeriod, attemptedUsage, limit` |
| `UsageCounterReset` | `tenantId, entitlementKind, newBillingPeriod, resetAt` |

## 5. Commands

- `ProvisionTenant { displayName, planTier }`
- `ChangePlanTier { tenantId, newPlanTier }`
- `SuspendTenant { tenantId, reason }`
- `RequestMailboxConnection { tenantId, provider, providerAccountEmail }`
- `ActivateMailboxConnection { mailboxConnectionId, credentialRefId }`
- `PauseMailboxConnection { mailboxConnectionId }`
- `ResumeMailboxConnection { mailboxConnectionId }`
- `DisconnectMailboxConnection { mailboxConnectionId, reason }`
- `RecordUsageConsumption { tenantId, entitlementKind, amount }`
- `ResetUsageCounter { tenantId, entitlementKind }`

## 6. Repository Interfaces

```typescript
interface TenantRepository {
  findById(tenantId: TenantId): Promise<Tenant | null>;
  findByBillingStatus(status: BillingStatus, limit: number, cursor?: string): Promise<Page<Tenant>>;
  save(tenant: Tenant): Promise<void>;
}

interface MailboxConnectionRepository {
  findById(mailboxConnectionId: MailboxConnectionId): Promise<MailboxConnection | null>;
  findByTenant(tenantId: TenantId): Promise<MailboxConnection[]>;
  findByProviderAccount(tenantId: TenantId, provider: Provider, providerAccountEmail: string): Promise<MailboxConnection | null>;
  countActiveByTenant(tenantId: TenantId): Promise<number>;
  findByStatus(status: ConnectionStatus, limit: number, cursor?: string): Promise<Page<MailboxConnection>>;
  save(connection: MailboxConnection): Promise<void>;
}

interface UsageCounterRepository {
  findCurrent(tenantId: TenantId, entitlementKind: string): Promise<UsageCounter | null>;
  incrementIfWithinLimit(tenantId: TenantId, entitlementKind: string, amount: number): Promise<{ accepted: boolean; counter: UsageCounter }>;
  save(counter: UsageCounter): Promise<void>;
}
```

## 7. Domain Services

- **`MailboxConnectionProvisioningService`**: orchestrates `RequestMailboxConnection` and `ActivateMailboxConnection` across the `Tenant` (entitlement/billing-status check) and `MailboxConnection` aggregates, and calls out to the `IdentityProviderAccessAdapter` ACL for credential validity — the only place these three concerns meet.
- **`EntitlementEnforcementService`**: the shared check used by other contexts (via a query API, not direct repository access) to answer "is tenant X within budget for Y" — backs [[04-llm-adjudication]]'s pre-spend check against `UsageCounter`.
- **`PlanChangeValidationService`**: validates a `ChangePlanTier` request against current active connection count before allowing the downgrade invariant to pass.

## 8. Anti-Corruption Layer / Integration Adapters

```typescript
// Port this context depends on — implemented by an adapter that calls into
// Identity & Provider Access's own API/event stream. This context never reads
// Identity & Provider Access's internal credential storage directly.
interface IdentityProviderAccessAdapter {
  checkCredentialValid(credentialRefId: CredentialRefId): Promise<{ valid: boolean; scopes: string[]; expiresAt: Date | null }>;
  initiateOAuthGrant(tenantId: TenantId, provider: Provider): Promise<{ authorizationUrl: string; pendingCredentialRefId: CredentialRefId }>;
}
```

- **`IdentityProviderAccessAdapter`** isolates this context from Identity & Provider Access's internal OAuth/token-vault model — `ActivateMailboxConnection` calls `checkCredentialValid` and only proceeds on `valid: true`, never inspecting the token itself.

## 9. Relationships to Other Bounded Contexts

| Context | Relationship | Notes |
|---|---|---|
| **1. Identity & Provider Access** | Customer–Supplier (this context is Customer) | Depends on credential validity confirmation via the ACL above before activating any `MailboxConnection`; does not dictate how credentials are stored or refreshed. |
| **2. Mail Ingestion** | Open Host Service (this context is the Supplier/definer) | Publishes `MailboxConnectionActivated`/`Paused`/`Disconnected` as the Published Language that drives ingestion watcher lifecycle (start/stop push subscriptions); Mail Ingestion is a Conformist consumer. |
| **3. Classification Engine** | Open Host Service (this context is Supplier) | Exposes `allowedCategories` from `Entitlements` so Classification Engine knows which categories are in-scope per tenant plan. |
| **4. LLM Adjudication** | Customer–Supplier (this context is Supplier) | LLM Adjudication is a Conformist consumer of the `EntitlementEnforcementService`/`UsageCounter` budget gate — it must check-and-reserve before every call, with no ability to negotiate the cap itself. |
| **6. Action & Provider Sync** | Conformist consumer of this context | Reads `MailboxConnection.status` to decide sync eligibility (a `Paused` mailbox halts sync processing). |
| **7. Feedback & Learning** | Conformist consumer of this context | Scopes `RuleWeightProfile` forking by `tenantId` as defined here, without altering tenant lifecycle. |
| **9. Audit & Compliance** | Partnership | Tenant/mailbox lifecycle events (especially `MailboxConnectionDisconnected` and DSR-relevant tenant boundaries) are jointly relevant — Audit & Compliance's DSR scope (§9 of this doc set) is defined in terms of a Tenant's current and historical `MailboxConnection`s. |
| **10. Notification & Reporting** | Supplier (this context is Supplier) | Publishes plan/usage events consumed for spend and utilization dashboards. |
| **5. Relationship Graph** | none direct | No dependency — VIP/reciprocity scoring operates within a mailbox's data, scoped by `mailboxId` already established here, but this context has no further involvement. |
