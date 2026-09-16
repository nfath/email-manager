# ADR-022: LLM Cost Governance & Spend Budget Circuit Breaker

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: cost-governance, llm, budget, circuit-breaker, finops

## Context

Per §5 of the research report, current (Sep 2026) Claude pricing is Haiku 4.5 at $1/$5 per MTok in/out (recommended default tier for classification), Sonnet 5 at $2/$10, and Opus 5 at $5/$25. Cost-optimization levers identified: **Batch API gives a flat 50% discount on both input and output and stacks with prompt caching**; **prompt caching costs 10% of base input price on cache reads**; and forcing structured/short JSON output (rather than prose) is "the single biggest lever on output-token cost." §5 also cites a single-source benchmark (~94% accuracy at ~$0.008/email using a fine-tuned smaller model) explicitly flagged as "directional, not universal truth" — meaning actual per-tenant spend must be measured empirically in production, not assumed from this figure.

Per §7 Stage 4, only the minority of traffic (messages ambiguous after rule-engine scoring) reaches the LLM at all — but per §3's mailbox-scoped rate-limit reasoning, spend still scales roughly with active-tenant-count and mailbox-volume, meaning an unbounded or misbehaving tenant (e.g., unusually high mail volume or unusually high ambiguous-rate due to a rule-engine regression) could generate outsized spend without a governing mechanism. No circuit-breaker or budget-cap mechanism is specified in the report itself — this ADR fills that gap, since research gap #6 (§9, LLM adjudication fallback behavior) and the general cost discussion in §5 together imply a spend-governance mechanism is needed but was left to implementation.

## Decision

