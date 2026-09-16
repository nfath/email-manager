# Bounded Context 05: Relationship Graph

## Purpose

Owns contact reciprocity/frequency scoring, the manually-curated VIP list, calendar co-attendance signal aggregation, and the People API/Contacts API integration used to distinguish genuine personal contacts from bulk senders (research §1 "People API for Contacts," §4 "Priority/VIP" row, §7 Stage 6 "VIP list; learned sender reputation"). This context is the source of truth for "how important is this sender to this user," consumed by Classification Engine (Context 03) as one more scoring input and by LLM Adjudication (Context 04) as prompt context.

---

## 1. Ubiquitous Language Glossary

| Term | Definition |
|---|---|
| **ContactRelationship** | The aggregate root: the accumulated relationship state between the mailbox owner and one external correspondent (email address), including reciprocity metrics, VIP status, and contact-book presence. |
| **ReciprocityScore** | A value object: a normalized measure of two-way engagement — sent-to-them count × replied-to-them count, normalized against overall mail volume (research §4). |
| **VipDesignation** | A value object recording that a correspondent is on the user-curated VIP list, distinct from (and an override on top of) the computed `ReciprocityScore`. |
| **CoAttendanceSignal** | A value object capturing shared calendar meeting attendance between the user and a correspondent, as a supporting VIP signal (research §4, roadmap V3). |
| **ContactBookPresence** | A value object recording whether a correspondent appears in the user's actual address book (People API `contacts.readonly`) versus only as an auto-detected "other contact" (`otherContacts.list`, read-only, research §1) versus neither. |
| **InteractionEvent** | An entity: one observed instance of the user sending to, or receiving and replying to, a given correspondent — the raw event stream `ReciprocityScore` is computed from. |
| **BulkSenderSignal** | A value object flagging that a correspondent's traffic pattern matches bulk/automated sending (used as a *negative* signal against personal-contact and VIP classification, complementing Context 03's header-based bulk detection). |
| **ReciprocityWindow** | A value object defining the rolling time window (e.g., trailing 180 days) over which `ReciprocityScore` is computed, so relationships decay naturally rather than staying permanently "important" from a single old exchange. |

---

## 2. Aggregate Root: `ContactRelationship`

### Identity
- `relationshipId: RelationshipId` (surrogate key)
- Natural key: `(tenantId, mailboxOwnerConnectionId, correspondentEmailAddress)` — one aggregate per (mailbox, correspondent) pair, since reciprocity is inherently mailbox-owner-relative, not global

