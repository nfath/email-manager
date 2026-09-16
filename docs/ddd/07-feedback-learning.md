# Bounded Context 7: Feedback & Learning

## Purpose

Captures user corrections — manual re-label/un-label actions taken either through the product UI or detected as drift directly in the provider (Gmail/Outlook) — and turns them into adjusted rule weights for the **Classification Engine** and updated few-shot examples for **LLM Adjudication**. Corrections are never applied back to those contexts synchronously or by direct method call; this context accumulates evidence and **publishes** learned adjustments as events, so Classification Engine and LLM Adjudication remain decoupled from this context's internal learning algorithm.

Source layout convention:

```
src/feedback-learning/
  domain/
    entities/         # Correction, RuleWeightProfile, FewShotExampleSet
    value-objects/     # FeedbackSignal, WeightDelta, FewShotExample
    events/            # Domain events
    services/          # FeedbackAggregationService, CalibrationService
    repositories/       # Repository interfaces
  application/         # RecordCorrection use case, learning-batch scheduler
  infrastructure/       # Provider-divergence adapter, Postgres repos
  index.ts              # Public API of the context
```

## 1. Ubiquitous Language

| Term | Definition |
|---|---|
| **Correction** | An immutable record that a message's applied categories were changed by the user (or detected as diverging from the system's verdict), distinct from the original verdict that produced them. |
| **Feedback Signal** | The normalized, learning-algorithm-ready projection of a Correction: which rule/signal or LLM prompt version produced the wrong verdict, and what the right answer was. |
| **Rule Weight Profile** | The current per-category, per-signal weight set consumed by the Classification Engine's additive scoring model (SpamAssassin/Rspamd-style, per research report §6). Owned and versioned here, not in Classification Engine. |
| **Weight Delta** | A bounded, proposed change to one signal's weight within a Rule Weight Profile, derived from aggregated Feedback Signals. |
| **Few-Shot Example** | A curated `(message features, correct category)` pair added to or evicted from the example pool used in LLM Adjudication's cached prompt. |
| **Learning Batch** | An aggregated set of Feedback Signals processed together (async, per research report §7 Stage 6) rather than adjusting weights on every single correction. |
| **Implicit Correction** | A Correction inferred from Action & Provider Sync's reconciliation-sweep drift detection (user changed a label directly in Gmail/Outlook) rather than from an explicit in-product action. |
| **Calibration Threshold** | The per-category confidence cutoff above which the Classification Engine finalizes a verdict without LLM escalation; tuned here based on observed false-positive/negative rates. |

## 2. Aggregates

### 2.1 Aggregate Root: `Correction`

One instance per correction event. **Append-only** — never updated after creation.

**Fields**: `correctionId`, `mailboxId`, `messageId`, `priorVerdictId`, `previousCategories: InternalCategory[]`, `newCategories: InternalCategory[]`, `correctedBy` (`userId | 'system-reconciliation'`), `correctionSource` (`ExplicitUiAction | ProviderDivergence`), `originalDecisionSource` (`RuleEngine | LlmAdjudication`), `originalConfidence`, `correctedAt`.

**Invariants**:
1. **Prior verdict required**: a `Correction` must reference an existing `priorVerdictId` obtained from Classification Engine or LLM Adjudication — a message that was never classified cannot be "corrected" (there is nothing to correct against); such an action is a fresh manual categorization, modeled elsewhere, not a `Correction`.
2. **Non-trivial change**: `newCategories` must differ from `previousCategories` (set-inequality) — a Correction recording an identical set is rejected as a no-op, so learning signals are never diluted by duplicate/no-change events.
3. **Immutability**: once created, a `Correction` exposes no mutating methods — it is a fact of history; a later reversal by the user is recorded as a *new* `Correction` referencing this one's resulting state as its new `previousCategories`, preserving the full correction chain for audit (see [[09-audit-compliance]]).

### 2.2 Aggregate Root: `RuleWeightProfile`

One instance per tenant (or a shared global default profile that a tenant can opt out of and fork).

**Fields**: `profileId`, `tenantId | 'global-default'`, `weights: Map<InternalCategory, Map<SignalId, number>>`, `version`, `lastAdjustedAt`.

