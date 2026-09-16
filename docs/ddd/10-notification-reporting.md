# Bounded Context 10: Notification & Reporting

## Purpose

Delivers user-facing digests/summaries of triage activity, real-time alerts for high-confidence phishing detections (and other alert-worthy verdicts), and dashboards/reporting aggregates surfacing classification volume, accuracy trends (sourced from Feedback & Learning corrections), and LLM spend (sourced from the cost-governance data tracked alongside LLM Adjudication). This context is a **downstream consumer of nearly every other context's events**; it owns no classification logic of its own, only aggregation, scheduling, and delivery.

Source layout convention:

```
src/notification-reporting/
  domain/
    entities/         # DigestSchedule, AlertRule, Alert, MetricSnapshot
    value-objects/     # DeliveryChannel, MetricDimension, AlertCondition
    events/            # Domain events
    services/          # DigestCompositionService, AlertThrottleService, MetricAggregationService
    repositories/       # Repository interfaces
  application/         # GenerateDigest use case, EvaluateAlertRule use case
  infrastructure/       # NotificationDeliveryAdapter, cross-context event translators, Postgres repos
  index.ts              # Public API of the context
```

## 1. Ubiquitous Language

| Term | Definition |
|---|---|
| **Digest** | A periodic (daily/weekly), user-facing summary of triage activity across a mailbox or tenant. |
| **Alert** | A real-time, immediate notification instance for a high-priority event (e.g., a high-confidence phishing detection). |
| **Alert Rule** | The tenant-configured trigger condition (category + confidence threshold), delivery channel, and throttle window that governs when Alerts are created. |
| **Report** | An on-demand or scheduled aggregated metrics view (volume, accuracy, spend) over a time window. |
| **Metric Snapshot** | A point-in-time rollup of counters, computed idempotently from a watermarked range of upstream events, used to serve reports without recomputing from raw history each time. |
| **Delivery Channel** | The mechanism a notification is sent through: email digest, push notification, in-app banner, or outbound webhook. |
| **Throttle Window** | The bounded time period within which an `AlertRule` limits the number of Alerts it will actually send, to avoid alert fatigue. |
| **Source Event Watermark** | The last upstream event offset/timestamp already folded into a `MetricSnapshot`, preventing double-counting on recompute. |

## 2. Aggregates

### 2.1 Aggregate Root: `DigestSchedule`

One instance per `(tenantId, mailboxId, userId)` (a user may want a digest even if not the sole owner of a shared mailbox).

**Fields**: `digestScheduleId`, `tenantId`, `mailboxId`, `userId`, `cadence` (`Daily | Weekly`), `deliveryChannel`, `contentPreferences: InternalCategory[]`, `lastSentAt`, `nextScheduledAt`.

**Invariants**:
1. **Monotonic scheduling**: `nextScheduledAt` must always be strictly greater than `lastSentAt` — a digest cannot be scheduled to resend for a period already covered.
2. **No overlapping-window double-send**: changing `cadence` (e.g., Weekly → Daily) recomputes `nextScheduledAt` from `lastSentAt` forward under the *new* cadence, but never produces two digests whose content windows overlap — the transition digest's window starts exactly where the last one's ended, never re-including already-reported activity.

### 2.2 Aggregate Root: `AlertRule`

One instance per `(tenantId, category)` that the tenant has enabled for alerting.

**Fields**: `alertRuleId`, `tenantId`, `category`, `minConfidence`, `deliveryChannel`, `throttleWindow` (`{ durationMinutes, maxAlerts }`), `enabled`.

**Invariants**:
1. **Alert-eligible categories only**: an `AlertRule` may only be defined for categories flagged alert-eligible (e.g., `phishing`, `priority_vip`) — attempting to create one for, say, `newsletter` is rejected; not every category warrants real-time interruption.
2. **Confidence floor**: `minConfidence` must be ≥ a system-wide minimum (e.g., 0.7) — an `AlertRule` cannot be configured so loose that it fires on low-confidence, high-false-positive verdicts.

### 2.3 Aggregate Root: `Alert`

One instance per triggered evaluation of an `AlertRule` against a verdict.

**Fields**: `alertId`, `alertRuleId`, `tenantId`, `mailboxId`, `messageId`, `triggeringCategory`, `triggeringConfidence`, `status` (`Pending | Sent | Suppressed | Failed`), `sentAt`.

**Invariants**:
1. **Throttle enforcement without silent drop**: if the owning `AlertRule`'s `throttleWindow` max count has already been reached for the tenant in the current window, the `Alert` is created with `status = Suppressed`, not simply discarded — every trigger is recorded for audit/reporting continuity even when not delivered (see [[09-audit-compliance]] for the underlying decision audit; this aggregate records the *notification-layer* outcome, a distinct fact).
2. **Terminal status**: once `Sent`, `Suppressed`, or `Failed`, an `Alert`'s status does not change again — a delivery retry after `Failed` creates the retry as domain-service behavior (a new attempt tracked separately), not a mutation of the original record.

