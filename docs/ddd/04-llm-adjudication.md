# Bounded Context 04: LLM Adjudication

## Purpose

Owns the escalation path for messages/categories the Classification Engine (Context 03) leaves at `MEDIUM` confidence: batching ambiguous items into structured multi-item LLM calls, managing prompt-cache lifecycle for the fixed taxonomy/few-shot system prompt, tracking cost and latency per call, and merging the resulting verdicts back into `MessageClassification` (research §5, §7 Stage 4). This context is the sole owner of the LLM-provider relationship (Claude API) and the sole place model-cost economics are reasoned about.

---

## 1. Ubiquitous Language Glossary

| Term | Definition |
|---|---|
| **AdjudicationBatch** | The aggregate root: a group of ambiguous category-escalations bundled into one LLM call (or one Batch API job), sharing a cached system prompt. |
| **AdjudicationItem** | An entity within a batch: one message's ambiguous category (or set of categories) submitted for LLM judgment, plus the resulting verdict once returned. |
| **AdjudicationVerdict** | A value object: the LLM's structured output for one item — categories assigned, confidence, and a short rationale string. |
| **PromptCacheHandle** | A value object referencing an active prompt-cache entry (the taxonomy + few-shot system prompt) with its cache-hit/miss economics tracked. |
| **CallMode** | `SYNCHRONOUS` (time-sensitive, e.g., real-time nudge) or `BATCH_API` (backlog sweep, 50% discount, higher latency tolerance) — research §5. |
| **ModelTier** | Which Claude model served a given call: `HAIKU_4_5` (default), `SONNET_5`, `OPUS_5` — research §5 pricing table. |
| **CostLedgerEntry** | A value object recording the metered cost of one LLM call: input/output tokens, cache-read vs cache-write tokens, model tier, computed USD cost. |
| **ContentMinimizationDecision** | A value object recording whether an item was submitted to the LLM with metadata-only or full-body content, and why — the auditable trace of the "only escalate to body content for the minority that need it" policy (research §5, §8). |
| **AdjudicationFallbackPolicy** | A value object defining what happens when the LLM call fails or returns low-confidence output for an item — research §9 item 6's identified open question, resolved here as an explicit domain concept rather than left implicit. |

---

## 2. Aggregate Root: `AdjudicationBatch`

### Identity
- `batchId: AdjudicationBatchId`

### Fields
- `callMode: CallMode`
- `modelTier: ModelTier`
- `promptCacheHandle: PromptCacheHandle`
- `items: AdjudicationItem[]` (child entities)
- `status: BatchStatus` (`ASSEMBLING` | `SUBMITTED` | `AWAITING_BATCH_RESULT` | `COMPLETED` | `PARTIALLY_FAILED` | `FAILED`)
- `submittedAt: DateTime | null`
- `completedAt: DateTime | null`
- `providerBatchJobId: string | null` (Anthropic Batch API job ID, when `callMode == BATCH_API`)
- `costLedgerEntry: CostLedgerEntry | null` (populated on completion)
- `createdAt: DateTime`

### Entity: `AdjudicationItem`
- `itemId: AdjudicationItemId`
- `messageIdentity: MessageIdentity`
- `escalatedCategories: Category[]` (may be more than one category escalated together for the same message, to minimize per-message call overhead)
- `contentMinimizationDecision: ContentMinimizationDecision`
- `submittedSignals: Signal[]` (the rule-engine signals already known, passed as context to reduce LLM guesswork and token count)
- `submittedContent: AdjudicationInputContent` (metadata-only fields, or metadata + body per the minimization decision)
- `verdict: AdjudicationVerdict | null`
- `itemStatus: ItemStatus` (`PENDING` | `VERDICT_RECEIVED` | `FAILED` | `FALLBACK_APPLIED`)

