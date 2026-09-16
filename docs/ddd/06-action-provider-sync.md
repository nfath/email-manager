# Bounded Context 6: Action & Provider Sync

## Purpose

Translates finalized internal category verdicts (produced by the **Classification Engine** and **LLM Adjudication** contexts) into provider-native primitives — Gmail labels applied via `batchModify`, and Outlook categories (with selective folder moves) applied via Graph batch requests. This context owns the **idempotent diff-and-apply** logic: it never blind-overwrites provider state, it computes what is currently applied vs. what should be applied and issues only the delta.

Source layout convention:

```
src/action-provider-sync/
  domain/
    entities/         # MessageSyncState, ProviderCategoryMapping
    value-objects/     # ProviderPrimitiveRef, SyncDiff, AppliedPrimitive
    events/            # Domain events
    services/          # SyncDiffService, BatchChunkingService
    repositories/       # Repository interfaces
  application/         # RequestCategorySync use case, ReconciliationSweep use case
  infrastructure/       # GmailLabelSyncAdapter, OutlookCategorySyncAdapter, Postgres repos
  index.ts              # Public API of the context
```

## 1. Ubiquitous Language

| Term | Definition |
|---|---|
| **Category Verdict** | The finalized set of internal categories assigned to a message, handed off from Classification Engine or LLM Adjudication. Immutable input to this context. |
| **Provider Primitive** | A provider-native mechanism used to represent a category: a Gmail label, an Outlook category, or (rarely) a folder move. |
| **Applied Primitive State** | The set of provider primitives this context believes are currently attached to a message, as of the last confirmed sync. |
| **Desired Primitive State** | The set of provider primitives that *should* be attached, derived from the current Category Verdict via the Provider Category Mapping. |
| **Sync Diff** | The computed `{toAdd, toRemove}` sets obtained by comparing Desired vs. Applied state. The unit of work sent to the provider. |
| **Provider Category Mapping** | The per-mailbox, per-internal-category record of which provider primitive (label ID / category name) represents it. Created once, cached, reused. |
| **Batch Window** | A grouping of one or more message syncs chunked to fit provider batch constraints (Gmail ≤1,000 message IDs/call; Graph ≤20 sub-requests/batch, 4MB cap). |
| **Reconciliation Sweep** | A periodic re-read of provider-side state to detect drift (e.g., a user manually removed a label in Gmail directly) and correct the Applied Primitive State record. |
| **Manual Override Category** | A category the user has explicitly pinned/unpinned through direct provider action or the product UI; this context must not silently revert it (see [[07-feedback-learning]]). |

## 2. Aggregates

### 2.1 Aggregate Root: `MessageSyncState`

One instance per `(mailboxId, messageId)`. Tracks the sync lifecycle for a single message across its provider.

**Fields**: `mailboxId`, `messageId` (internal identity, hash of Message-ID header + account per the research report §3.4), `provider` (`gmail | outlook`), `desiredCategories: InternalCategory[]`, `appliedPrimitives: AppliedPrimitive[]`, `manualOverrideCategories: InternalCategory[]`, `syncStatus` (`Pending | Diffed | Applying | Synced | Failed`), `lastVerdictId`, `lastSyncedAt`, `retryCount`, `lastError`.

**Invariants**:
1. **No-op idempotency**: `computeDiff()` must return an empty `SyncDiff` if `desiredPrimitives(desiredCategories) == appliedPrimitives` — re-running sync for an already-synced message issues zero provider calls. This is the concrete form of "safe to reapply."
2. **Manual override respected**: a category present in `manualOverrideCategories` is excluded from `toRemove` even if the latest Category Verdict no longer includes it — this context never fights a user's explicit correction (that correction record lives in [[07-feedback-learning]]; this aggregate only holds the resulting exclusion flag).
3. **Verdict monotonicity**: a `MessageSyncState` only accepts a new Category Verdict whose `verdictId` is newer (by timestamp/sequence) than `lastVerdictId` — an out-of-order/stale verdict delivery (possible with async LLM Adjudication batches) is rejected, not applied.
4. **Transition guard**: `syncStatus` can only move `Applying → Synced` after every primitive in the diff has an explicit provider-confirmed result (success or permanent failure) — partial application leaves status `Failed`, never `Synced`, so a later reconciliation sweep will retry the remainder.
5. **Retry ceiling**: `retryCount` beyond a configured max (e.g., 5) stops automatic retries and requires the Reconciliation Sweep (not the original apply path) to pick it back up, preventing tight failure loops against a rate-limited provider.

### 2.2 Aggregate Root: `ProviderCategoryMapping`

One instance per `(mailboxId, internalCategory)`.

**Fields**: `mailboxId`, `internalCategory`, `providerPrimitiveRef: ProviderPrimitiveRef` (`{type: 'label'|'category'|'folder', providerId, displayName}`), `createdAt`, `active`.

