# ADR-013: User Feedback / Correction Loop & Retraining Strategy

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: feedback, retraining, rule-engine, llm, learning

## Context

Per §6 of the research report, SaneBox's most reusable UX insight is "drag one email to retrain the sender forever" — a single correction should durably change future behavior for that sender/pattern, not just the one message. Per §7 (Stage 6, State Store), the architecture explicitly calls for "user-correction feedback loop (when user manually re-labels, feed back to adjust rule weights/few-shot examples)" as a first-class responsibility of the state store, and research gap #6 (§9) explicitly leaves open "what fallback logic applies when LLM adjudication fails or returns low confidence," which this ADR must address alongside the correction loop itself.

There is no existing public reference dataset or accepted algorithm for this exact 11-category taxonomy (§4 confidence grading: "Medium-low" for needs-reply/VIP algorithms, expect empirical tuning). This ADR therefore defines the *mechanism* for capturing and applying feedback, not fixed target weights — those are tuned empirically per [[adr-008-deterministic-rule-scoring-engine]].

## Decision

### Correction capture
Every user-initiated override (re-label, un-label, move-out-of-category, manual VIP add/remove) is captured as a `ClassificationCorrection` record:

```
ClassificationCorrection {
  correction_id: uuid
  tenant_id, mailbox_id
  message_identity_id (FK to internal message identity, per ADR-007)
  original_category: string, original_source: 'rule'|'llm'|'native'
  original_confidence: float
  corrected_category: string | null   -- null = "remove all categories"
  corrected_by: 'user_explicit' | 'user_implicit' (e.g., moved out of applied label)
  contributing_symbols: string[]      -- snapshot of the rule symbols that fired at classification time
  created_at: timestamptz
}
```

Corrections are captured via two paths:
1. **Explicit**: a tenant-facing "mark as X" / "not X" action surfaced through the admin/config API ([[adr-018-tenant-admin-configuration-api]]).
2. **Implicit**: a reconciliation-sweep diff (per [[adr-005-ingestion-sync-push-reconciliation]]) detects the user removed a system-applied label/category or moved mail out of a system-created folder — logged as a lower-confidence implicit signal, not immediately actioned (implicit signals require ≥3 corroborating instances before affecting weights, to avoid overfitting to one-off manual cleanup).

### Rule-weight adjustment (fast path, no retraining cycle)
- Each `contributing_symbols` entry on a correction decrements that symbol's per-tenant weight multiplier by a fixed step (default 5%, floor 0.5x, ceiling 1.5x of the global default weight defined in [[adr-008-deterministic-rule-scoring-engine]]) for *that tenant only* — corrections never mutate global default weights, only a per-tenant override table.
- This directly implements the SaneBox "one correction changes future behavior for this sender/pattern forever" pattern: a `SENDER_DOMAIN_MATCH` symbol correction also inserts/updates a per-tenant `sender_override` row (sender domain → category, confidence 1.0, source `user_correction`) consulted *before* the general rule engine runs — i.e., a direct override cache, not just a weight nudge.
- Per-tenant weight overrides are re-applied on every subsequent classification for that tenant; they decay is NOT time-based (no silent forgetting) — an override persists until explicitly reverted or superseded by a newer correction on the same symbol/sender.

### LLM few-shot adjustment (slower path, batch cycle)
- Corrections where `original_source = 'llm'` are accumulated into a per-tenant `correction_examples` pool.
- A nightly batch job (not real-time) selects up to 20 most-recent, most-diverse corrected examples per category and rebuilds that tenant's few-shot block appended to the cached system prompt used in [[adr-009-llm-adjudication-service]]. The base system prompt/taxonomy remains globally shared and cached; only the trailing few-shot block is tenant-specific, keeping prompt-cache hit rates high (per §5 cost strategy) while still personalizing.
- No fine-tuning of the underlying LLM is performed in V1/V2 — few-shot injection only. Fine-tuning is an explicit V3+ consideration, out of scope here, contingent on volume justifying the operational cost.

### LLM/rule disagreement and low-confidence fallback (closes research gap #6)
When the LLM adjudication layer fails (timeout, malformed response, provider error) or returns confidence below a configurable floor (default 40/100):
1. Fall back to the Stage 3 rule-engine score alone, even if it fell in the "ambiguous" band — apply the category only if the rule score alone exceeds a lowered secondary threshold (default: 60% of the normal high-confidence threshold).
2. If the rule score is also inconclusive, the message is left **unclassified** (no category applied) rather than guessed — it is queued into a `needs_manual_review` state visible via the admin API, never silently dropped.
3. Every fallback event is logged with reason code (`llm_timeout`, `llm_malformed`, `llm_low_confidence`) for observability ([[adr-015-observability-slos]]) and counted toward the LLM adjudication service's error budget.

### Data retention of corrections
Correction records are retained for 24 months per tenant (configurable) to support both weight decay analysis and compliance audit trail, and are included in the tenant data-export/deletion flows defined in [[adr-014-data-retention-gdpr-ccpa-compliance]].

## Consequences

### Positive
- Directly operationalizes the report's highest-value reusable UX pattern (SaneBox one-correction-forever) without requiring a full ML retraining pipeline for V1/V2.
- Explicit, auditable fallback behavior closes a previously open research gap (§9 #6) rather than leaving LLM-failure behavior undefined in production.
- Per-tenant override table means one tenant's corrections never leak into another tenant's classification behavior — consistent with multi-tenant isolation in [[adr-003-multi-tenant-data-isolation]].

### Negative
- Per-tenant sender-override cache adds a lookup on the classification hot path for every message (mitigated by keeping it in the same cache tier as sender-domain allowlists in [[adr-008-deterministic-rule-scoring-engine]]).
- Few-shot injection is a weaker personalization mechanism than fine-tuning; may plateau in effectiveness at high correction volumes — flagged as a V3+ revisit trigger (e.g., >500 corrections/tenant/month sustained).
- Implicit-signal capture (detecting label removal) adds complexity to the reconciliation sweep and risk of false-positive "correction" inference from unrelated user mailbox reorganization — mitigated by the ≥3-corroboration rule but not eliminated.

### Neutral
- Weight-adjustment step size (5%) and confidence floor (40/100) are initial defaults pending empirical tuning against production correction volume, consistent with the report's general caution (§4, §9) that thresholds are not derivable from research alone.

## Links
- Related: ADR-001 (layered pipeline), ADR-007 (message identity/idempotency — corrections key off message identity), ADR-008 (rule/signal scoring engine — receives weight overrides), ADR-009 (LLM adjudication service — receives few-shot updates and reports low-confidence/failure events), ADR-003 (multi-tenant isolation), ADR-014 (retention/compliance), ADR-015 (observability), ADR-018 (tenant admin config API — surfaces explicit corrections)

## Verification

- Integration test: apply a correction, re-classify an identical fixture message for the same tenant, assert the corrected category now wins via the sender-override cache without waiting for a rule-weight recompute cycle.
- Integration test: simulate LLM timeout and malformed-JSON response; assert fallback path produces either a rule-only verdict or `needs_manual_review`, never an unhandled exception or silent drop.
- Batch job test: seed 20+ correction examples, run the nightly few-shot rebuild job, assert the resulting prompt stays within the token budget defined in [[adr-009-llm-adjudication-service]] and cache-hit rate on the fixed prefix remains unaffected.
- Metric: `correction_reversion_rate` (how often a user corrects a correction) tracked per tenant as a proxy for override quality; alert if sustained above 15% for any tenant (per [[adr-015-observability-slos]]).