### Value Objects
- `PromptCacheHandle { cacheKey: string, systemPromptVersion: string, expiresAt: DateTime, cacheReadCount: number }`
- `AdjudicationVerdict { categories: Category[], confidence: 'HIGH' | 'MEDIUM' | 'LOW', rationale: string }`
- `ContentMinimizationDecision { level: 'METADATA_ONLY' | 'METADATA_PLUS_BODY', justification: string }`
- `CostLedgerEntry { inputTokens: number, outputTokens: number, cacheReadTokens: number, cacheWriteTokens: number, modelTier: ModelTier, computedCostUsd: number }`
- `AdjudicationInputContent { headers: NormalizedHeader[], structuralMarkers: StructuralMarker[], bodyExcerpt: string | null }`
- `AdjudicationFallbackPolicy { onLlmFailure: 'DEFER_TO_RULE_ENGINE_SCORE' | 'REQUIRE_MANUAL_REVIEW' | 'DEFAULT_LOW_CONFIDENCE'; onLowConfidenceVerdict: 'ACCEPT_ANYWAY' | 'REQUIRE_MANUAL_REVIEW' }` (research §9 item 6 — this domain models the decision explicitly rather than leaving it as an operational ambiguity)

### Invariants

1. **A batch's items must all share the same `PromptCacheHandle`.** Items are only added to an `ASSEMBLING` batch if `promptCacheHandle.systemPromptVersion` matches the batch's — this is the structural enforcement of "prompt-cache the fixed system prompt + few-shot examples, batch multiple emails' content into one request" (research §5); mixing prompt versions in one batch would defeat cache-hit economics and is rejected outright.
2. **`BATCH_API` call mode is only valid for non-time-sensitive items.** An `AdjudicationItem` whose message is flagged time-sensitive (a policy input from the escalating category — e.g., a real-time "needs reply" nudge) cannot be added to a `BATCH_API`-mode batch; it must go into a `SYNCHRONOUS`-mode batch instead. The aggregate rejects `AddItemToBatch` when this mismatch is detected.
3. **Content minimization must be justified per item, never defaulted to full-body silently.** `contentMinimizationDecision.level == METADATA_PLUS_BODY` is only settable when `justification` references a specific unresolved ambiguity (e.g., `"promo_vs_newsletter_lexicon_ambiguous"`, `"needs_reply_interrogative_check"`, `"phishing_residual_after_lookalike_clear"`) — this is the domain-level enforcement of research §5's "only send full body content... for minority that... land in ambiguous bucket." An item requesting body content without a recognized justification code is rejected.
4. **A batch cannot transition to `COMPLETED` while any item is still `PENDING`.** Partial results (some items succeeded, some failed) transition the batch to `PARTIALLY_FAILED`, never silently to `COMPLETED` — ensures failed items are never mistaken for successful, low-risk verdicts downstream.
5. **Every `AdjudicationItem` reaching a terminal state must have exactly one of: a `verdict`, or a `FALLBACK_APPLIED` status with a recorded fallback action.** A `FAILED` item without an applied fallback cannot be considered terminal — the aggregate enforces that `AdjudicationFallbackPolicy` is consulted and its outcome recorded (`FALLBACK_APPLIED`) before the item is excluded from retry consideration. This directly resolves research §9 item 6 by making "what happens on failure" a first-class, always-populated field rather than an unhandled edge case.
6. **Cost ledger entries are immutable once written.** `costLedgerEntry` is set exactly once, at batch completion, from the provider's actual usage response (never estimated/back-filled) — required for accurate cost tracking per research §5's cost-optimization goals.
7. **A `LOW`-confidence verdict cannot silently finalize a category as `HIGH` confidence.** If `verdict.confidence == LOW`, the item's outcome is governed by `AdjudicationFallbackPolicy.onLowConfidenceVerdict`; the aggregate must not let a low-confidence LLM answer masquerade as a decisive rule-engine-equivalent verdict when merged back into Context 03.

---

## 3. Domain Events

| Event | Payload |
|---|---|
| `AdjudicationBatchAssembled` | `{ batchId, callMode, modelTier, itemCount, promptCacheHandle, assembledAt }` |
| `AdjudicationBatchSubmitted` | `{ batchId, providerBatchJobId, submittedAt }` |
| `AdjudicationItemVerdictReceived` | `{ batchId, itemId, messageIdentity, verdict, occurredAt }` — consumed by Context 03 via `ApplyAdjudicationVerdict` |
| `AdjudicationItemFailed` | `{ batchId, itemId, messageIdentity, providerErrorCode, occurredAt }` |
| `AdjudicationFallbackApplied` | `{ batchId, itemId, messageIdentity, fallbackAction, occurredAt }` |
| `AdjudicationBatchCompleted` | `{ batchId, itemCount, succeededCount, failedCount, costLedgerEntry, completedAt }` |
| `PromptCacheRefreshed` | `{ cacheKey, systemPromptVersion, refreshedAt }` |
| `AdjudicationCostRecorded` | `{ batchId, computedCostUsd, modelTier, callMode, occurredAt }` — feeds cost dashboards/budget alerts (Context 10) |