### Fields
- `tenantId: TenantId`
- `mailboxOwnerConnectionId: ConnectionId` (whose mailbox this relationship is scored against — a reference to Context 01's aggregate, by ID only)
- `correspondentEmailAddress: EmailAddress`
- `correspondentDisplayName: string | null`
- `reciprocityScore: ReciprocityScore`
- `vipDesignation: VipDesignation | null` (null = not manually designated VIP)
- `coAttendanceSignal: CoAttendanceSignal`
- `contactBookPresence: ContactBookPresence`
- `bulkSenderSignal: BulkSenderSignal | null`
- `interactionSummary: InteractionSummary` (denormalized rollup counters, recomputed from `InteractionEvent`s — kept on the aggregate for cheap reads by Context 03's scoring without needing to replay the full event list every time)
- `lastInteractionAt: DateTime | null`
- `createdAt: DateTime`, `updatedAt: DateTime`

### Entity: `InteractionEvent` (child, append-only)
- `interactionEventId: InteractionEventId`
- `direction: 'SENT_BY_USER' | 'RECEIVED_BY_USER'`
- `wasReplyToExistingThread: boolean`
- `providerThreadId: string`
- `occurredAt: DateTime`
- `sourceMessageIdentity: MessageIdentity` (reference to Context 02's message, by ID only — no message content duplicated here)

### Value Objects
- `ReciprocityScore { sentCount: number, receivedAndRepliedCount: number, receivedNoReplyCount: number, normalizedScore: number, window: ReciprocityWindow, computedAt: DateTime }` — `normalizedScore` is the sent×replied product normalized against the mailbox owner's overall trailing-window mail volume, so a highly active mailbox and a quiet one produce comparable scores
- `VipDesignation { designatedBy: 'USER' | 'ADMIN', designatedAt: DateTime, note: string | null }`
- `CoAttendanceSignal { sharedMeetingCount: number, lastSharedMeetingAt: DateTime | null, window: ReciprocityWindow }`
- `ContactBookPresence { inPrimaryAddressBook: boolean, inAutoDetectedOtherContacts: boolean, lastSyncedAt: DateTime }`
- `BulkSenderSignal { matchedPattern: string, detectedAt: DateTime }`
- `InteractionSummary { totalSent: number, totalReceived: number, totalReplied: number, firstInteractionAt: DateTime | null }`
- `ReciprocityWindow { days: number }` (e.g., `{ days: 180 }`)

### Invariants

1. **A `VipDesignation` is a manual override that always outranks a computed `ReciprocityScore`, and one never overwrites the other silently.** Per research §4 ("manual VIP list... as deterministic override"), if `vipDesignation != null`, the relationship is treated as VIP for classification purposes regardless of `reciprocityScore.normalizedScore` — the aggregate must never auto-clear a `VipDesignation` based on reciprocity decay; only an explicit `RemoveVipDesignation` command can remove it.
2. **`ReciprocityScore` must recompute against a bounded, sliding `ReciprocityWindow`, never as an unbounded lifetime total.** Recomputing `normalizedScore` from `InteractionEvent`s older than `window.days` is prohibited — enforced by the scoring domain service only ever querying events within the window — so that a single old exchange years ago cannot keep inflating importance indefinitely (supports natural relationship decay, an explicit design choice distinguishing this from a naive all-time counter).
3. **`ContactBookPresence.inPrimaryAddressBook` can only be set `true` from a `contacts.readonly`/Graph `Contacts.Read` sync, never inferred from mail traffic alone.** Auto-detected correspondents (from mail/chat patterns) must be recorded only under `inAutoDetectedOtherContacts`, per the People API's own distinction (research §1: `otherContacts` vs `contacts.readonly`) — the aggregate rejects any command attempting to set `inPrimaryAddressBook = true` without a `ContactBookSyncPerformed` provenance token.
4. **A correspondent flagged with `BulkSenderSignal` cannot simultaneously hold a system-computed (non-manual) high `ReciprocityScore`-driven VIP inference.** If bulk-sender traits are detected (e.g., high one-way send volume with near-zero reply correlation, matching an automated-sender pattern), the normalized score computation must cap `normalizedScore` at a low ceiling regardless of raw sent/received counts — prevents a high-volume automated notification sender from being miscomputed as a close personal contact. This cap does not apply if a `VipDesignation` exists (invariant #1 still wins — a user can manually VIP a bulk-flagged address, e.g., a monitored ticketing system).
5. **`InteractionEvent`s are append-only and immutable; recomputation never deletes history.** `interactionSummary` and `reciprocityScore` are derived/cached fields recomputed from the event log, but the event log itself is never truncated except by an explicit data-retention policy command from Context 09 (Audit & Compliance) honoring the configured `ReciprocityWindow` plus a compliance retention buffer — routine scoring recomputation must not be the trigger for event deletion.
6. **Co-attendance signal contributes to VIP scoring only when both parties are confirmed meeting attendees, not merely invitees.** `sharedMeetingCount` increments only for meetings where the correspondent's calendar response status is `accepted` (or the meeting organizer field matches, for meetings without RSVP tracking) — an unanswered or declined invite must not count, to avoid inflating co-attendance from meetings that never actually happened together.

---

## 3. Domain Events

| Event | Payload |
|---|---|
| `InteractionRecorded` | `{ relationshipId, interactionEventId, direction, wasReplyToExistingThread, occurredAt }` |
| `ReciprocityScoreRecomputed` | `{ relationshipId, correspondentEmailAddress, previousScore: number, newScore: number, window, computedAt }` |
| `VipDesignationAdded` | `{ relationshipId, correspondentEmailAddress, designatedBy, note, designatedAt }` |
| `VipDesignationRemoved` | `{ relationshipId, correspondentEmailAddress, removedBy, removedAt }` |
| `ContactBookSyncPerformed` | `{ mailboxOwnerConnectionId, syncedCount, newContactsFound: number, syncedAt }` |
| `BulkSenderSignalDetected` | `{ relationshipId, correspondentEmailAddress, matchedPattern, detectedAt }` |
| `CoAttendanceSignalUpdated` | `{ relationshipId, sharedMeetingCount, lastSharedMeetingAt, occurredAt }` |
| `ContactRelationshipCreated` | `{ relationshipId, mailboxOwnerConnectionId, correspondentEmailAddress, createdAt }` (first-ever observed interaction) |

---

## 4. Commands

| Command | Description |
|---|---|
| `RecordInteraction` | Appends an `InteractionEvent`, creating a new `ContactRelationship` if none exists for the pair (`ContactRelationshipCreated`), triggered by messages passing through ingestion/classification (sent-folder observation and inbound-with-reply observation). |
| `RecomputeReciprocityScore` | Recalculates `reciprocityScore` from the windowed event log; scheduled periodically or triggered on new-interaction thresholds rather than on every single event, to bound compute cost. |
| `AddVipDesignation` | User/admin action adding a manual VIP override. |
| `RemoveVipDesignation` | User/admin action removing a manual VIP override (invariant #1 — the only path that clears it). |
| `SyncContactBook` | Pulls `contacts.readonly` (People API) or `Contacts.Read` (Graph) and updates `ContactBookPresence` for known correspondents; also surfaces net-new contacts. |
| `SyncOtherContacts` | Pulls `otherContacts.list`/`otherContacts:search` (Google-only, read-only auto-detected contacts) as a secondary, lower-confidence presence signal. |
| `RecordCoAttendance` | Updates `CoAttendanceSignal` from a calendar-meeting-attendee observation (accepted status only, invariant #6). |
| `DetectBulkSenderPattern` | Evaluates a correspondent's interaction pattern against bulk-sender heuristics and records `BulkSenderSignal` if matched. |
| `PurgeExpiredInteractionEvents` | Compliance-driven deletion of interaction events beyond the retention buffer, invoked by/on behalf of Context 09 (invariant #5). |

---

## 5. Repository Interface

```typescript
interface ContactRelationshipRepository {
  findByCorrespondent(
    mailboxOwnerConnectionId: ConnectionId,
    correspondentEmailAddress: EmailAddress
  ): Promise<ContactRelationship | null>;

  findOrCreate(
    mailboxOwnerConnectionId: ConnectionId,
    correspondentEmailAddress: EmailAddress
  ): Promise<ContactRelationship>;

  findAllVipForMailbox(mailboxOwnerConnectionId: ConnectionId): Promise<ContactRelationship[]>;

  findTopByReciprocityScore(
    mailboxOwnerConnectionId: ConnectionId,
    limit: number
  ): Promise<ContactRelationship[]>;

  findDueForScoreRecomputation(threshold: number, limit: number): Promise<ContactRelationship[]>;
  // relationships with enough new InteractionEvents since last computedAt to warrant a refresh

  findByBulkSenderFlag(mailboxOwnerConnectionId: ConnectionId): Promise<ContactRelationship[]>;

  save(relationship: ContactRelationship): Promise<void>;

  appendInteractionEvent(
    relationshipId: RelationshipId,
    event: InteractionEvent
  ): Promise<void>; // append-only fast path, distinct from full aggregate save

  purgeEventsOlderThan(
    mailboxOwnerConnectionId: ConnectionId,
    cutoff: DateTime
  ): Promise<{ purgedCount: number }>;
}
```

---

## 6. Domain Services

### `ReciprocityScoringService`
Spans a `ContactRelationship`'s windowed `InteractionEvent`s to compute the normalized sent×replied score (invariant #2), and applies the bulk-sender cap (invariant #4).

```typescript
interface ReciprocityScoringService {
  computeScore(
    events: InteractionEvent[],
    window: ReciprocityWindow,
    bulkSenderSignal: BulkSenderSignal | null,
    mailboxOverallVolume: number
  ): ReciprocityScore;
}
```

### `VipInferenceService`
Spans `ReciprocityScore`, `CoAttendanceSignal`, `VipDesignation`, and `BulkSenderSignal` together to produce the single effective "is this sender important" verdict Context 03 consumes as a scoring input — encapsulates the precedence rule of invariant #1 (manual override always wins) plus how co-attendance and reciprocity combine when no manual designation exists.

```typescript
interface VipInferenceService {
  inferEffectiveVipStatus(relationship: ContactRelationship): EffectiveVipStatus;
}

interface EffectiveVipStatus {
  isVip: boolean;
  source: 'MANUAL_DESIGNATION' | 'COMPUTED_RECIPROCITY_AND_COATTENDANCE' | 'NONE';
  confidence: number;
}
```

### `BulkSenderDetectionService`
Spans a correspondent's send/receive pattern (high one-way volume, near-zero reciprocation, regular/automated cadence) to flag `BulkSenderSignal` — complements but does not duplicate Context 03's header-based bulk detection (`Precedence: bulk`, `List-*` headers); this service works purely from behavioral pattern, useful when a bulk sender doesn't emit the standard headers.

```typescript
interface BulkSenderDetectionService {
  evaluate(events: InteractionEvent[]): BulkSenderSignal | null;
}
```

---

## 7. Anti-Corruption Layer / Integration Adapters

```typescript
interface ContactDirectoryPort {
  fetchPrimaryContacts(accessToken: TokenLease): Promise<DirectoryContact[]>;
  // Google People API contacts.readonly, or Graph Contacts.Read

  fetchAutoDetectedContacts(accessToken: TokenLease): Promise<DirectoryContact[]>;
  // Google People API otherContacts.list / otherContacts:search only — read-only, no Graph equivalent;
  // the ACL returns an empty result for Microsoft connections rather than erroring, since this is a
  // Google-specific capability gap the domain must tolerate gracefully
}

interface DirectoryContact {
  emailAddress: string;
  displayName: string | null;
  sourceContactId: string; // opaque provider-native contact ID, kept for potential future write-back (not currently supported per research: People API contacts are read-only from this integration)
}
```

```typescript
// Google People API adapter — isolates contacts.readonly + otherContacts.list/search
// endpoint shapes and their read-only constraint (research §1)
class GooglePeopleApiAdapter implements ContactDirectoryPort { /* ... */ }

// Microsoft Graph contacts adapter — isolates GET /me/contacts (Contacts.Read);
// implements fetchAutoDetectedContacts as a no-op returning [] (no Graph equivalent to otherContacts)
class GraphContactsAdapter implements ContactDirectoryPort { /* ... */ }
```

```typescript
interface CalendarAttendancePort {
  fetchAcceptedMeetingsWithAttendee(
    accessToken: TokenLease,
    attendeeEmail: EmailAddress,
    window: ReciprocityWindow
  ): Promise<CalendarMeetingSummary[]>;
  // Google Calendar API events.list with attendee filter, or Graph /me/events;
  // isolates each provider's distinct attendee-response-status vocabulary
  // (Google: responseStatus 'accepted'; Graph: attendee status.response 'accepted')
  // into a single normalized 'accepted' concept, supporting invariant #6
}
```

```typescript
class GoogleCalendarAttendanceAdapter implements CalendarAttendancePort { /* ... */ }
class GraphCalendarAttendanceAdapter implements CalendarAttendancePort { /* ... */ }
```

All three ACL interfaces obtain their `TokenLease` from Context 01's `TokenLeaseService` — this context never handles OAuth credentials directly.

---

## 8. Relationships to Other Bounded Contexts

- **Identity & Provider Access (Context 01) — Customer-Supplier, Context 01 upstream.** Pure consumer of `IssueTokenLease` scoped to contacts/calendar capabilities; degrades gracefully (per Context 01 invariant #2) when those scopes are absent from a given `MailboxConnection`, disabling `SyncContactBook`/`RecordCoAttendance` for that mailbox rather than failing.
- **Mail Ingestion (Context 02) — Customer-Supplier, Context 02 upstream.** Consumes header data (`From`/`To`/`Cc`, thread ID, timestamps) from normalized `RawMessage`s to drive `RecordInteraction` — never needs body content, consistent with content minimization.
- **Classification Engine (Context 03) — Open Host Service / Published Language, this context upstream.** Exposes `EffectiveVipStatus` and known-contact-domain lists (for lookalike-phishing-domain comparison) as a stable published interface; Context 03 treats this context's output as one more `Signal` input, never reaching into `InteractionEvent` internals directly.
- **LLM Adjudication (Context 04) — Customer-Supplier, this context upstream.** Supplies `EffectiveVipStatus`/reciprocity summary as read-only prompt context for `NEEDS_REPLY`/`PRIORITY_VIP` adjudication items.
- **Feedback & Learning (Context 07) — Customer-Supplier, Context 07 upstream.** When a user manually corrects a `PRIORITY_VIP` misclassification, Context 07 captures that correction and may issue `AddVipDesignation`/`RemoveVipDesignation` commands back into this context — Context 07 is the supplier of the correction signal, this context is the customer applying it to relationship state.
- **Tenant & Account Management (Context 08) — Customer-Supplier, Context 08 upstream.** References `tenantId` for multi-tenant scoping and respects tenant-level entitlement limits (e.g., a plan tier might cap co-attendance/calendar-signal features) without owning tenant lifecycle itself.
- **Audit & Compliance (Context 09) — Customer-Supplier, both directions.** This context supplies interaction-event data subject to retention policy (Context 09 dictates the purge cutoff, invariant #5, making Context 09 upstream for that specific policy); this context is upstream supplying `ContactBookSyncPerformed`/data-access events for audit trail and data-subject-request (DSR) fulfillment support, since contact data is itself personal data subject to GDPR/CCPA scope.
