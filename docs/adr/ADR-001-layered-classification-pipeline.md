# ADR-001: Layered Classification Pipeline Architecture (Rules → LLM Adjudication → Action)

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: architecture, classification, pipeline, cost-control

## Context

Per §7 of the research report ("Recommended Architecture"), the system must classify every inbound message across two providers (Gmail, Microsoft Graph) into a shared 11-category taxonomy under tight cost, latency, and privacy constraints. The report explicitly recommends the Rspamd 4-stage pipeline (pre-filters → parallel main filters → post-filters → action decision) as "arguably the best architectural template for this project" (§6, Rspamd section), because it runs cheap deterministic rules first and treats ML/LLM as a *refinement* layer on named rule outputs rather than a replacement for them.

Per §4, the signal table shows that 8 of the 11 categories (newsletters, sales/deals, LinkedIn, job postings, ecommerce/receipts, meeting cancellations, personal contacts, social) are resolvable with high confidence from headers/metadata alone using RFC-standard signals (`List-Unsubscribe` RFC 2369, `Auto-Submitted` RFC 3834, iTIP/iMIP `METHOD:CANCEL` RFC 5546/6047, schema.org `Order` JSON-LD). Only "needs-reply," residual phishing, and ambiguous newsletter/promo splits genuinely require LLM judgment (§4, §5). Per §5, LLM cost/latency data (Haiku 4.5 at $1/$5 per MTok, Batch API 50% discount, prompt caching at 10% of base input cost) means routing every message through an LLM is both financially wasteful and unnecessarily privacy-invasive (§5, §8 content minimization).

A monolithic "send everything to an LLM" design would also make behavior non-auditable (no way to explain *why* a message got a category) and would not degrade gracefully if the LLM provider has an outage.

## Decision

Adopt a 5-stage synchronous-per-message pipeline, executed per `RawMessage` after provider normalization (see ADR-002):

```
Stage 1: Ingestion & Normalization   (ADR-002, ADR-005)
Stage 2: Signal Extraction            — deterministic, every message, no network egress beyond own DB/allowlist lookups
Stage 3: Rule/Scoring Engine          — additive weighted rules per category (ADR-008)
Stage 4: LLM Adjudication             — only for messages that fail to reach a high-confidence verdict in Stage 3 (ADR-009)
Stage 5: Action/Apply Layer           — provider-specific translation and idempotent write (ADR-002, ADR-007)
```

**Stage boundary contract** (each stage receives/emits a typed struct so stages are independently testable):

```typescript
interface StageInput {
  internalMessageId: string;      // ADR-007 identity hash
  raw: RawMessage;                // normalized headers + MIME + body ref
  signals?: SignalSet;            // populated after Stage 2
  scores?: CategoryScoreMap;      // populated after Stage 3
}

interface CategoryVerdict {
  category: Category;             // one of the 11 taxonomy values (ADR-010)
  confidence: number;             // 0.0-1.0
  decidedBy: 'rule' | 'llm' | 'platform_native';
  ruleIds?: string[];              // which rules fired (for audit)
}
```

**Escalation gate (Stage 3 → Stage 4)**: A message escalates to LLM adjudication if, for any candidate category, the rule engine's normalized score falls in the band `[low_threshold, high_threshold)` (see ADR-008 for exact per-category values) — i.e. neither confidently included nor confidently excluded — OR the message matches a category flagged as "LLM-required" in the taxonomy table (`needs_reply`, residual `phishing` beyond the deterministic auth-header gate). Messages fully resolved in Stage 3 (expected majority per §7: newsletters, receipts, cancellations, social, LinkedIn, most promo/deals) skip Stage 4 entirely — zero LLM spend, zero body content sent off-box for those messages (§8 content minimization).

**Fallback behavior when Stage 4 is unavailable or times out** (research gap #6 in §9, resolved here): default to the Stage 3 rule-engine's best-scoring category at `decidedBy: 'rule'` with confidence capped at 0.5, and enqueue the message for reprocessing on next reconciliation sweep (ADR-005). Never block Stage 5 indefinitely on LLM availability.

Each stage emits structured logs keyed by `internalMessageId` and stage name for observability and for the correction/feedback loop (ADR-013, batch 012-022) to attribute a wrong verdict to the specific stage/rule/prompt version that produced it.

## Consequences

### Positive
- Majority of mail volume is resolved by Stage 3 alone, minimizing LLM cost and privacy surface area (§5, §8).
- Each stage is independently unit-testable and independently swappable (e.g., rule engine can be replaced without touching LLM adjudication code).
- Audit trail (`decidedBy`, `ruleIds`) gives every classification a traceable cause, required for the feedback/retraining loop.
- Graceful degradation: LLM outage does not block core mail processing, only defers ambiguous-category refinement.

### Negative
- Five-stage pipeline is more engineering complexity than a single classifier call; requires careful interface discipline between stages.
- Threshold tuning (the escalation band) has no public reference dataset for this exact taxonomy (§9, gap #5) — will require empirical tuning against real mailbox data post-launch.
- Two-tier confidence semantics (rule vs LLM) must be carefully documented so downstream consumers (ADR-018 admin API) don't conflate them.

### Neutral
- This pipeline shape is provider-agnostic; provider-specific concerns are isolated to Stage 1 and Stage 5 only (see ADR-002).

## Links
- Related: ADR-002 (provider abstraction feeds Stage 1/5), ADR-005 (ingestion sync feeds Stage 1), ADR-007 (message identity used across all stages), ADR-008 (Stage 3 rule engine detail), ADR-009 (Stage 4 LLM adjudication detail), ADR-010 (taxonomy referenced by CategoryVerdict), ADR-013 (feedback loop consumes stage audit trail, batch 012-022), ADR-015 (observability consumes per-stage structured logs, batch 012-022), ADR-022 (LLM cost governance gates Stage 4 invocation, batch 012-022)

## Verification

- Unit tests per stage with fixture `RawMessage`s covering each of the 11 categories' documented signals (§4 table) — assert Stage 3 alone resolves the 8 deterministic categories without invoking Stage 4 (mock the LLM client and assert zero calls).
- Integration test: a corpus of messages with known ground-truth labels is run through the full pipeline; measure % resolved at Stage 3 vs escalated to Stage 4 (target: ≥70% resolved without LLM, tracked as a launch metric, not a hard gate given §9 gap #5).
- Chaos test: simulate Stage 4 (LLM) unavailability/timeout and assert Stage 5 still applies Stage-3 fallback verdicts within SLA and the message is correctly re-queued for reconciliation.