**Invariants**:
1. **Bounded delta per adjustment cycle**: any single `WeightDelta` applied to a signal's weight is capped at ±0.2 (of a 0–1 normalized weight range) — prevents one correction, or a small burst of them, from swinging the rule engine's scoring disproportionately (a single mis-click should not retrain the system).
2. **Minimum sample size**: a `WeightDelta` for a given `(category, signalId)` is only computed and applied from a `Learning Batch` containing at least 5 Feedback Signals implicating that same signal — below that threshold, the signal is tracked but no adjustment is proposed, avoiding overfitting to one user's idiosyncratic one-off correction.
3. **Monotonic versioning**: every applied `WeightDelta` increments `version`; the profile is never edited in place without a version bump, so the Classification Engine can detect and pick up changes via version comparison rather than polling for content diffs.

### 2.3 Aggregate Root: `FewShotExampleSet`

One instance per internal category.

**Fields**: `category`, `examples: FewShotExample[]`, `maxSize` (default 50), `lastCuratedAt`.

**Invariants**:
1. **Bounded pool size**: `examples.length` never exceeds `maxSize` — adding an example beyond capacity requires an eviction in the same operation (oldest-and-lowest-value example evicted first), keeping the cached LLM Adjudication prompt's few-shot block bounded in token cost.
2. **No duplicate source message**: two `FewShotExample`s cannot originate from the same `messageId` — a single message can only contribute one lesson to the pool, preventing one heavily-corrected message from crowding out diverse examples.

## 3. Value Objects

- `FeedbackSignal { messageId, signalIdOrPromptVersion, category, wasFalsePositive: boolean, wasFalseNegative: boolean, source }`
- `WeightDelta { category, signalId, delta, sampleSize, computedAt }`
- `FewShotExample { messageId, features: Record<string, unknown>, correctCategory, addedAt, value: number }` — `value` is a curation score (e.g., recency-weighted correction frequency for that message shape)

## 4. Domain Events

| Event | Payload (key fields) |
|---|---|
| `CorrectionRecorded` | `correctionId, mailboxId, messageId, previousCategories, newCategories, correctionSource, correctedAt` |
| `LearningBatchAssembled` | `batchId, feedbackSignalCount, categoriesTouched, assembledAt` |
| `RuleWeightAdjusted` | `profileId, category, signalId, delta, newWeight, version, adjustedAt` |
| `FewShotExampleAdded` | `category, messageId, value, addedAt` |
| `FewShotExampleEvicted` | `category, messageId, reason, evictedAt` |
| `LearningAdjustmentPublished` | `profileId, version, affectedCategories, publishedAt` — the Published Language event consumed downstream |
| `CalibrationThresholdUpdated` | `category, oldThreshold, newThreshold, observedFalsePositiveRate, updatedAt` |

## 5. Commands

- `RecordCorrection { mailboxId, messageId, priorVerdictId, newCategories, correctedBy, correctionSource }`
- `AssembleLearningBatch { since: Date }`
- `ComputeRuleWeightAdjustment { batchId }`
- `PublishLearningAdjustment { profileId }`
- `AddFewShotExample { category, messageId, features, correctCategory }`
- `EvictFewShotExample { category, messageId, reason }`
- `RecalibrateThreshold { category }`

## 6. Repository Interfaces

```typescript
interface CorrectionRepository {
  findById(correctionId: CorrectionId): Promise<Correction | null>;
  findByMessageId(mailboxId: MailboxId, messageId: MessageId): Promise<Correction[]>; // full chain
  findSince(tenantId: TenantId, since: Date, limit: number, cursor?: string): Promise<Page<Correction>>;
  findUnprocessed(limit: number): Promise<Correction[]>; // not yet folded into a Learning Batch
  save(correction: Correction): Promise<void>; // insert-only, no update method exposed
}

interface RuleWeightProfileRepository {
  findByTenant(tenantId: TenantId | 'global-default'): Promise<RuleWeightProfile | null>;
  findByIdAtVersion(profileId: ProfileId, version: number): Promise<RuleWeightProfile | null>; // audit/rollback reads
  save(profile: RuleWeightProfile): Promise<void>; // must check version for optimistic concurrency
}

interface FewShotExampleSetRepository {
  findByCategory(category: InternalCategory): Promise<FewShotExampleSet | null>;
  save(set: FewShotExampleSet): Promise<void>;
}
```

