# Bounded Context 9: Audit & Compliance

## Purpose

Maintains an immutable audit log of every classification decision and every applied provider action (who/what/when/why, and which layer decided), handles data-subject requests (DSR) for GDPR/CCPA (export/delete a tenant's data), enforces retention-policy auto-purge of raw message content after a configurable window, and tracks which messages had body content sent to the LLM for DPA-relevant disclosure accounting. This context is deliberately positioned as an **Open Host Service that other contexts conform to** — it defines the `DataLifecycleParticipant` contract that Mail Ingestion, Classification Engine, and LLM Adjudication implement so that export/purge orchestration never requires this context to reach into their internal storage directly.

Source layout convention:

```
src/audit-compliance/
  domain/
    entities/         # AuditEntry, DataSubjectRequest, RetentionPolicy
    value-objects/     # DecisionProvenance, DsrScope, LlmContentDisclosure
    events/            # Domain events
    services/          # DsrFulfillmentService, RetentionEnforcementService
    repositories/       # Repository interfaces
  application/         # SubmitDsr use case, RunRetentionPurge use case
  infrastructure/       # DataLifecycleParticipant adapters per participating context, Postgres (append-only) repos
  index.ts              # Public API of the context
```

## 1. Ubiquitous Language

| Term | Definition |
|---|---|
| **Audit Entry** | An immutable record of one classification decision or provider action — who/what/when/why and which layer decided. Never updated after creation. |
| **Decision Provenance** | Which layer produced a verdict (`RuleEngine`, `LlmAdjudication`, or `UserOverride`) plus confidence and rule/prompt version, attached to an Audit Entry. |
| **Data Subject Request (DSR)** | A GDPR/CCPA-driven request to export or delete a tenant's (or an end-user's) data, scoped to one or more mailboxes. |
| **Retention Policy** | The per-tenant rule governing how long raw message body content is retained before automatic purge. |
| **LLM Content Disclosure Record** | A tracked fact that a specific message's body content was sent to an LLM — the basis for DPA scope determination (per research report §5/§8). |
| **Purge Job** | A scheduled or triggered execution that deletes expired raw content across all participating contexts' storage. |
| **Data Lifecycle Participant** | Any bounded context that stores raw or derived message content and therefore must implement export/purge hooks defined by this context. |

## 2. Aggregates

### 2.1 Aggregate Root: `AuditEntry`

**Fields**: `auditEntryId`, `tenantId`, `mailboxId`, `messageId`, `actionType` (`ClassificationDecision | ProviderApply | ProviderRemove | LlmContentDisclosure | DsrExport | DsrDelete | RetentionPurge`), `actorType` (`System | User | Llm`), `decisionProvenance: DecisionProvenance | null`, `categories: InternalCategory[] | null`, `causedByAuditEntryId: AuditEntryId | null`, `occurredAt`, `evidenceRef` (a pointer/hash, never the raw content itself).

