# Bounded Context 03: Classification Engine

## Purpose

Owns the deterministic, cheap, rule-based scoring layer that runs against every normalized `RawMessage` (from Context 02): header parsing, sender-domain allowlist lookups, schema.org JSON-LD parsing, iCalendar `METHOD` read-through, native-category read-through, and the SpamAssassin/Rspamd-style additive weighted-rule model (research §4, §7 Stage 2-3). Produces a per-category confidence score and either finalizes a verdict outright or flags the message as ambiguous for escalation to Context 04 (LLM Adjudication).

---

## 1. Ubiquitous Language Glossary

| Term | Definition |
|---|---|
| **MessageClassification** | The aggregate root: the complete classification record for one message, holding per-category scores, the current verdict set, and the decision trail. |
| **CategoryScore** | A value object: one category's accumulated signal score, its confidence tier, and whether it has crossed the finalization threshold. |
| **Signal** | A value object representing one atomic piece of evidence extracted from a message (e.g., "has List-Unsubscribe header", "sender domain matches ESP allowlist") with a name, a boolean/weighted contribution, and provenance. |
| **Rule** | An entity: a named, versioned, weighted predicate over a `RawMessage`'s signals that contributes a signed score to one or more `CategoryScore`s — the SpamAssassin/Rspamd "symbol" concept (research §6). |
| **RuleSet** | A value object: the versioned collection of active `Rule`s for a given category, allowing weight tuning/A-B rollout without redeploying code. |
| **Category** | The 11-value enum from the unified taxonomy: `NEWSLETTER`, `JOB_POSTING`, `SOCIAL`, `ECOMMERCE_RECEIPT`, `PROMO_DEAL`, `LINKEDIN`, `MEETING_CANCELLED`, `NEEDS_REPLY`, `PRIORITY_VIP`, `PHISHING`, `PERSONAL`. |
| **ConfidenceTier** | `HIGH` (finalized without escalation), `MEDIUM` (ambiguous, escalation candidate), `LOW` (insufficient signal, defaults per category policy). |
| **DecisionSource** | Records who/what finalized a category verdict: `RULE_ENGINE` \| `LLM_ADJUDICATION` \| `NATIVE_PLATFORM_PRIOR` \| `USER_OVERRIDE`. |
| **SenderDomainAllowlist** | An entity: a maintained, versioned list of known sending domains per category-relevant grouping (ESPs, ATS platforms, social platforms, LinkedIn) — the "no canonical current list, needs empirical collection" artifact from research §4/§9. |
| **StructuralMarker** | A value object capturing a parsed structural fact already lifted out of MIME by Context 02 (`isCalendarCancelMethod`) or parsed here (schema.org `Order`/`ParcelDelivery` JSON-LD block, `Authentication-Results` SPF/DKIM/DMARC verdict). |
| **ClassificationRun** | An entity recording one execution of the rule engine against a `RawMessage` — supports re-running with an updated `RuleSet` version without losing history (research §7 Stage 6: "which layer decided it, for auditability/retraining"). |

---

## 2. Aggregate Root: `MessageClassification`