---

## 4. Commands

| Command | Description |
|---|---|
| `AssembleAdjudicationBatch` | Creates a new `ASSEMBLING` batch for a given `callMode`/`modelTier`/prompt-cache version. |
| `AddItemToBatch` | Adds an `AdjudicationItem` (from a `CategoryEscalatedForAdjudication` event) to an assembling batch, enforcing invariants #1-#3. |
| `SubmitBatch` | Closes assembly and sends the batch to the Claude API (sync call or Batch API job submission). |
| `PollBatchJobStatus` | For `BATCH_API` mode, checks the Anthropic Batch API job status until results are ready. |
| `RecordItemVerdict` | Applies a returned verdict to an `AdjudicationItem`, transitioning it to `VERDICT_RECEIVED`. |
| `ApplyFallbackPolicy` | Invoked when an item fails or returns low confidence; consults `AdjudicationFallbackPolicy` and records the outcome. |
| `CompleteBatch` | Finalizes the batch once all items are terminal, computing and recording `CostLedgerEntry`. |
| `RefreshPromptCache` | Rotates to a new `systemPromptVersion` (e.g., taxonomy or few-shot examples updated by Context 07's retraining loop). |
| `RequestBodyContentForItem` | Issues the `FetchMessageBody` request (via Context 02, brokered through Context 03) when assembling an item that needs `METADATA_PLUS_BODY`. |

---

## 5. Repository Interface

```typescript
interface AdjudicationBatchRepository {
  findById(batchId: AdjudicationBatchId): Promise<AdjudicationBatch | null>;

  findAssembling(callMode: CallMode, modelTier: ModelTier): Promise<AdjudicationBatch | null>;
  // the currently-open batch new items get appended to, before a size/time threshold triggers submission

  findAwaitingBatchResult(): Promise<AdjudicationBatch[]>;
  // feeds the Batch API polling job

  findByProviderBatchJobId(providerBatchJobId: string): Promise<AdjudicationBatch | null>;

  findItemByMessageIdentity(identity: MessageIdentity): Promise<AdjudicationItem | null>;

  save(batch: AdjudicationBatch): Promise<void>;

  sumCostByDateRange(from: DateTime, to: DateTime): Promise<{ totalCostUsd: number; callCount: number }>;
  // supports cost dashboards / budget alerting
}

interface PromptCacheRepository {
  findActiveHandle(systemPromptVersion: string): Promise<PromptCacheHandle | null>;
  save(handle: PromptCacheHandle): Promise<void>;
}
```

---

## 6. Domain Services

### `BatchAssemblyService`
Spans incoming escalation events and open `AdjudicationBatch` aggregates to decide which batch a new item joins (or whether a new batch must be assembled), balancing the research-cited "1.2-1.9x median per-item speedup" from multi-item batching against latency requirements for `SYNCHRONOUS` mode.

```typescript
interface BatchAssemblyService {
  assignItemToBatch(escalation: EscalationRequest): Promise<AdjudicationBatchId>;
  shouldTriggerSubmission(batch: AdjudicationBatch): boolean;
  // true when size threshold, time-in-assembly threshold, or explicit flush is reached
}
```

### `ContentMinimizationPolicyService`
Spans an `AdjudicationItem`'s escalated category + rule-engine signals to decide the minimum content level needed (invariant #3), implementing the "8 of 11 categories resolvable without body" principle at the per-item level for the residual ambiguous minority.

```typescript
interface ContentMinimizationPolicyService {
  decide(category: Category, signals: Signal[]): ContentMinimizationDecision;
}
```

### `CostAccountingService`
Spans the provider's raw usage response and the internal `CostLedgerEntry` model, applying current Claude pricing (Haiku 4.5 $1/$5 per MTok, cache reads at 10% of base input, Batch API 50% discount stacking with caching — research §5).

```typescript
interface CostAccountingService {
  computeCost(usage: ProviderUsageReport, modelTier: ModelTier, callMode: CallMode): CostLedgerEntry;
}
```

### `AdjudicationFallbackService`
Spans a failed/low-confidence item and the configured `AdjudicationFallbackPolicy` to determine and apply the resolved fallback action (invariant #5, resolving research §9 item 6).

```typescript
interface AdjudicationFallbackService {
  resolveFallback(item: AdjudicationItem, policy: AdjudicationFallbackPolicy): FallbackAction;
}
```

---

## 7. Anti-Corruption Layer / Integration Adapters

```typescript
interface LlmProviderPort {
  submitSynchronousCall(request: AdjudicationCallRequest): Promise<AdjudicationCallResponse>;

  submitBatchJob(requests: AdjudicationCallRequest[]): Promise<{ providerBatchJobId: string }>;

  pollBatchJob(providerBatchJobId: string): Promise<BatchJobPollResult>;

  fetchBatchResults(providerBatchJobId: string): Promise<AdjudicationCallResponse[]>;
}

interface AdjudicationCallRequest {
  systemPromptCacheKey: string;
  systemPromptContent: string; // sent once per cache-miss; the ACL manages cache_control blocks
  items: {
    itemId: string;
    messageMetadata: AdjudicationInputContent;
    signals: Signal[];
  }[]; // multiple items packed into one structured-JSON request, per research §5 batching recommendation
  modelTier: ModelTier;
}

interface AdjudicationCallResponse {
  verdicts: { itemId: string; categories: Category[]; confidence: string; rationale: string }[];
  usage: ProviderUsageReport;
}

interface ProviderUsageReport {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
}

interface BatchJobPollResult {
  status: 'in_progress' | 'ended' | 'failed' | 'canceled';
  resultsAvailable: boolean;
}
```

```typescript
// Isolates the Claude Messages API / Batch API wire format, structured-output (JSON) parsing,
// prompt-cache cache_control block placement, and model-ID selection (claude-haiku-4-5-20251001, etc.)
// from the domain — the domain only ever sees AdjudicationCallRequest/Response.
class ClaudeApiAdjudicationAdapter implements LlmProviderPort { /* ... */ }
```

---

## 8. Relationships to Other Bounded Contexts

- **Classification Engine (Context 03) — Partnership.** As noted in Context 03's own relationships section, the two contexts co-evolve the escalation contract (`Signal`/`CategoryScore` shapes feed directly into LLM prompts); this context's `ApplyAdjudicationVerdict` call-back into Context 03 closes the loop, and both contexts must agree on category-conflict semantics so an LLM verdict cannot reintroduce a conflict Context 03's exclusivity matrix would reject.
- **Mail Ingestion (Context 02) — Customer-Supplier, this context (indirectly, via Context 03) as customer.** `RequestBodyContentForItem` ultimately triggers Context 02's `FetchMessageBody`; this context never calls Context 02 directly, always brokering through Context 03 to keep a single owner of "when does a body fetch happen" policy (Context 03's `RequestBodyFetchForSignal` command already exists for this; this context reuses it rather than duplicating a second body-fetch trigger path).
- **Feedback & Learning (Context 07) — Customer-Supplier, Context 07 upstream.** Context 07 owns the few-shot example curation and system-prompt taxonomy updates based on user corrections; this context is the *customer* consuming new `systemPromptVersion`s via `RefreshPromptCache`. This context supplies `AdjudicationVerdict` + `rationale` history back to Context 07 as a signal of where the LLM's judgment and the user's eventual correction diverge.
- **Identity & Provider Access (Context 01) — no direct relationship.** This context never touches mailbox OAuth; it only calls the Claude API, which uses this system's own Anthropic API credentials, not a per-mailbox provider credential.
- **Relationship Graph (Context 05) — Customer-Supplier, Context 05 upstream.** VIP/reciprocity signals computed by Context 05 may be included in `submittedSignals` for `PRIORITY_VIP` or `NEEDS_REPLY` adjudication items, as read-only context passed into the LLM prompt.
- **Audit & Compliance (Context 09) — Customer-Supplier, this context upstream.** Every `ContentMinimizationDecision` at `METADATA_PLUS_BODY` and every `AdjudicationCostRecorded` event is supplied to Context 09 for DPA-scope tracking (this context is a data processor whenever body content is sent to the LLM, per research §8) and cost/budget audit trails.
- **Notification & Reporting (Context 10) — Conformist.** Consumes `AdjudicationCostRecorded` as-is for cost dashboards, without a negotiated contract back to this context.
