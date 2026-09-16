# Bounded Context 02: Mail Ingestion

## Purpose

Owns detection of new/changed mail activity on a connected mailbox, and the normalization of provider-native message shapes (Gmail REST message resource, Microsoft Graph message resource) into a single internal representation (`RawMessage`) that every downstream context (Classification Engine, LLM Adjudication, Relationship Graph) consumes without ever branching on provider type. This is "the one place provider differences get absorbed" per the research report's Stage 1.

---

## 1. Ubiquitous Language Glossary

| Term | Definition |
|---|---|
| **RawMessage** | The aggregate root: an internal, provider-agnostic normalized representation of one email message, its headers, MIME structure, and body. |
| **MessageIdentity** | A value object uniquely identifying a message across provider re-delivery and across providers, derived from a hash of the RFC 5322 `Message-ID` header + the owning `MailboxConnection` ID (per research §3.4: "internal message identity mapped to provider-specific IDs"). |
| **WatchSubscription** | An entity representing a live push-notification registration with a provider: Gmail's `users.watch` (Pub/Sub topic binding) or a Graph webhook subscription. |
| **ReconciliationSweep** | An entity representing one run of the pull-based reconciliation fallback (Gmail `history.list` or Graph delta query) that catches changes missed by push. |
| **HistoryCursor** | A value object holding the provider's incremental-sync position: Gmail's `historyId` or Graph's `@odata.deltaLink`, scoped per `MailboxConnection`. |
| **NormalizedHeader** | A value object: `{ name: string, value: string }` — lower-cased header name, raw value, extracted uniformly from either provider's header array shape. |
| **MimePart** | A value object tree representing the parsed MIME structure (content-type, content-disposition, body, nested parts) — normalized from Gmail's nested `payload.parts` or Graph's `body`/`attachments` shape. |
| **IngestionState** | The lifecycle status of a `RawMessage` within this context: `RECEIVED` → `NORMALIZED` → `HANDED_OFF` (to Classification Engine) → `FAILED_NORMALIZATION`. |
| **PushNotificationEnvelope** | A value object wrapping the raw inbound webhook/Pub/Sub payload before it is resolved into concrete changed-message IDs. |
| **WatchRenewalSchedule** | A value object tracking next-renewal-due timestamp per `WatchSubscription`, respecting Gmail's 7-day (daily-recommended) vs Graph's ~3-day (24h-recommended) expiry windows. |

---

## 2. Aggregate Roots

### 2.1 `RawMessage` (primary aggregate root)

#### Identity
- `messageIdentity: MessageIdentity` (the natural/business key — `hash(Message-ID header, connectionId)`)
- `rawMessageId: RawMessageId` (UUID surrogate key for storage/reference)