### Identity
- `messageIdentity: MessageIdentity` (same value object as Context 02's `RawMessage` — this aggregate is 1:1 with a `RawMessage` but owned independently, since classification has its own lifecycle/versioning)

### Fields
- `connectionId: ConnectionId`
- `categoryScores: Map<Category, CategoryScore>`
- `finalizedCategories: Category[]` (the subset of `categoryScores` that have reached a terminal verdict — a message may finalize into 0-N categories, per research §3's "0-N categories" model)
- `escalatedCategories: Category[]` (categories in `MEDIUM` tier awaiting/consumed by LLM adjudication)
- `structuralMarkers: StructuralMarker[]`
- `nativeCategoryHintsApplied: NativeCategoryHint[]` (copied read-through from `RawMessage`, retained here for scoring provenance)
- `ruleSetVersion: string` (which `RuleSet` version produced the current scores — required for retraining/audit per research §7 Stage 6)
- `latestRunId: ClassificationRunId`
- `status: ClassificationStatus` (`PENDING` | `RULE_SCORED` | `AWAITING_ADJUDICATION` | `FINALIZED` | `RESCORE_REQUIRED`)
- `createdAt: DateTime`, `updatedAt: DateTime`

### Value Objects
- `CategoryScore { category: Category, rawScore: number, confidenceTier: ConfidenceTier, contributingSignals: Signal[], decisionSource: DecisionSource | null, finalizedAt: DateTime | null }`
- `Signal { name: string, ruleId: string, weight: number, matched: boolean, evidence: string }` — `evidence` is a short human-readable string (e.g., `"List-Unsubscribe header present"`, `"sender domain lever.co matched ATS allowlist"`) for auditability, never raw body content, to keep this aggregate cheap to store/log per the content-minimization principle
- `StructuralMarker { type: 'JSONLD_ORDER' | 'JSONLD_PARCEL_DELIVERY' | 'ICAL_METHOD_CANCEL' | 'AUTH_RESULTS_VERDICT' | 'LIST_UNSUBSCRIBE_PRESENT' | 'PRECEDENCE_BULK' | 'AUTO_SUBMITTED', detail: string }`
- `NativeCategoryHint` (shared shape with Context 02)

### Entity: `ClassificationRun` (child, referenced by ID, append-only history)
- `runId: ClassificationRunId`
- `messageIdentity: MessageIdentity`
- `ruleSetVersion: string`
- `signalsEvaluated: Signal[]`
- `resultingScores: CategoryScore[]`
- `triggeredBy: RunTrigger` (`INITIAL_INGESTION` | `RULESET_UPDATED_RESCORE` | `USER_CORRECTION_RESCORE`)
- `executedAt: DateTime`
- `durationMs: number`

### Entity: `SenderDomainAllowlist` (separate small aggregate, referenced by the scoring service, not embedded in `MessageClassification`)
- `allowlistId: string`
- `groupName: string` (`ESP_NEWSLETTER`, `ATS_JOB_POSTING`, `SOCIAL_PLATFORM`, `LINKEDIN`, `ECOMMERCE_KNOWN_RETAILER`)
- `domains: string[]`
- `version: number`
- `lastUpdatedAt: DateTime`
- `source: 'CURATED' | 'EMPIRICAL_COLLECTION'` (flags the research-identified gap: ATS domains especially are empirically collected, not a canonical spec — research §9 item 1)

### Invariants

1. **A category can only be `FINALIZED` by `RULE_ENGINE` if its score crosses the category's configured high-confidence threshold.** Each `Category` has an independently configured threshold (SpamAssassin/Rspamd additive-scoring pattern, research §6) — crossing it sets `confidenceTier = HIGH` and `decisionSource = RULE_ENGINE`; the aggregate must reject a `FinalizeCategoryFromRules` command where `rawScore` is below that category's threshold.
2. **Mutually-exclusive category conflicts must be resolved deterministically, never left ambiguous in `FINALIZED` state.** Some categories are logically exclusive in practice per research (e.g., a message cannot be both `MEETING_CANCELLED` — driven by a hard iCalendar METHOD:CANCEL structural fact — and any other category as its *primary* disposition), while others legitimately co-occur (`NEWSLETTER` + `PROMO_DEAL`). The aggregate enforces a configured exclusivity matrix; attempting to finalize a category that conflicts with an already-finalized exclusive category must fail loudly (`CategoryConflictError`), forcing rescoring rather than silently picking one.
3. **`MEETING_CANCELLED` finalizes purely from the structural marker, never from weighted scoring.** Per research §4 ("essentially none" — solved at MIME parse level), if `structuralMarkers` contains `ICAL_METHOD_CANCEL`, the category must finalize at `HIGH` confidence immediately, bypassing the additive-weight accumulation path entirely — this is enforced as a short-circuit rule with priority 0 in the `RuleSet`, and the aggregate must not allow this category to sit in `MEDIUM`/escalated state.
4. **Escalation to `AWAITING_ADJUDICATION` is only permitted for categories whose rule-based signals leave them at `MEDIUM` confidence** — a `LOW`-confidence category with no meaningful signal at all defaults per category policy (e.g., absence of evidence for `PHISHING` defaults to "not phishing," not "ambiguous, ask the LLM," to control LLM cost per research §5). The aggregate must reject an escalation request for a category still at `LOW` unless a policy flag explicitly marks that category as "escalate-on-silence" (reserved for `NEEDS_REPLY`, which per research §4 is "the single most LLM-dependent category").
5. **`PHISHING` verdicts from the rule engine alone are never sufficient to finalize at `HIGH` confidence purely from SPF/DKIM/DMARC pass.** Per research §4, auth-results pass is "necessary but insufficient" (does not catch lookalike/cousin domains that pass all three checks) — the aggregate must require *both* an auth-results signal *and* a lookalike-domain-check signal to be favorable before `PHISHING` can finalize as "not phishing" at `HIGH` confidence from rules alone; any lookalike-domain match, regardless of auth-results, forces at minimum `MEDIUM` (escalation-eligible) and typically drives a `HIGH`-confidence positive `PHISHING` finalization directly from rules (a positive match is stronger evidence than a clean pass is of safety).
6. **Rescoring on `RuleSet` version change must preserve prior `ClassificationRun` history, never overwrite it.** Invariant supporting research §7 Stage 6 auditability — a new `ClassificationRun` entity is appended, and `latestRunId`/`categoryScores` are updated to point to it, but old runs remain queryable.
7. **A `FINALIZED` status is only reachable once every applicable category (i.e., every category not excluded by the exclusivity matrix) has either a `RULE_ENGINE`, `LLM_ADJUDICATION`, `NATIVE_PLATFORM_PRIOR`, or default-`LOW`-confidence-closed verdict** — the aggregate cannot report itself `FINALIZED` while any category remains `AWAITING_ADJUDICATION`.

---

## 3. Domain Events

| Event | Payload |
|---|---|
| `ClassificationRunStarted` | `{ runId, messageIdentity, ruleSetVersion, triggeredBy, startedAt }` |
| `CategoryScoreUpdated` | `{ messageIdentity, category, rawScore, confidenceTier, contributingSignalNames: string[], occurredAt }` |
| `CategoryFinalizedByRules` | `{ messageIdentity, category, rawScore, decisionSource: 'RULE_ENGINE', occurredAt }` |
| `CategoryFinalizedByNativePlatformPrior` | `{ messageIdentity, category, nativeHintSource, occurredAt }` (e.g., a native-category read-through alone was sufficient for a low-stakes category) |
| `CategoryEscalatedForAdjudication` | `{ messageIdentity, category, rawScore, contributingSignalNames: string[], escalatedAt }` — this is the primary event Context 04 subscribes to |
| `CategoryConflictDetected` | `{ messageIdentity, conflictingCategories: Category[], occurredAt }` |
| `MessageClassificationFinalized` | `{ messageIdentity, finalizedCategories: Category[], decisionSources: Record<Category, DecisionSource>, occurredAt }` — the primary event Context 06 (Action & Provider Sync) subscribes to |
| `SenderDomainAllowlistUpdated` | `{ allowlistId, groupName, addedDomains: string[], removedDomains: string[], newVersion, updatedAt }` |
| `RescoreRequired` | `{ messageIdentity, reason: 'RULESET_UPDATED' \| 'USER_CORRECTION', requestedAt }` |

---

## 4. Commands

| Command | Description |
|---|---|
| `RunClassification` | Triggered by `MessageHandedOffForClassification` (Context 02); executes the full `RuleSet` against a `RawMessage`'s signals, producing a new `ClassificationRun`. |
| `FinalizeCategoryFromRules` | Commits a `HIGH`-confidence category verdict once threshold is crossed. |
| `FinalizeCategoryFromNativePrior` | Commits a verdict for a category where the native platform classifier's signal alone is policy-sufficient (used sparingly, e.g., `SOCIAL` when `CATEGORY_SOCIAL` is present and no conflicting signal exists). |
| `EscalateCategoryForAdjudication` | Marks a `MEDIUM`-confidence category as `AWAITING_ADJUDICATION` and publishes `CategoryEscalatedForAdjudication`. |
| `ApplyAdjudicationVerdict` | Consumes a verdict handed back from Context 04 and finalizes the category with `decisionSource = LLM_ADJUDICATION` (invoked by Context 04, not self-initiated). |
| `RequestBodyFetchForSignal` | Issues `FetchMessageBody` (Context 02) when a specific rule requires body content it doesn't yet have (e.g., promotional-lexicon scoring) — gated so this only happens for messages already past cheap header-only elimination. |
| `RescoreMessage` | Re-runs classification against an updated `RuleSet` or after a user correction feeds back adjusted weights (from Context 07). |
| `UpdateSenderDomainAllowlist` | Adds/removes domains from a maintained allowlist (empirical-collection workflow, research §9 item 1). |
| `PublishRuleSetVersion` | Activates a new versioned `RuleSet` for a category (supports controlled rollout/weight tuning per research §9 item 5). |

---

## 5. Repository Interfaces

```typescript
interface MessageClassificationRepository {
  findByMessageIdentity(identity: MessageIdentity): Promise<MessageClassification | null>;

  findAwaitingAdjudication(limit: number): Promise<MessageClassification[]>;
  // feeds Context 04's batch-pull for the escalation queue

  findByFinalizedCategory(
    connectionId: ConnectionId,
    category: Category,
    since: DateTime
  ): Promise<MessageClassification[]>;

  findRequiringRescore(ruleSetVersion: string, limit: number): Promise<MessageClassification[]>;

  save(classification: MessageClassification): Promise<void>;
}

interface ClassificationRunRepository {
  findByMessageIdentity(identity: MessageIdentity): Promise<ClassificationRun[]>; // full history
  findLatest(identity: MessageIdentity): Promise<ClassificationRun | null>;
  save(run: ClassificationRun): Promise<void>; // append-only
}

interface SenderDomainAllowlistRepository {
  findByGroup(groupName: string): Promise<SenderDomainAllowlist | null>;
  findAll(): Promise<SenderDomainAllowlist[]>;
  save(allowlist: SenderDomainAllowlist): Promise<void>;
}

interface RuleSetRepository {
  findActiveVersion(category: Category): Promise<RuleSet>;
  findVersion(category: Category, version: string): Promise<RuleSet | null>;
  save(ruleSet: RuleSet): Promise<void>;
}
```

---

## 6. Domain Services

### `SignalExtractionService`
Spans a `RawMessage` (read from Context 02) and produces the raw `Signal[]` inputs to scoring — header parsing (`List-Unsubscribe`, `List-Id`, `Precedence`, `Auto-Submitted`, `Authentication-Results`), schema.org JSON-LD extraction, sender-domain allowlist lookups, and native-category read-through. This is intentionally a domain service rather than aggregate behavior because it reads across the `RawMessage` (foreign context) and `SenderDomainAllowlist` (sibling aggregate).

```typescript
interface SignalExtractionService {
  extractSignals(message: RawMessageView, allowlists: SenderDomainAllowlist[]): Signal[];
}
```

### `AdditiveScoringService`
The SpamAssassin/Rspamd-style engine: applies a `RuleSet`'s weighted rules against extracted signals to produce `CategoryScore`s, independently per category (research §6).

```typescript
interface AdditiveScoringService {
  score(signals: Signal[], ruleSet: RuleSet): CategoryScore;
}
```

### `CategoryExclusivityService`
Spans multiple `CategoryScore`s within one `MessageClassification` to enforce invariant #2 — resolves conflicts per a configured exclusivity matrix before allowing finalization.

```typescript
interface CategoryExclusivityService {
  checkConflicts(categoryScores: Map<Category, CategoryScore>): CategoryConflict[];
}
```

### `LookalikeDomainDetectionService`
Spans a message's sender domain against the user's known-contact-domain list (from Context 05) and a maintained brand-domain list, computing edit-distance-based lookalike risk — feeds invariant #5's phishing gate.

```typescript
interface LookalikeDomainDetectionService {
  assessLookalikeRisk(senderDomain: string, knownDomains: string[], brandDomains: string[]): LookalikeAssessment;
}
```

---

## 7. Anti-Corruption Layer / Integration Adapters

This context's primary external dependency is not a live remote API but *parsing untrusted, provider-normalized structured content* (JSON-LD, iCalendar) embedded in message bodies — the ACL here isolates the domain's `StructuralMarker` model from third-party markup formats' actual grammar.

```typescript
interface StructuredContentParserPort {
  parseJsonLd(bodyHtml: string): JsonLdBlock[];
  // isolates schema.org Order/ParcelDelivery vocabulary drift from the domain's StructuralMarker shape

  parseICalendar(icsContent: string): ICalendarParseResult;
  // isolates RFC 5546 (iTIP) / RFC 6047 (iMIP) grammar from the domain

  parseAuthenticationResults(headerValue: string): AuthResultsVerdict;
  // isolates RFC 8601 Authentication-Results header grammar variance across sending MTAs
}

interface JsonLdBlock { type: 'Order' | 'ParcelDelivery' | 'Other'; raw: Record<string, unknown> }
interface ICalendarParseResult { method: string | null; isCancel: boolean }
interface AuthResultsVerdict { spf: 'pass' | 'fail' | 'none'; dkim: 'pass' | 'fail' | 'none'; dmarc: 'pass' | 'fail' | 'none' }
```

```typescript
class SchemaOrgJsonLdAdapter implements Pick<StructuredContentParserPort, 'parseJsonLd'> { /* ... */ }
class ICalendarAdapter implements Pick<StructuredContentParserPort, 'parseICalendar'> { /* ... */ }
class AuthResultsHeaderAdapter implements Pick<StructuredContentParserPort, 'parseAuthenticationResults'> { /* ... */ }
```

Note: this context does **not** talk to Gmail/Graph/LLM APIs directly — it consumes already-normalized `RawMessage` data from Context 02 and hands escalation candidates to Context 04. No provider ACL is owned here.

---

## 8. Relationships to Other Bounded Contexts

- **Mail Ingestion (Context 02) — Customer-Supplier, Context 02 upstream.** This context consumes `MessageHandedOffForClassification` and the normalized `RawMessage` read model as its published language; it issues `RequestBodyFetchForSignal` back to Context 02 when a rule needs body content, making Context 02 responsive to this context's on-demand requests within the content-minimization contract.
- **LLM Adjudication (Context 04) — Partnership.** The two contexts co-evolve tightly: this context defines what "ambiguous" means (`MEDIUM` tier, escalation policy) and Context 04 defines the adjudication contract; `CategoryScore`/`Signal` shapes are shared vocabulary passed directly into LLM prompts, so a change to signal naming here requires coordinated prompt/few-shot updates in Context 04 — a Partnership, not a one-directional Open Host relationship.
- **Relationship Graph (Context 05) — Customer-Supplier, Context 05 upstream.** This context is a *customer* of Context 05's known-contact-domain list (for `LookalikeDomainDetectionService`) and reciprocity signals (feeding `PRIORITY_VIP` scoring as one more rule input) but never mutates Context 05's data.
- **Action & Provider Sync (Context 06) — Open Host Service / Published Language, this context upstream.** `MessageClassificationFinalized` is the primary event Context 06 subscribes to in order to translate finalized categories into provider labels/categories.
- **Feedback & Learning (Context 07) — Customer-Supplier, Context 07 upstream for weight updates.** Context 07 captures user corrections and is the *supplier* of updated `RuleSet` weights/thresholds via `PublishRuleSetVersion`; this context is the *customer* consuming those updates and re-scoring (`RescoreMessage`) accordingly. Conversely, this context supplies `ClassificationRun` history to Context 07 as training/evaluation data — a bidirectional Partnership in practice, but the rule-weight-authority direction is Customer-Supplier with Context 07 as supplier.
- **Audit & Compliance (Context 09) — Customer-Supplier, this context upstream.** Every `CategoryFinalizedByRules`/`CategoryFinalizedByNativePlatformPrior`/conflict event feeds the audit trail (decision-source auditability, research §7 Stage 6).
- **Notification & Reporting (Context 10) — Conformist.** Consumes `MessageClassificationFinalized` (particularly `PHISHING` verdicts) as-is for alerting, without a negotiated custom contract.