**Invariants**:
1. **Uniqueness**: exactly one **active** mapping per `(mailboxId, internalCategory)` — creating a second active mapping for the same pair is rejected; the caller must deactivate the old one first (needed because Outlook category `displayName` is immutable once created per the Graph API — remapping means creating a new category, not renaming, per research report §2).
2. **Provider primitive type consistency**: for a given `provider`, `providerPrimitiveRef.type` must be a type that provider supports (Gmail: `label` only; Outlook: `category` or `folder`) — enforced at creation, not left to the adapter to silently coerce.
3. **Immutability of `providerId` once created**: only `active` may transition (true → false on deprecation); `providerId`/`displayName` are set once at creation and never mutated in place, matching the Outlook constraint and keeping Gmail's cached label ID trustworthy without a re-lookup.

## 3. Value Objects

- `AppliedPrimitive { primitiveType, providerId, appliedAt }`
- `ProviderPrimitiveRef { type: 'label' | 'category' | 'folder', providerId, displayName }`
- `SyncDiff { toAdd: ProviderPrimitiveRef[], toRemove: ProviderPrimitiveRef[] }`
- `InternalCategory` — enum of the 11 taxonomy values (`newsletter`, `job_posting`, `social`, `ecommerce_receipt`, `promo_deal`, `linkedin`, `meeting_cancelled`, `needs_reply`, `priority_vip`, `phishing`, `personal`)

## 4. Domain Events

| Event | Payload (key fields) |
|---|---|
| `CategorySyncRequested` | `mailboxId, messageId, verdictId, desiredCategories, requestedAt` |
| `SyncDiffComputed` | `mailboxId, messageId, toAdd, toRemove, computedAt` |
| `ProviderPrimitiveApplied` | `mailboxId, messageId, primitiveType, providerId, appliedAt` |
| `ProviderPrimitiveRemoved` | `mailboxId, messageId, primitiveType, providerId, removedAt` |
| `MessageSyncCompleted` | `mailboxId, messageId, verdictId, appliedPrimitives, completedAt` |
| `MessageSyncFailed` | `mailboxId, messageId, verdictId, failedPrimitives, error, retryCount` |
| `ProviderCategoryMappingCreated` | `mailboxId, internalCategory, providerPrimitiveRef, createdAt` |
| `ReconciliationSweepCompleted` | `mailboxId, messagesChecked, driftDetectedCount, correctionsIssued, sweptAt` |

## 5. Commands

- `RequestCategorySync { mailboxId, messageId, verdictId, desiredCategories }`
- `ComputeSyncDiff { mailboxId, messageId }`
- `ApplyProviderPrimitives { mailboxId, messageId, diff }`
- `RecordAppliedState { mailboxId, messageId, appliedPrimitives }`
- `RetryFailedSync { mailboxId, messageId }`
- `RunReconciliationSweep { mailboxId, windowStart, windowEnd }`
- `CreateProviderCategoryMapping { mailboxId, internalCategory, providerPrimitiveRef }`
- `DeactivateProviderCategoryMapping { mailboxId, internalCategory }`

## 6. Repository Interfaces

```typescript
interface MessageSyncStateRepository {
  findByMessageId(mailboxId: MailboxId, messageId: MessageId): Promise<MessageSyncState | null>;
  findPendingSync(mailboxId: MailboxId, limit: number): Promise<MessageSyncState[]>;
  findByStatus(mailboxId: MailboxId, status: SyncStatus, limit: number, cursor?: string): Promise<Page<MessageSyncState>>;
  findStaleSynced(mailboxId: MailboxId, olderThan: Date, limit: number): Promise<MessageSyncState[]>; // reconciliation candidates
  save(state: MessageSyncState): Promise<void>;
  saveBatch(states: MessageSyncState[]): Promise<void>; // one txn per Gmail batchModify/Graph batch window
}

interface ProviderCategoryMappingRepository {
  findActive(mailboxId: MailboxId, internalCategory: InternalCategory): Promise<ProviderCategoryMapping | null>;
  findAllActive(mailboxId: MailboxId): Promise<ProviderCategoryMapping[]>;
  save(mapping: ProviderCategoryMapping): Promise<void>;
  deactivate(mailboxId: MailboxId, internalCategory: InternalCategory): Promise<void>;
}
```

## 7. Domain Services

