# ADR-009: LLM Adjudication Service Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: llm, cost-optimization, claude, batch-api, prompt-caching

## Context

Per §5 of the research report, "Current pricing (Sep 2026) — Claude models preferred for this task" gives: Haiku 4.5 at $1/$5 per MTok in/out ("recommended default tier — classification/routing is its explicit use case"), Sonnet 5 at $2/$10 per MTok in/out, Opus 5 at $5/$25 per MTok in/out. The report identifies concrete cost-optimization mechanisms: **Batch API gives a flat 50% discount on both input and output, and stacks with prompt caching**; **prompt caching costs 10% of base input price** on non-Fable models. §5's "practical combo" recommendation is explicit: "Prompt-cache the fixed system prompt/taxonomy + few-shot examples, batch multiple emails' variable content into one request, use Haiku 4.5, prefer async Batch API for non-latency-sensitive work (initial sweep) and synchronous calls only for time-sensitive cases (real-time nudges)."

§5 also cites batching literature showing "prompt batching (multiple items, one prompt, one structured response) gives ~1.2-1.9x median per-item speedup vs one-call-per-item" and that "forcing structured/short outputs (single-token enum or JSON, not prose) is the single biggest lever on output-token cost." A single-source benchmark (flagged §9 gap #3, not to be treated as guaranteed) suggests fine-tuned small models can hit ~94% accuracy at ~$0.008/email — directional only.

Per §8, content minimization is a stated design goal: only messages that reach this stage (ambiguous after Stage 3, or in an `llmRequired` category per ADR-008) should have body content sent to an LLM at all.

## Decision

**Model tier**: Default to **Claude Haiku 4.5** for all adjudication calls. Sonnet 5/Opus 5 are not used in the standard pipeline; they are reserved as an optional escalation tier only if a future confidence-monitoring signal (ADR-013 feedback loop, batch 012-022) shows Haiku systematically underperforming on a specific category — this is an explicit non-default, config-gated override, not part of the initial design, to keep steady-state cost predictable (see ADR-022 cost governance, batch 012-022).

**Invocation modes**:
1. **Batch mode (default)** — used for reconciliation-sweep-discovered backlog and any non-time-sensitive classification. Submitted via Anthropic's Batch API, gaining the 50% batch discount stacked with prompt caching. Batch jobs are polled asynchronously; results are written back into `classification_state` (ADR-007) as they complete, and Stage 5 (apply layer, ADR-001) proceeds once a batch item resolves — the pipeline does not block other messages on a slow batch job.
2. **Synchronous mode** — used only for push-triggered, user-visible time-sensitive cases (e.g., a message the user is actively viewing that needs immediate needs-reply classification for a UI badge). Synchronous calls skip the batch discount but still use prompt caching. Synchronous mode is rate-limited separately from batch mode to avoid an interactive spike starving batch throughput.

**Request shape — batched multi-item classification, not one-call-per-email**:

```typescript
interface AdjudicationRequest {
  systemPromptVersion: string;   // cache key — fixed taxonomy + few-shot examples, prompt-cached
  items: Array<{
    message_id: string;          // internal_message_id (ADR-007)
    headers_summary: string;     // minimized: only fields relevant to ambiguous categories
    body_excerpt?: string;       // only included if category requires body (needs_reply, phishing residual)
    candidate_categories: Category[]; // narrowed by Stage 3 rule scores, not all 11
    rule_scores: Record<Category, number>; // gives the LLM the rule engine's partial evidence
  }>;
}

interface AdjudicationResponse {
  results: Array<{
    message_id: string;
    categories: Array<{ category: Category; confidence: number }>;
    rationale_short?: string;    // 1 short phrase, for audit — not prose explanation
  }>;
}
```

Requests batch **up to 20 messages per call** (tuned against Haiku's context window and to keep individual batch-item latency reasonable; not a hard provider limit, a design choice balancing the "1.2-1.9x per-item speedup" cited in §5 against per-call payload size). The system prompt (taxonomy definitions, few-shot examples per category, output-format instructions) is marked for prompt caching and versioned (`systemPromptVersion`) so taxonomy changes (ADR-021, batch 012-022) invalidate the cache deliberately rather than silently serving stale-taxonomy-cached responses.

**Structured output enforcement**: The response schema is enforced via Claude's structured-output/tool-use mechanism (JSON schema-constrained output), never free-form prose — directly implementing §5's "single biggest lever on output-token cost." `rationale_short` is capped at ~10 words specifically to avoid the cost blowup of unconstrained explanation text.

**Content minimization gate (enforced before this stage is ever invoked)**: Per §8 and ADR-001's escalation gate, only messages that are (a) ambiguous after Stage 3 for a body-requiring category, or (b) in an explicitly `llmRequired` category, ever have `body_excerpt` populated. For categories resolvable from headers alone even when escalated (e.g., an ambiguous newsletter/promo split), `body_excerpt` is omitted entirely — only `headers_summary` and rule scores are sent. This is enforced in code (a pre-request assembly function that only attaches body content for a fixed allowlist of category+ambiguity combinations), not left to prompt discipline alone.

**Fallback on failure/timeout**: Per ADR-001's Stage 4 fallback contract — if the LLM call errors or times out (default 30s for sync, batch jobs have no hard timeout but a staleness alert if a batch is pending >6h), the pipeline falls back to Stage 3's best-effort rule verdict and re-queues for a later retry, never blocking Stage 5 indefinitely.

## Consequences

### Positive
- Batch API + prompt caching stacking directly minimizes steady-state LLM spend for the (expected, per ADR-001) minority of ambiguous traffic.
- Structured multi-item output avoids per-call overhead and keeps output tokens minimal, addressing both cost and latency.
- Body content is attached only when strictly necessary, materially reducing the privacy/DPA surface area described in §8 versus a naive "send every email" design.

### Negative
- Batch mode introduces result latency (minutes to hours depending on Batch API queue depth) which is acceptable for reconciliation-sweep traffic but requires the synchronous fallback path to exist and be separately maintained/tested.
- Fixed default to Haiku 4.5 risks under-performing on genuinely hard cases (e.g., sophisticated phishing) without a tuned escalation trigger in place from day one — the escalation-to-Sonnet/Opus path is designed but not automatically wired up initially, a deliberate scope-reduction to avoid unbounded cost risk pre-launch (see ADR-022, batch 012-022).
- The 94%-accuracy/$0.008-per-email benchmark cited in §5 is single-source (§9 gap #3) and must not be used as a committed SLA in customer-facing materials.

### Neutral
- Model tier and batch-size-per-call (20) are configuration values, expected to be tuned once real cost/latency/accuracy data is available in production.

## Links
- Related: ADR-001 (Stage 4 of the pipeline), ADR-008 (rule scores feed the adjudication request as partial evidence), ADR-010 (taxonomy and category definitions used in the system prompt), ADR-021 (`systemPromptVersion` ties to taxonomy versioning, batch 012-022), ADR-022 (cost governance/spend circuit breaker gates this service, batch 012-022), ADR-013 (feedback loop uses adjudication rationale/outcomes for retraining, batch 012-022)

## Verification

- Cost dashboard tracking $/email adjudicated, batch vs sync split, and cache-hit rate on the system prompt — validated against §5's directional cost expectations, not treated as a guarantee.
- Golden-set evaluation (ADR-019, batch 012-022): a held-out labeled corpus run through adjudication on every prompt/model version change, tracking per-category precision/recall drift.
- Contract test asserting `body_excerpt` is never populated for a request unless the category+ambiguity combination is on the explicit content-minimization allowlist (a unit test on the pre-request assembly function, not a manual review process).
- Latency SLO check: synchronous-mode p95 latency under a defined budget (e.g., 5s) tracked separately from batch-mode turnaround time.