## 7. Domain Services

- **`FeedbackAggregationService`**: consumes unprocessed `Correction`s, normalizes each into one or more `FeedbackSignal`s (a Correction can implicate multiple signals if several rule signals contributed to the wrong verdict), and assembles a `LearningBatch` once volume/time thresholds are met (per research report §7 Stage 6: async, not per-correction).
- **`WeightAdjustmentService`**: given a `LearningBatch`, computes candidate `WeightDelta`s per `(category, signal)` respecting the minimum-sample-size and bounded-delta invariants, then applies them to the tenant's `RuleWeightProfile` (or the `global-default` if the tenant has not forked one).
- **`FewShotCurationService`**: selects which corrected messages are distinctive enough to become new `FewShotExample`s (favoring corrections where `originalDecisionSource = LlmAdjudication`, since those are precisely the boundary cases the prompt needs to learn from) and runs the bounded-pool eviction policy.
- **`CalibrationService`**: tracks observed false-positive/false-negative rates per category from Corrections and proposes `CalibrationThresholdUpdated` events when a category's rate drifts outside an acceptable band — this is what tunes how aggressively Classification Engine finalizes without LLM escalation.

## 8. Anti-Corruption Layer / Integration Adapters

This context has no direct external (Gmail/Graph) API dependency — it never talks to a provider. Its only adapter translates a *cross-context* signal into its own model:

```typescript
// Translates Action & Provider Sync's reconciliation drift signal into an Implicit Correction,
// so this context's Correction aggregate never has to understand provider-native diff shapes.
interface ProviderDivergenceTranslator {
  translate(drift: ReconciliationDriftReport): RecordCorrectionCommand[];
}

interface ReconciliationDriftReport {
  mailboxId: MailboxId;
  messageId: MessageId;
  systemAppliedCategories: InternalCategory[];
  providerObservedCategories: InternalCategory[]; // what a human actually left applied
  detectedAt: Date;
}
```

- **`ProviderDivergenceTranslator`** consumes [[06-action-provider-sync]]'s `ReconciliationSweepCompleted`-adjacent drift data and produces `RecordCorrection` commands with `correctionSource: 'ProviderDivergence'` — this is how a user directly editing Gmail labels (bypassing the product UI entirely) still feeds the learning loop.

## 9. Relationships to Other Bounded Contexts

| Context | Relationship | Notes |
|---|---|---|
| **3. Classification Engine** | Supplier via Published Language (this context is Supplier) | Classification Engine is a Conformist consumer of `LearningAdjustmentPublished` / `RuleWeightAdjusted` events — it applies the new `RuleWeightProfile` version as given, without renegotiating the weight-adjustment algorithm. |
| **4. LLM Adjudication** | Supplier via Published Language (this context is Supplier) | LLM Adjudication conforms to `FewShotExampleAdded`/`Evicted` events to refresh its cached prompt's example block; it does not curate examples itself. |
| **6. Action & Provider Sync** | Customer–Supplier (this context is Customer) | Consumes reconciliation drift reports as a source of Implicit Corrections via the `ProviderDivergenceTranslator` ACL above. |
| **9. Audit & Compliance** | Partnership | Every `CorrectionRecorded` event is also durably audited (who/what/when) since a correction is itself a decision-relevant fact; both contexts co-evolve the correction-chain audit trail. |
| **10. Notification & Reporting** | Supplier (this context is Supplier) | Publishes correction volume and accuracy-trend-relevant data (false-positive/negative rates from `CalibrationService`) consumed by dashboards. |
| **8. Tenant & Account Management** | Conformist | Reads `tenantId` scoping only — respects per-tenant `RuleWeightProfile` forking but does not alter tenant/plan lifecycle. |
| **1. Identity & Provider Access** | none direct | No dependency — this context never calls a provider API. |
| **2. Mail Ingestion** | none direct | No dependency — operates purely on already-classified messages and their verdict history. |
| **5. Relationship Graph** | none direct | VIP/reciprocity scores are inputs to Classification Engine, not to this context; corrections about `priority_vip` misclassifications still flow through the normal Correction → Classification Engine path rather than a direct link. |