**Invariants**:
1. **Append-only**: no method on `AuditEntry` mutates state after construction; the repository interface exposes no `update`/`delete` — corrections to the record of fact are made by writing a *new* `AuditEntry`, never by editing history (matches [[07-feedback-learning]]'s Correction chain philosophy).
2. **Causal chain required for actions**: an `AuditEntry` of type `ProviderApply`/`ProviderRemove` must set `causedByAuditEntryId` pointing to the `ClassificationDecision` entry that produced the verdict being applied — an apply/remove action audited without a traceable originating decision is rejected at construction.
3. **No raw content storage**: `evidenceRef` is a pointer (hash of message content, or a reference ID into the owning context's own store) — the `AuditEntry` itself never embeds message body/subject text, keeping the audit log itself out of DPA scope expansion.

### 2.2 Aggregate Root: `DataSubjectRequest`

**Fields**: `dsrId`, `tenantId`, `requestType` (`Export | Delete`), `scope: DsrScope` (`{ mailboxIds: MailboxId[], dateRange?: {from, to} }`), `status` (`Received | InProgress | Completed | Failed | Rejected`), `participantAcknowledgments: Map<ParticipantContext, AckStatus>`, `requestedAt`, `completedAt`, `resultRef`.

**Invariants**:
1. **All-participant acknowledgment required for completion**: `status` cannot become `Completed` for a `Delete` request until every registered `DataLifecycleParticipant` (Mail Ingestion's raw store, Classification Engine's feature cache, LLM Adjudication's request/response cache) has returned a purge acknowledgment recorded in `participantAcknowledgments` — a partial purge leaves status `InProgress`, never silently `Completed`.
2. **Export honesty about prior purges**: an `Export` DSR must report already-purged content as explicitly "purged, unavailable as of `<date>`" rather than omit it silently or fabricate reconstructed content — the result set is built from what each participant actually has, not from what the system once had.
3. **No retroactive scope narrowing after `InProgress`**: once a `DataSubjectRequest` moves to `InProgress`, its `scope` is frozen — the orchestrating saga acted on a specific mailbox set and cannot have that set changed underneath it mid-flight; a scope change requires a new DSR.

### 2.3 Aggregate Root: `RetentionPolicy`

One instance per Tenant.

**Fields**: `tenantId`, `retentionWindowDays`, `appliesToCategories: InternalCategory[] | 'all'`, `lastPurgeRunAt`.

**Invariants**:
1. **Non-negative window**: `retentionWindowDays >= 0` (0 meaning "purge immediately after processing," the maximally privacy-conservative setting).
2. **Open-DSR purge guard**: `RunRetentionPurge` must exclude any message that is within the `scope` of a `DataSubjectRequest` currently `Received` or `InProgress` for an `Export` — content a user has asked to export is not auto-purged out from under that request; the purge for that message is deferred until the export completes or the window is re-evaluated on the next scheduled run.
3. **Policy change does not retroactively purge or restore**: updating `retentionWindowDays` to a shorter value takes effect only on the *next* scheduled `RunRetentionPurge` evaluation — it never triggers an immediate out-of-band purge, and it never resurrects content already purged under the prior (longer) window.

## 3. Value Objects

- `DecisionProvenance { source: 'RuleEngine' | 'LlmAdjudication' | 'UserOverride', confidence: number, ruleProfileVersion?: number, llmPromptVersion?: string }`
- `DsrScope { mailboxIds: MailboxId[], dateRange?: { from: Date, to: Date } }`
- `LlmContentDisclosure { messageId, mailboxId, sentToLlmAt, contentPortionsSent: ('subject' | 'body' | 'headers')[], promptVersion }`
- `AckStatus { participant: ParticipantContext, status: 'Pending' | 'Acknowledged' | 'Failed', acknowledgedAt?: Date }`

## 4. Domain Events

| Event | Payload (key fields) |
|---|---|
| `ClassificationDecisionAudited` | `auditEntryId, tenantId, mailboxId, messageId, decisionProvenance, categories, occurredAt` |
| `ProviderActionAudited` | `auditEntryId, causedByAuditEntryId, mailboxId, messageId, actionType, occurredAt` |
| `LlmContentDisclosureRecorded` | `auditEntryId, mailboxId, messageId, contentPortionsSent, promptVersion, occurredAt` |
| `DataSubjectRequestReceived` | `dsrId, tenantId, requestType, scope, requestedAt` |
| `DataSubjectRequestCompleted` | `dsrId, tenantId, requestType, resultRef, completedAt` |
| `DataSubjectRequestFailed` | `dsrId, tenantId, reason, failedAt` |
| `RetentionPolicyUpdated` | `tenantId, oldWindowDays, newWindowDays, updatedAt` |
| `RetentionPurgeExecuted` | `tenantId, messagesPurged, deferredForOpenDsr, purgedAt` |
| `RetentionPurgeFailed` | `tenantId, reason, failedAt` |

## 5. Commands

- `RecordClassificationDecisionAudit { tenantId, mailboxId, messageId, decisionProvenance, categories }`
- `RecordProviderActionAudit { causedByAuditEntryId, mailboxId, messageId, actionType }`
- `RecordLlmContentDisclosure { mailboxId, messageId, contentPortionsSent, promptVersion }`
- `SubmitDataSubjectRequest { tenantId, requestType, scope }`
- `ExecuteDataSubjectRequest { dsrId }`
- `AcknowledgeParticipantPurge { dsrId, participant, status }`
- `UpdateRetentionPolicy { tenantId, retentionWindowDays }`
- `RunRetentionPurge { tenantId }`

## 6. Repository Interfaces

```typescript
interface AuditEntryRepository {
  findById(auditEntryId: AuditEntryId): Promise<AuditEntry | null>;
  findByMessageId(mailboxId: MailboxId, messageId: MessageId): Promise<AuditEntry[]>; // full decision + action chain
  findByTenantAndDateRange(tenantId: TenantId, from: Date, to: Date, actionType?: ActionType, limit?: number, cursor?: string): Promise<Page<AuditEntry>>;
  findByCausedBy(auditEntryId: AuditEntryId): Promise<AuditEntry[]>; // downstream actions caused by one decision
  save(entry: AuditEntry): Promise<void>; // insert-only
}

interface DataSubjectRequestRepository {
  findById(dsrId: DsrId): Promise<DataSubjectRequest | null>;
  findOpenByMailbox(mailboxId: MailboxId): Promise<DataSubjectRequest[]>; // used by RetentionEnforcementService's purge guard
  findByTenant(tenantId: TenantId, status?: DsrStatus): Promise<DataSubjectRequest[]>;
  save(dsr: DataSubjectRequest): Promise<void>;
}

interface RetentionPolicyRepository {
  findByTenant(tenantId: TenantId): Promise<RetentionPolicy | null>;
  findDueForPurge(asOf: Date, limit: number): Promise<RetentionPolicy[]>;
  save(policy: RetentionPolicy): Promise<void>;
}
```

## 7. Domain Services

- **`DsrFulfillmentService`**: orchestrates the cross-context export/delete saga — issues export/purge requests to every registered `DataLifecycleParticipant` adapter, collects `AckStatus` responses, and drives `DataSubjectRequest.status` transitions per Invariant 2.1.1. Owns retry/timeout policy for slow or unresponsive participants.
- **`RetentionEnforcementService`**: scans `RetentionPolicy`s due for evaluation, computes the purge-eligible message set per tenant, applies the open-DSR guard (Invariant 2.3.2), and dispatches purge commands to participants, recording `RetentionPurgeExecuted`/`Failed`.
- **`AuditQueryService`**: read-side service backing compliance reporting and [[10-notification-reporting]]'s compliance-facing views — never exposes raw content, only decision/action metadata.

## 8. Anti-Corruption Layer / Integration Adapters

```typescript
// Defined and owned by Audit & Compliance (Open Host Service). Each context
// that stores raw or derived message content implements this port with its
// own adapter — Audit & Compliance never reaches into their storage directly.
interface DataLifecycleParticipant {
  exportData(scope: DsrScope): Promise<ParticipantExportResult>;
  purgeData(scope: DsrScope): Promise<ParticipantPurgeResult>;
  reportRetentionEligible(tenantId: TenantId, olderThan: Date): Promise<MessageId[]>;
}

interface ParticipantExportResult {
  participant: ParticipantContext;
  records: Array<{ messageId: MessageId; data: Record<string, unknown>; alreadyPurged: boolean }>;
}

interface ParticipantPurgeResult {
  participant: ParticipantContext;
  purgedMessageIds: MessageId[];
  failedMessageIds: MessageId[];
}
```

- **`MailIngestionDataLifecycleAdapter`**, **`ClassificationEngineDataLifecycleAdapter`**, **`LlmAdjudicationDataLifecycleAdapter`** — one per participating context, each implementing `DataLifecycleParticipant` against that context's own storage; `DsrFulfillmentService` and `RetentionEnforcementService` depend only on the port, never on a concrete participant's schema.

## 9. Relationships to Other Bounded Contexts

| Context | Relationship | Notes |
|---|---|---|
| **2. Mail Ingestion** | Open Host Service (this context is definer) | Mail Ingestion implements `DataLifecycleParticipant` against its `RawMessage` store; conforms to the port contract this context defines. |
| **3. Classification Engine** | Open Host Service (this context is definer) | Every finalized verdict triggers `RecordClassificationDecisionAudit`; also implements `DataLifecycleParticipant` for any cached features derived from message content. |
| **4. LLM Adjudication** | Open Host Service (this context is definer) + Customer–Supplier | Every LLM call triggers `RecordLlmContentDisclosure` (this context is Supplier of the disclosure-tracking obligation); LLM Adjudication also implements `DataLifecycleParticipant` for prompt/response caches. |
| **6. Action & Provider Sync** | Open Host Service (this context is definer) | Every `ProviderPrimitiveApplied`/`Removed` triggers `RecordProviderActionAudit`, chained via `causedByAuditEntryId` to the originating decision. |
| **7. Feedback & Learning** | Partnership | Corrections are jointly relevant facts — `CorrectionRecorded` events are durably audited here while Feedback & Learning owns the learning-algorithm semantics. |
| **8. Tenant & Account Management** | Customer–Supplier (this context is Customer) | `DsrScope.mailboxIds` and tenant boundaries are defined in terms of Tenant & Account Management's current/historical `MailboxConnection` records — this context queries there to resolve "what mailboxes does this DSR cover," including mailboxes disconnected before the request. |
| **10. Notification & Reporting** | Supplier (this context is Supplier) | `AuditQueryService` backs compliance dashboards and accuracy-adjacent reporting that needs decision provenance. |
| **1. Identity & Provider Access** | none direct | No dependency — credential lifecycle audit, if needed, flows through Tenant & Account Management's mailbox lifecycle events, not directly from Identity & Provider Access. |
| **5. Relationship Graph** | none direct | No dependency — VIP/reciprocity data is an input to Classification Engine's decisions, which are what gets audited, not a separate audit subject. |