### Per-tenant budget model
- Every tenant has a `llm_budget` record: `monthly_cap_usd` (default $50/month per tenant at MVP, configurable via [[adr-018-tenant-admin-configuration-api]]'s `PATCH /v1/tenants/{tenant_id}/llm-budget`), `current_month_spend_usd` (updated in near-real-time from `llm_spend_usd_total` metric, [[adr-015-observability-slos]]), and `alert_thresholds` (default `[0.7, 0.9, 1.0]` of cap).
- Spend is attributed per-tenant by tagging every LLM request (sync or Batch API) with `tenant_id` and accumulating actual billed cost (input tokens × rate + output tokens × rate, with cache-read tokens billed at the 10% rate and Batch API requests billed at the 50%-discounted rate) rather than a flat per-call estimate, so the governance mechanism reflects true cost, not an approximation.

### Circuit breaker behavior (tiered, not binary)
| Spend threshold | Action |
|---|---|
| 70% of monthly cap | Alert only (per [[adr-015-observability-slos]]); no behavior change |
| 90% of monthly cap | **Automatic downgrade**: remaining ambiguous-band messages for that tenant route to Haiku 4.5 exclusively (if a higher tier was configured) and are preferentially queued for the next Batch API window rather than synchronous calls, to maximize the 50% Batch discount for the remainder of the month |
| 100% of monthly cap | **Backoff to rule-engine-only mode** for that tenant: LLM adjudication is skipped entirely; messages that would have escalated to LLM per [[adr-012-phishing-detection-subsystem]]/[[adr-008-deterministic-rule-scoring-engine]] instead use the lowered-threshold rule-only fallback path already defined in [[adr-013-user-feedback-correction-loop]] (same mechanism as an LLM outage/failure) rather than a bespoke budget-specific code path — reusing the existing fallback logic instead of introducing a second one |
| Any threshold | All threshold crossings are logged and surfaced via `GET /v1/tenants/{tenant_id}/llm-budget` and the tenant dashboard; a tenant hitting 100% is never silently degraded without visibility |

- Budget enforcement is **per-tenant, not global** — one tenant exceeding its cap never throttles or degrades another tenant's classification quality, consistent with the isolation principle in [[adr-003-multi-tenant-data-isolation]].
- A tenant may opt out of automatic downgrade/backoff (configurable, for enterprise tenants with negotiated uncapped or higher-cap contracts) but the 70%/90%/100% *alerting* remains mandatory regardless of opt-out, so spend visibility is never fully disabled even when enforcement is.

### Cost-minimization defaults (enforced architecturally, not just by convention)
- The LLM adjudication service ([[adr-009-llm-adjudication-service]]) defaults every request to Haiku 4.5 unless a specific category/tenant configuration explicitly requires a higher tier (e.g., a particularly high-stakes phishing residual case might justify Sonnet 5 — this is a deliberate, reviewed exception path, not a default).
- Every LLM request is required to use the shared cached system-prompt/taxonomy prefix (per [[adr-009-llm-adjudication-service]] and [[adr-013-user-feedback-correction-loop]]'s tenant-specific few-shot suffix design) — a request bypassing the cache path is treated as a bug and alerts via the `llm_cache_hit_ratio` metric ([[adr-015-observability-slos]]).
- Structured JSON output (category + confidence + minimal reasoning symbols, not free-text prose) is enforced via response-format constraints in the LLM client, directly operationalizing §5's "biggest lever on output-token cost" finding.
- Non-latency-sensitive traffic (the initial backlog sweep for a newly onboarded mailbox, or routine reconciliation-driven reclassification) defaults to the Batch API path; only messages requiring near-real-time triage (e.g., interactive "why was this labeled X" flows) use synchronous calls — this routing decision is made automatically based on message age/context, not left to per-call developer discretion.

### Reporting
- Monthly per-tenant spend report (input/output tokens, cache-hit savings, Batch-API savings, sync vs. batch split) available via the tenant admin API and used both for internal FinOps review and, where applicable, customer-facing cost transparency.

## Consequences

### Positive
- Tiered (not binary) circuit breaker means a tenant approaching budget degrades gracefully (cheaper model, batch-preferred) before an outright hard stop, minimizing the chance of unexpectedly halting classification mid-month.
- Reusing the existing LLM-failure fallback path (from [[adr-013-user-feedback-correction-loop]]) for the 100%-cap backoff case avoids building and maintaining a second, parallel "no LLM available" code path — one mechanism serves two triggers (provider failure and budget exhaustion).
- Architecturally enforced defaults (Haiku-first, cached-prompt-required, structured-output-required, batch-preferred) mean cost discipline doesn't depend on every future developer remembering the cost-optimization guidance from §5 — it's a structural property of the request path.

### Negative
- Per-tenant real-time spend tracking requires accurate, low-latency cost attribution from the LLM provider's billing/usage data, which may lag actual API responses slightly (token counts are known immediately, but reconciling against actual billed cost, e.g., Batch API discount application, may only be confirmed after batch completion) — the system should track spend from token counts directly rather than waiting on provider invoicing, and treat provider invoice reconciliation as a periodic audit, not the real-time source of truth.
- Automatic tier-downgrade at 90% could, in principle, reduce classification quality for a tenant right before month-end if their ambiguous-message rate spikes — an explicit tradeoff of cost control over classification quality that must be communicated to affected tenants, not a silent quality regression.
- Default $50/tenant/month cap is an initial guess with no cited benchmark in the research report (§5's cost figures are per-email, not per-tenant-aggregate) — will need calibration against real tenant mail volumes and ambiguous-rate percentages once production data exists.

### Neutral
- Enterprise opt-out of enforcement (while keeping alerting mandatory) is a deliberate compromise between protecting the platform's own margin (default tenants) and accommodating negotiated contracts (large tenants) — expected to be revisited as the commercial model matures.

## Links
- Related: ADR-003 (multi-tenant isolation — per-tenant budget scoping), ADR-008 (rule engine — fallback path reused at 100% cap), ADR-009 (LLM adjudication service — model tier defaults, cache/structured-output enforcement), ADR-012 (phishing detection — example of a justified higher-tier exception), ADR-013 (feedback loop — shared fallback mechanism reused here), ADR-015 (observability — spend metrics and alert thresholds), ADR-018 (tenant admin API — budget configuration endpoint), ADR-019 (testing strategy — cache-hit-rate regression check tied to cost)

## Verification

- Unit test: simulate a tenant crossing 70%/90%/100% thresholds and assert the corresponding alert/downgrade/backoff behavior triggers exactly once per threshold crossing (no repeated re-triggering, no missed transition).
- Integration test: confirm a tenant at 100% cap correctly falls back to rule-engine-only classification via the same code path exercised by an LLM provider outage, verifying the two triggers share one implementation.
- Cost-accounting test: replay a batch of requests with known token counts (including cache-read and Batch-API-discounted requests) and assert computed `current_month_spend_usd` matches the expected billed amount within a defined tolerance.
- Monthly reconciliation: compare system-computed spend against the LLM provider's actual invoice; alert on any discrepancy beyond a small tolerance (e.g., 2%), feeding back into the real-time cost-tracking logic if drift is found.
- Isolation test: drive one tenant to 100% of its cap and confirm a second tenant's classification latency/quality/spend is entirely unaffected.