### 2.4 Aggregate Root: `MetricSnapshot`

One instance per `(tenantId, periodId, dimension)` — e.g., `(tenant-123, 2026-09, volumeByCategory)`.

**Fields**: `tenantId`, `periodId`, `dimension` (`volumeByCategory | accuracyTrend | llmSpend`), `values: Record<string, number>`, `sourceEventWatermark`, `computedAt`, `revision`.

**Invariants**:
1. **Idempotent recompute via watermark**: recomputing a snapshot for the same `(tenantId, periodId, dimension)` must only fold in events *after* `sourceEventWatermark` — never reprocess the same event twice into the aggregate values.
2. **Closed-period immutability with versioned revision**: once a `periodId` is in the past (period has fully elapsed), a recompute does not silently overwrite `values` — it increments `revision` and records a new `computedAt`, preserving every prior published revision for reproducibility of any report already delivered to a compliance stakeholder (aligns with [[09-audit-compliance]]'s reproducibility needs for compliance-facing reports).

## 3. Value Objects

- `DeliveryChannel` — enum `Email | Push | InApp | Webhook`, with channel-specific address/endpoint
- `MetricDimension` — enum `volumeByCategory | accuracyTrend | llmSpend`
- `AlertCondition { category: InternalCategory, minConfidence: number }`
- `ThrottleWindow { durationMinutes: number, maxAlerts: number }`

## 4. Domain Events

| Event | Payload (key fields) |
|---|---|
| `DigestGenerated` | `digestScheduleId, tenantId, mailboxId, windowStart, windowEnd, contentSummary, generatedAt` |
| `DigestSent` | `digestScheduleId, deliveryChannel, sentAt` |
| `DigestDeliveryFailed` | `digestScheduleId, reason, failedAt` |
| `AlertTriggered` | `alertId, alertRuleId, tenantId, mailboxId, messageId, triggeringCategory, triggeringConfidence, triggeredAt` |
| `AlertSent` | `alertId, deliveryChannel, sentAt` |
| `AlertSuppressed` | `alertId, alertRuleId, reason, suppressedAt` |
| `MetricSnapshotComputed` | `tenantId, periodId, dimension, values, revision, computedAt` |
| `ReportRequested` | `reportId, tenantId, dimension, periodRange, requestedBy, requestedAt` |
| `ReportGenerated` | `reportId, tenantId, resultRef, generatedAt` |

## 5. Commands

- `ScheduleDigest { tenantId, mailboxId, userId, cadence, deliveryChannel, contentPreferences }`
- `GenerateDigest { digestScheduleId }`
- `SendDigest { digestScheduleId, content }`
- `DefineAlertRule { tenantId, category, minConfidence, deliveryChannel, throttleWindow }`
- `EvaluateAlertRule { alertRuleId, mailboxId, messageId, category, confidence }`
- `SendAlert { alertId }`
- `ComputeMetricSnapshot { tenantId, periodId, dimension }`
- `RequestReport { tenantId, dimension, periodRange, requestedBy }`

## 6. Repository Interfaces

```typescript
interface DigestScheduleRepository {
  findById(digestScheduleId: DigestScheduleId): Promise<DigestSchedule | null>;
  findDueForGeneration(asOf: Date, limit: number): Promise<DigestSchedule[]>;
  findByMailbox(mailboxId: MailboxId): Promise<DigestSchedule[]>;
  save(schedule: DigestSchedule): Promise<void>;
}

interface AlertRuleRepository {
  findByTenantAndCategory(tenantId: TenantId, category: InternalCategory): Promise<AlertRule | null>;
  findEnabledByTenant(tenantId: TenantId): Promise<AlertRule[]>;
  save(rule: AlertRule): Promise<void>;
}

interface AlertRepository {
  findById(alertId: AlertId): Promise<Alert | null>;
  countInWindow(alertRuleId: AlertRuleId, windowStart: Date, windowEnd: Date, status?: AlertStatus): Promise<number>; // throttle check
  findByMessageId(mailboxId: MailboxId, messageId: MessageId): Promise<Alert[]>;
  save(alert: Alert): Promise<void>;
}

interface MetricSnapshotRepository {
  findLatest(tenantId: TenantId, periodId: string, dimension: MetricDimension): Promise<MetricSnapshot | null>;
  findRevisions(tenantId: TenantId, periodId: string, dimension: MetricDimension): Promise<MetricSnapshot[]>; // full revision history
  findRange(tenantId: TenantId, dimension: MetricDimension, from: string, to: string): Promise<MetricSnapshot[]>;
  save(snapshot: MetricSnapshot): Promise<void>;
}
```

## 7. Domain Services

- **`DigestCompositionService`**: gathers classification-volume and category-level activity for a `DigestSchedule`'s window (via `MetricSnapshot` reads, not raw event replay) and composes the digest content payload, respecting `contentPreferences`.
- **`AlertThrottleService`**: checks `AlertRepository.countInWindow` against the `AlertRule.throttleWindow` before allowing `EvaluateAlertRule` to create a `Sent`-bound `Alert`; produces `Suppressed` outcomes deterministically per Invariant 2.3.1.
- **`MetricAggregationService`**: the core rollup engine — consumes translated cross-context events (via the ACL translators below) and updates `MetricSnapshot`s idempotently using the watermark invariant; this is the one place raw event volume from five other contexts gets turned into a handful of reportable dimensions.

## 8. Anti-Corruption Layer / Integration Adapters

This context has no external (Gmail/Graph/LLM-provider) API dependency. Its adapters instead translate **other bounded contexts' Published Language events** into this context's own internal `MetricEvent` shape, so `MetricAggregationService` never has to understand five different upstream event schemas directly:

```typescript
interface MetricEvent {
  tenantId: TenantId;
  occurredAt: Date;
  dimension: MetricDimension;
  delta: Record<string, number>; // e.g., { [category]: 1 } for volume, { costUsd: 0.008 } for spend
}

interface CrossContextEventTranslator {
  translate(rawEvent: UpstreamDomainEvent): MetricEvent | null; // null if not reportable
}
```

- **`ClassificationVolumeTranslator`** — translates `ClassificationDecisionAudited`-style events (sourced via [[09-audit-compliance]]'s query API, not by re-deriving decisions itself) into `volumeByCategory` deltas.
- **`AccuracyTrendTranslator`** — translates `CorrectionRecorded` events from [[07-feedback-learning]] into `accuracyTrend` deltas (false-positive/negative counts per category).
- **`LlmSpendTranslator`** — translates cost-governance/spend-tracking events emitted alongside [[04-llm-adjudication]]'s calls into `llmSpend` deltas.
- **`PhishingAlertTranslator`** — translates high-confidence `phishing` verdicts from [[03-classification-engine]] / [[04-llm-adjudication]] directly into `EvaluateAlertRule` invocations (this is the one translator that drives a command, not just a metric delta, since phishing alerting is time-sensitive).
- **`NotificationDeliveryAdapter`** — the true external-system ACL: abstracts the actual delivery mechanism (email/push/webhook provider) behind a single port so `SendDigest`/`SendAlert` domain logic never depends on a specific vendor SDK.

```typescript
interface NotificationDeliveryAdapter {
  deliver(channel: DeliveryChannel, payload: DigestPayload | AlertPayload): Promise<{ delivered: boolean; providerRef?: string }>;
}
```

## 9. Relationships to Other Bounded Contexts

| Context | Relationship | Notes |
|---|---|---|
| **3. Classification Engine** | Conformist (this context is downstream) | Consumes phishing/high-confidence verdict events as-is via `PhishingAlertTranslator`; does not influence scoring. |
| **4. LLM Adjudication** | Conformist (this context is downstream) | Consumes spend-tracking events via `LlmSpendTranslator` for the LLM-spend reporting dimension. |
| **6. Action & Provider Sync** | Conformist (this context is downstream) | Consumes `MessageSyncCompleted`/`Failed` and `ReconciliationSweepCompleted` volumes for sync-health reporting. |
| **7. Feedback & Learning** | Conformist (this context is downstream) | Consumes `CorrectionRecorded`/calibration events via `AccuracyTrendTranslator` for the accuracy-trend reporting dimension. |
| **9. Audit & Compliance** | Partnership | Jointly defines what compliance-facing reports may surface (raw content never crosses into a report); `ClassificationVolumeTranslator` reads through Audit & Compliance's `AuditQueryService` rather than re-deriving decisions independently. |
| **8. Tenant & Account Management** | Conformist | Reads `tenantId`/`mailboxId` scoping and plan-tier context (e.g., whether a tenant's plan includes dashboards at all) as given. |
| **1. Identity & Provider Access** | none direct | No dependency. |
| **2. Mail Ingestion** | none direct | No dependency — ingestion volume is not itself a reporting dimension in this design; only classification/sync/spend outcomes are. |
| **5. Relationship Graph** | none direct | No dependency — VIP scoring feeds Classification Engine's verdicts, which are what this context reports on, not a direct source. |