- **`SyncDiffService`**: computes `SyncDiff` for a `MessageSyncState` given its desired categories, the active `ProviderCategoryMapping`s for the mailbox, and manual-override exclusions. Pure, no I/O.
- **`BatchChunkingService`**: groups pending `MessageSyncState`s into provider-legal batch windows — Gmail: chunks of ≤1,000 message IDs per `batchModify` call, staying under the 250 quota-units/sec per-user ceiling; Outlook: chunks of ≤20 sub-requests per batch, staying under the ~4 concurrent-requests-per-mailbox effective limit, and unwraps nested per-sub-request 429s from the outer 200 envelope rather than treating the whole batch as successful (a documented Graph pitfall per the research report §2).
- **`ReconciliationService`**: drives the periodic sweep — re-reads provider-side label/category state for a sample of `Synced` messages, compares against `appliedPrimitives`, and for any drift either (a) updates the recorded applied state if the divergence looks like a legitimate external change, or (b) emits a signal consumed by [[07-feedback-learning]] as an implicit correction candidate.

## 8. Anti-Corruption Layer / Integration Adapters

```typescript
// Domain-facing port — no Gmail/Graph shapes leak past this boundary
interface ProviderSyncPort {
  applyPrimitives(mailboxId: MailboxId, ops: BatchSyncOperation[]): Promise<ProviderSyncResult[]>;
  readAppliedPrimitives(mailboxId: MailboxId, messageIds: MessageId[]): Promise<Map<MessageId, ProviderPrimitiveRef[]>>;
  createPrimitive(mailboxId: MailboxId, internalCategory: InternalCategory, displayName: string): Promise<ProviderPrimitiveRef>;
}

interface BatchSyncOperation {
  messageId: MessageId;
  add: ProviderPrimitiveRef[];
  remove: ProviderPrimitiveRef[];
}

interface ProviderSyncResult {
  messageId: MessageId;
  succeeded: ProviderPrimitiveRef[];
  failed: { primitive: ProviderPrimitiveRef; reason: string }[];
}
```

- **`GmailLabelSyncAdapter implements ProviderSyncPort`**: translates `BatchSyncOperation[]` into `users.messages.batchModify` calls (`addLabelIds`/`removeLabelIds`), chunked to ≤1,000 IDs; calls `users.labels.create` (5 quota units) lazily on first `createPrimitive`, respecting the 500-user-label hard limit; surfaces per-message failures individually even though `batchModify` itself has no per-message result shape (retries singly on partial batch failure).
- **`OutlookCategorySyncAdapter implements ProviderSyncPort`**: translates operations into Graph `$batch` requests against `PATCH /me/messages/{id}` (categories array) or folder-move actions for the narrow archived-out-of-inbox case; unwraps each sub-response status individually before reporting success (never trusts the outer 200); creates master categories via the mastercategories API on first use, respecting the 25-color/unlimited-name constraint and the immutable-`displayName` rule (mapping deactivation + recreation, never rename).

## 9. Relationships to Other Bounded Contexts

| Context | Relationship | Notes |
|---|---|---|
| **3. Classification Engine** | Customer–Supplier (this context is Customer) | Consumes `CategoryVerdictFinalized`-style events as the trigger for `RequestCategorySync`. Cannot force upstream's scoring behavior. |
| **4. LLM Adjudication** | Customer–Supplier (this context is Customer) | Same relationship as above for verdicts that were escalated; must respect verdict ordering/monotonicity (Invariant 3) since adjudication batches can resolve out of order. |
| **1. Identity & Provider Access** | Conformist | Accepts OAuth-scoped API clients/tokens exactly as issued; does not negotiate scope — if a token lacks `gmail.labels` or `MailboxSettings.ReadWrite`, this context fails closed rather than requesting elevated access itself. |
| **7. Feedback & Learning** | Partnership | Reconciliation-detected drift and manual-override flags are shared bidirectionally: this context surfaces drift signals; Feedback & Learning turns them into Corrections and publishes back which categories are user-pinned. |
| **8. Tenant & Account Management** | Conformist | Reads `MailboxConnection` lifecycle state (Active/Paused/Disconnected) to decide whether a mailbox is sync-eligible at all; a Paused mailbox halts all `RequestCategorySync` processing. |
| **9. Audit & Compliance** | Open Host Service (Audit & Compliance is the definer) | Every `ProviderPrimitiveApplied`/`Removed` event is also published to Audit & Compliance's `DataLifecycleParticipant`-style ingestion path for immutable logging; this context conforms to that published event contract rather than inventing its own audit format. |
| **10. Notification & Reporting** | Supplier (this context is Supplier) | Publishes `MessageSyncCompleted`/`Failed` and `ReconciliationSweepCompleted` volumes as raw material for dashboards; Notification & Reporting is a Conformist consumer of these event shapes. |
| **2. Mail Ingestion** | none direct | No direct dependency — Mail Ingestion produces `RawMessage`s consumed upstream by Classification Engine, not by this context. |
| **5. Relationship Graph** | none direct | No direct dependency — VIP/reciprocity scoring feeds Classification Engine, which is this context's upstream supplier. |