#### Fields
- `connectionId: ConnectionId` (from Context 01 — referenced by ID only, never loaded as a foreign aggregate)
- `provider: Provider` (`GOOGLE` | `MICROSOFT`)
- `providerMessageId: string` (Gmail message `id` or Graph message `id` — opaque, provider-specific, kept for API calls back to the provider, e.g. by Context 06's action layer)
- `providerThreadId: string | null`
- `headers: NormalizedHeader[]`
- `mimeStructure: MimePart` (root MIME part, recursively containing children)
- `snippet: string` (short preview text, always available cheaply from both providers)
- `bodyFetchState: BodyFetchState` (`NOT_FETCHED` | `FETCHED` | `FETCH_SKIPPED_QUOTA` | `FETCH_FAILED`) — full body is fetched lazily/deliberately per research §5's content-minimization guidance, not eagerly for every message
- `bodyText: string | null`, `bodyHtml: string | null` (populated only when `bodyFetchState == FETCHED`)
- `receivedAt: DateTime` (provider's internal timestamp, e.g. Gmail `internalDate`)
- `sizeEstimateBytes: number`
- `ingestionState: IngestionState`
- `ingestionSource: IngestionSource` (`PUSH` | `RECONCILIATION_SWEEP`)
- `nativeCategoryHints: NativeCategoryHint[]` — Gmail `CATEGORY_*` labels or Graph `inferenceClassification` value, read through as free prior signal (research §3.3)
- `normalizationErrors: string[]` (non-fatal parse warnings retained for diagnostics)
- `createdAt: DateTime`, `updatedAt: DateTime`

#### Value Objects
- `MessageIdentity { hash: string, connectionId: ConnectionId }`
- `NormalizedHeader { name: string, value: string }`
- `MimePart { contentType: string, contentDisposition: string | null, charset: string | null, bodyRef: BodyRef | null, children: MimePart[], isCalendarCancelMethod: boolean, isDeliveryStatusNotification: boolean }` — the `isCalendarCancelMethod` flag is precomputed here (parsing `Content-Type: text/calendar; method=CANCEL` or a `BEGIN:VCALENDAR`/`METHOD:CANCEL` body, per research §4) because it's a structural MIME fact, not a classification judgment, so it belongs in ingestion, not in Context 03.
- `NativeCategoryHint { source: 'GMAIL_CATEGORY' | 'GRAPH_INFERENCE_CLASSIFICATION', value: string }`
- `BodyFetchState` (enum as above)
- `IngestionSource` (enum as above)

#### Invariants

1. **Message identity is immutable and unique per connection.** Once a `RawMessage` is persisted, its `MessageIdentity` never changes; a repository `save()` on an existing `MessageIdentity` for the same connection must be an idempotent update (re-normalizing), never a duplicate insert — required because both push delivery and reconciliation sweeps can observe the same underlying message (research §3.4, §7 Stage 6).
2. **No hand-off to Classification without minimum required headers parsed.** `ingestionState` cannot transition to `HANDED_OFF` unless `headers` contains at least `From`, `Date`, and either `Message-ID` or a provider-assigned fallback identity; missing all three forces `FAILED_NORMALIZATION` with a diagnostic reason, never a silent partial hand-off.
3. **Body content is fetched only when required.** `bodyFetchState` may only transition from `NOT_FETCHED` to `FETCHED` via an explicit `FetchMessageBody` command carrying a justification (`requestedByContext` + `reason` — e.g., "Stage 3 rule ambiguous, escalating"); a `RawMessage` must never have its body populated purely because it exists, per the content-minimization principle in research §5/§8. This invariant is the structural enforcement point for privacy-by-design.
4. **Calendar-cancellation detection requires no body fetch.** If `mimeStructure` contains a part whose `Content-Type` is `text/calendar; method=CANCEL` (or inline `BEGIN:VCALENDAR` + `METHOD:CANCEL` is present in a `text/plain`/`text/calendar` part already fetched as part of the metadata-level MIME preview both providers return), `isCalendarCancelMethod` must be set to `true` at normalization time without requiring a full-body fetch escalation — this is a deterministic, zero-cost signal per research §4.
5. **Thread/message size sanity bound.** `sizeEstimateBytes` above a configured ceiling (e.g., 25MB) forces `bodyFetchState = FETCH_SKIPPED_QUOTA` regardless of downstream requests, to protect Gmail's 20-quota-unit `messages.get` cost model and Graph batch payload caps (research §1, §2) from being defeated by a single oversized message.
6. **Native category hints are read-only observations, never mutated post-ingestion.** Once set from the provider's payload, `nativeCategoryHints` is immutable for the life of the `RawMessage` — if the provider's own classifier changes its mind later, that arrives as a new sync event and is recorded as an updated hint via a fresh normalization pass, not an in-place silent edit that would break auditability.

### 2.2 `WatchSubscription` (secondary aggregate root)

#### Identity
- `subscriptionId: SubscriptionId`

#### Fields
- `connectionId: ConnectionId`
- `provider: Provider`
- `providerSubscriptionRef: string` (Gmail: the watch's implicit binding, tracked via topic name + historyId baseline; Graph: the `subscriptionId` returned by Graph)
- `pubsubTopic: string | null` (Gmail-only)
- `webhookNotificationUrl: string | null` (Graph-only)
- `expiresAt: DateTime`
- `renewalSchedule: WatchRenewalSchedule`
- `status: WatchStatus` (`ACTIVE` | `EXPIRED` | `RENEWAL_FAILED` | `CANCELLED`)
- `lastRenewedAt: DateTime | null`
- `consecutiveRenewalFailures: number`

#### Invariants

1. **Renewal must be scheduled strictly before provider-enforced expiry.** For Gmail, `renewalSchedule.nextRenewalDue` must be ≤ `expiresAt - 24h` (daily renewal recommended against a 7-day expiry); for Graph, `nextRenewalDue` must be ≤ `expiresAt - (expiresAt - now > 24h ? 24h : remaining/2)` — concretely, Graph subscriptions (≈4,230 min ≈ 3 days) are renewed at least every 24h, a materially more aggressive cadence than Gmail's, and the aggregate must reject a `renewalSchedule` that violates its provider's minimum cadence (research §2).
2. **At most one `ACTIVE` `WatchSubscription` per `(connectionId, provider)`.** Creating a new watch for a connection that already has an `ACTIVE` one must first cancel/replace the prior one — required because Graph enforces max 1,000 subscriptions per mailbox across all apps, and duplicate watches waste that ceiling and can double-fire notifications.
3. **Renewal-failure circuit breaker mirrors Context 01's pattern.** After `consecutiveRenewalFailures >= 3`, status transitions to `RENEWAL_FAILED` and a `WatchRenewalPersistentlyFailed` event fires so the reconciliation sweep (below) becomes the sole ingestion path until manually/automatically re-established — the system must not go dark on a mailbox silently.

### 2.3 `ReconciliationSweep` (tracking entity, owned by `WatchSubscription`'s lifecycle but modeled as its own root for independent scheduling)

#### Identity
- `sweepId: SweepId`

#### Fields
- `connectionId: ConnectionId`
- `historyCursor: HistoryCursor` (before the sweep)
- `newHistoryCursor: HistoryCursor | null` (after a successful sweep)
- `triggeredBy: SweepTrigger` (`SCHEDULED_PERIODIC` | `WATCH_GAP_SUSPECTED` | `WATCH_RENEWAL_FAILED_FALLBACK`)
- `startedAt: DateTime`, `completedAt: DateTime | null`
- `messagesDiscovered: number`
- `status: SweepStatus` (`RUNNING` | `COMPLETED` | `FAILED`)
- `failureReason: string | null`

#### Invariants

1. **Cursor advances monotonically and only on full success.** `newHistoryCursor` is only committed back to the `MailboxConnection`'s persisted cursor position when `status == COMPLETED`; a `FAILED` sweep must leave the previously committed cursor untouched so the next sweep re-attempts from the last known-good position rather than silently skipping a gap (Gmail `historyId` expiry — beyond ~7 days / 1M events — is a special failure mode requiring a full `messages.list` resync, modeled as a distinct `failureReason`).
2. **A sweep must run at least once per reconciliation interval regardless of watch health**, per the "push notifies, pull reconciles" pattern explicitly recommended by both providers (research §7 Stage 7) — this is enforced by the scheduling domain service below, not by the aggregate itself, but the aggregate rejects overlapping `RUNNING` sweeps for the same `connectionId` (only one concurrent sweep per mailbox).

---

## 3. Domain Events

| Event | Payload |
|---|---|
| `RawMessageIngested` | `{ rawMessageId, messageIdentity, connectionId, provider, ingestionSource, receivedAt }` |
| `RawMessageNormalized` | `{ rawMessageId, messageIdentity, nativeCategoryHints, hasCalendarCancelMethod: boolean, occurredAt }` |
| `RawMessageNormalizationFailed` | `{ rawMessageId, connectionId, reason, occurredAt }` |
| `MessageBodyFetched` | `{ rawMessageId, requestedByContext, bodyFetchState, occurredAt }` |
| `MessageHandedOffForClassification` | `{ rawMessageId, messageIdentity, connectionId, occurredAt }` (this is the primary integration event Context 03 subscribes to) |
| `WatchSubscriptionEstablished` | `{ subscriptionId, connectionId, provider, expiresAt, occurredAt }` |
| `WatchSubscriptionRenewed` | `{ subscriptionId, newExpiresAt, renewedAt }` |
| `WatchRenewalFailed` | `{ subscriptionId, consecutiveFailures, occurredAt }` |
| `WatchRenewalPersistentlyFailed` | `{ subscriptionId, connectionId, occurredAt }` |
| `PushNotificationReceived` | `{ connectionId, provider, envelopeSummary, receivedAt }` (internal, triggers resolution into concrete message fetch/list calls) |
| `ReconciliationSweepStarted` | `{ sweepId, connectionId, triggeredBy, startedAt }` |
| `ReconciliationSweepCompleted` | `{ sweepId, connectionId, messagesDiscovered, newHistoryCursor, completedAt }` |
| `ReconciliationSweepFailed` | `{ sweepId, connectionId, failureReason, completedAt }` |

---

## 4. Commands

| Command | Description |
|---|---|
| `EstablishWatchSubscription` | Registers `users.watch` (Gmail) or a Graph webhook subscription for a `MailboxConnection`. |
| `RenewWatchSubscription` | Renews an existing subscription ahead of its provider-specific expiry window. |
| `CancelWatchSubscription` | Explicitly tears down a subscription (e.g., on `MailboxConnectionRevoked` from Context 01). |
| `HandlePushNotification` | Accepts an inbound webhook/Pub/Sub payload, resolves it to concrete changed-message IDs via `history.list`/delta query, and triggers `IngestRawMessage` for each. |
| `IngestRawMessage` | Fetches a message's headers/MIME metadata from the provider and constructs/updates a `RawMessage` in `RECEIVED` state. |
| `NormalizeRawMessage` | Parses headers, MIME structure, calendar/JSON-LD structural markers into the normalized shape; transitions to `NORMALIZED` or `FAILED_NORMALIZATION`. |
| `FetchMessageBody` | Escalates a specific `RawMessage` to full body fetch, carrying `requestedByContext` and `reason` (invariant #3). |
| `HandOffForClassification` | Transitions a successfully normalized `RawMessage` to `HANDED_OFF` and publishes `MessageHandedOffForClassification`. |
| `StartReconciliationSweep` | Begins a `history.list`/delta-query pull pass for a connection. |
| `CompleteReconciliationSweep` | Commits the new `HistoryCursor` and records discovered-message counts. |

---

## 5. Repository Interfaces

```typescript
interface RawMessageRepository {
  findByIdentity(identity: MessageIdentity): Promise<RawMessage | null>;

  findByProviderMessageId(
    connectionId: ConnectionId,
    providerMessageId: string
  ): Promise<RawMessage | null>;

  findPendingNormalization(limit: number): Promise<RawMessage[]>;

  findAwaitingBodyFetch(connectionId: ConnectionId, limit: number): Promise<RawMessage[]>;

  findByConnectionAndDateRange(
    connectionId: ConnectionId,
    from: DateTime,
    to: DateTime
  ): Promise<RawMessage[]>;

  save(message: RawMessage): Promise<void>; // upsert keyed on MessageIdentity (invariant #1)

  countBySizeCategory(connectionId: ConnectionId): Promise<{ oversized: number; normal: number }>;
}

interface WatchSubscriptionRepository {
  findActiveByConnection(connectionId: ConnectionId): Promise<WatchSubscription | null>;
  findDueForRenewal(before: DateTime): Promise<WatchSubscription[]>;
  findPersistentlyFailed(): Promise<WatchSubscription[]>;
  save(subscription: WatchSubscription): Promise<void>;
}

interface ReconciliationSweepRepository {
  findRunningByConnection(connectionId: ConnectionId): Promise<ReconciliationSweep | null>;
  findLastCompletedByConnection(connectionId: ConnectionId): Promise<ReconciliationSweep | null>;
  findDueForScheduledSweep(before: DateTime): Promise<ConnectionId[]>;
  save(sweep: ReconciliationSweep): Promise<void>;
}
```

---

## 6. Domain Services

### `MessageNormalizationService`
Spans `RawMessage` and the provider-specific ACL to translate wire payloads into the internal MIME/header model. Encapsulates the header-name case-folding, MIME-tree flattening, and structural calendar/JSON-LD marker detection shared by invariant #4.

```typescript
interface MessageNormalizationService {
  normalize(providerPayload: ProviderMessagePayload, connection: ConnectionSummary): NormalizationResult;
}

interface NormalizationResult {
  headers: NormalizedHeader[];
  mimeStructure: MimePart;
  nativeCategoryHints: NativeCategoryHint[];
  warnings: string[];
}
```

### `PushGapReconciliationService`
Spans `WatchSubscription` and `ReconciliationSweep` to decide when a gap is *suspected* (e.g., a renewal failure, or an unusually long silence on a normally active mailbox) and trigger an out-of-cycle sweep rather than waiting for the next scheduled one.

```typescript
interface PushGapReconciliationService {
  evaluateGapRisk(connectionId: ConnectionId): Promise<GapRiskAssessment>;
  triggerSweepIfNeeded(connectionId: ConnectionId): Promise<ReconciliationSweep | null>;
}
```

### `WatchRenewalSchedulingService`
Spans all `WatchSubscription`s across mailboxes to compute renewal cadence per provider's distinct expiry mechanics (invariant #1 of `WatchSubscription`) and drive the periodic renewal job.

```typescript
interface WatchRenewalSchedulingService {
  computeNextRenewal(subscription: WatchSubscription): DateTime;
  selectDueForRenewal(now: DateTime): Promise<WatchSubscription[]>;
}
```

---

## 7. Anti-Corruption Layer / Integration Adapters

```typescript
// Shared internal port consumed by application services — hides Gmail vs Graph payload shapes entirely
interface MailProviderPort {
  establishWatch(connection: ConnectionSummary): Promise<ProviderWatchResult>;
  renewWatch(subscription: WatchSubscription): Promise<ProviderWatchResult>;
  cancelWatch(subscription: WatchSubscription): Promise<void>;

  resolvePushNotification(
    connection: ConnectionSummary,
    envelope: PushNotificationEnvelope
  ): Promise<string[]>; // returns list of provider message IDs that changed

  fetchMessageMetadata(
    connection: ConnectionSummary,
    providerMessageId: string
  ): Promise<ProviderMessagePayload>; // headers + MIME structure preview, NOT full body (cheap call)

  fetchMessageBody(
    connection: ConnectionSummary,
    providerMessageId: string
  ): Promise<ProviderMessageBody>; // explicit, separate, deliberate call — supports invariant #3

  pullHistorySince(
    connection: ConnectionSummary,
    cursor: HistoryCursor
  ): Promise<{ changedMessageIds: string[]; newCursor: HistoryCursor; cursorExpired: boolean }>;
}

interface ProviderWatchResult {
  providerSubscriptionRef: string;
  expiresAt: DateTime;
  pubsubTopic?: string;
  webhookNotificationUrl?: string;
}
```

```typescript
// Gmail adapter — isolates users.watch + Pub/Sub envelope decoding (double base64),
// users.messages.get (format=metadata vs format=full), users.history.list,
// and the 250 qu/s / 20u-per-get quota bookkeeping (research §1)
class GmailIngestionAdapter implements MailProviderPort { /* ... */ }

// Graph adapter — isolates webhook subscription lifecycle (~4,230 min expiry),
// GET /me/mailFolders/inbox/messages/delta, batch request nested-429 unwrapping
// (a 429 inside a 200 batch envelope — documented pitfall, research §2), and
// the 4-concurrent-request-per-mailbox backend ceiling
class GraphIngestionAdapter implements MailProviderPort { /* ... */ }
```

---

## 8. Relationships to Other Bounded Contexts

- **Identity & Provider Access (Context 01) — Customer-Supplier, with Context 01 upstream.** This context is a pure *customer* of `IssueTokenLease`; it never manages credentials itself. It also listens to `MailboxConnectionRevoked`/`MailboxConnectionSuspendedByPolicy` (Conformist toward Context 01's event shape) to tear down watches and stop ingesting for that connection.
- **Classification Engine (Context 03) — Open Host Service / Published Language, this context upstream.** `MessageHandedOffForClassification` plus the `RawMessage` read model (headers, MIME structure, native category hints, calendar-cancel flag) constitute the published language Context 03 consumes; Context 03 never reaches back into a provider API itself.
- **LLM Adjudication (Context 04) — indirect, via Context 03.** This context has no direct relationship; body-fetch escalation requests (invariant #3) are issued by Context 03/04 through the `FetchMessageBody` command, making this context a *supplier* responding to an on-demand *customer* request — a Customer-Supplier relationship where Context 03/04 is the customer dictating when the (expensive, privacy-sensitive) body fetch is justified.
- **Relationship Graph (Context 05) — Open Host Service, this context upstream (partial).** Context 05 reads `RawMessage` header data (`From`, `To`, `Cc`, `Date`, thread ID) to build reciprocity/frequency signals, but does not require body content — consistent with content-minimization.
- **Action & Provider Sync (Context 06) — Partnership.** Both this context and Context 06 depend on the same `providerMessageId`/`providerThreadId` identifiers and provider-specific batch/throttling ceilings (Gmail `batchModify` 1,000 IDs/call, Graph 20 sub-requests/batch); the two contexts' ACLs should share the same low-level HTTP client/quota-tracking infrastructure (not the same domain model) to avoid divergent throttling behavior — a Partnership relationship requiring coordinated evolution of the shared infrastructure boundary, even though the domain models remain separate.
- **Feedback & Learning (Context 07) — no direct relationship.** Context 07 operates on classification verdicts and user corrections, not raw ingestion.
- **Audit & Compliance (Context 09) — Customer-Supplier, this context upstream.** Ingestion/normalization failures and body-fetch escalations are supplied as events Context 09 consumes for retention-policy enforcement and DPA-scope tracking (each body fetch is a discrete, auditable "content left metadata-only processing" event per research §8).
