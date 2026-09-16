# ADR-015: Observability — Logging, Metrics, Tracing, Alerting

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: observability, metrics, tracing, alerting, slo

## Context

Per §3 of the research report, rate limits are mailbox-scoped (Gmail 250 quota units/sec/user, Graph ~4 concurrent requests/mailbox), arguing for a per-mailbox worker/queue model — which means backpressure and throttling failures are a per-mailbox phenomenon that must be individually observable, not just aggregated fleet-wide (a single noisy mailbox can otherwise hide in fleet averages). Per §2, Graph's nested-429 pitfall ("a throttled sub-request is nested inside a 200 batch envelope, so naive code checking only the outer status will silently swallow failures") is an explicitly documented failure mode that generic HTTP-status monitoring will miss — this ADR must specifically account for it.

Per §5, LLM cost is a first-class operational concern (Haiku 4.5 $1/$5 per MTok, Batch API 50% discount, prompt caching at 10% of base input price) — spend must be observable in near-real-time to support the circuit-breaker behavior defined in [[adr-022-llm-cost-governance-circuit-breaker]]. Per §7 Stage 6, the state store tracks "which layer decided" each classification (rule vs. LLM) for auditability — this data doubles as the primary classification-accuracy observability signal.

## Decision

### Structured logging
- All services emit structured JSON logs (one event per line) with a mandatory field set: `tenant_id`, `mailbox_id` (when applicable), `trace_id`, `span_id`, `event_type`, `timestamp`, `severity`.
- Message bodies and header content are **never** logged at any severity level (enforced via a logging middleware allowlist of loggable fields) — this is a hard boundary consistent with [[adr-014-data-retention-gdpr-ccpa-compliance]]; only message-identity hashes ([[adr-007-message-identity-idempotency]]) appear in logs, never raw content.

### Metrics (Prometheus-style, pull-based)
Core metrics, each labeled by `tenant_id` and `provider` (gmail|graph) where cardinality permits (tenant-level cardinality capped via a sampling/rollup rule beyond 500 active tenants to avoid metrics-cardinality blowup):

| Metric | Type | Purpose |
|---|---|---|
| `classification_latency_seconds` | Histogram | End-to-end time from ingestion to category applied, per stage (rule vs LLM) |
| `classification_accuracy_ratio` | Gauge (computed from golden-set eval, [[adr-019-testing-strategy]]) | Tracked per category, per taxonomy version ([[adr-021-schema-taxonomy-versioning]]) |
| `provider_quota_consumed_units` | Counter | Gmail quota units and Graph request counts consumed, per mailbox — direct instrumentation of §1/§2 quota tables |
| `provider_throttle_events_total` | Counter, labeled `nested: true|false` | Distinguishes outer-level 429s from Graph's nested-batch 429s (§2 pitfall) — requires batch response body inspection, not just HTTP status, to populate `nested: true` |
| `llm_spend_usd_total` | Counter, labeled `model`, `api_mode: sync|batch` | Feeds [[adr-022-llm-cost-governance-circuit-breaker]] |
| `llm_cache_hit_ratio` | Gauge | Tracks prompt-cache effectiveness (§5 cost strategy depends on high cache-hit rate) |
| `queue_depth` | Gauge, per-mailbox worker queue | Direct signal for the per-mailbox backpressure model in [[adr-006-per-mailbox-worker-queue-backpressure]] |
| `watch_subscription_expiry_seconds` | Gauge | Time until Gmail watch (7-day) / Graph subscription (~3-day) expiry — renewal-failure early warning per §1/§2 |
| `correction_reversion_rate` | Gauge, per tenant | Cross-referenced from [[adr-013-user-feedback-correction-loop]] |

### Distributed tracing
- OpenTelemetry spans across ingestion → signal extraction → rule scoring → (optional) LLM adjudication → action apply, one trace per message, propagated via the internal message-identity ID as a baggage attribute — enables root-cause tracing of a single misclassified message end-to-end without body content ever entering the trace.
- Spans record `contributing_symbols` (rule names) and `source` (rule/llm/native) as span attributes for the classification decision, mirroring the state store's audit fields (§7 Stage 6) so trace data and durable state agree.

### Alerting thresholds (initial defaults, tunable per [[adr-018-tenant-admin-configuration-api]])
| Alert | Condition | Severity |
|---|---|---|
| Watch/subscription expiring | `watch_subscription_expiry_seconds < 3600` (Gmail) or `< 7200` (Graph, shorter-lived per §2) without a renewal in flight | Page |
| Nested-429 storm | `provider_throttle_events_total{nested="true"}` rate > 5/min sustained 5min for one mailbox | Page |
| Queue depth backlog | `queue_depth` per mailbox > 10,000 sustained 15min | Warn → Page after 1h |
| LLM spend approaching budget | See [[adr-022-llm-cost-governance-circuit-breaker]] thresholds (70%/90%/100% of tenant budget) | Warn/Page per tier |
| Classification accuracy drift | `classification_accuracy_ratio` for any category drops >5 points vs. 30-day rolling baseline on golden-set eval | Warn |
| Cache-hit ratio collapse | `llm_cache_hit_ratio` < 50% sustained 1h (indicates prompt/taxonomy churn breaking cache, e.g., an uncoordinated taxonomy edit) | Warn |

### Dashboards
- Per-tenant operational dashboard (queue depth, quota consumption, spend) exposed read-only via the tenant admin API ([[adr-018-tenant-admin-configuration-api]]) for enterprise/self-serve transparency.
- Internal fleet dashboard aggregating across tenants for on-call use, with the tenant-cardinality rollup applied above 500 tenants.

## Consequences

### Positive
- Explicit nested-429 detection directly addresses a documented, non-obvious Graph failure mode (§2) that naive status-code monitoring would miss entirely.
- Per-mailbox metric labeling matches the mailbox-scoped rate-limit reality (§3), so a single noisy/throttled mailbox is individually diagnosable rather than diluted into fleet averages.
- Trace/log boundary that never carries body content keeps observability infrastructure itself out of GDPR/DPA scope, simplifying [[adr-014-data-retention-gdpr-ccpa-compliance]] compliance.

### Negative
- Per-mailbox/per-tenant label cardinality on Prometheus metrics requires active cardinality management (the 500-tenant rollup rule) — adds operational complexity as tenant count scales, and the rollup threshold will need revisiting as the product grows.
- Populating `nested: true/false` on throttle events requires parsing Graph batch response bodies specifically (not just status codes), a non-trivial instrumentation point that must be kept in sync with any change to the Graph batch client in [[adr-002-provider-abstraction-anti-corruption-layer]].

### Neutral
- Alert thresholds above are initial defaults; like rule-engine weights (§4, §9), they require empirical tuning against real production traffic patterns and are expected to be revised after the first operational quarter.

## Links
- Related: ADR-002 (provider abstraction — source of nested-429 instrumentation point), ADR-006 (per-mailbox worker/queue — queue_depth consumer), ADR-007 (message identity — trace correlation key), ADR-013 (feedback loop — correction_reversion_rate), ADR-014 (compliance — log/trace content boundary), ADR-018 (tenant config — per-tenant dashboards and alert tuning), ADR-019 (testing strategy — golden-set accuracy metric source), ADR-021 (taxonomy versioning — accuracy tracked per version), ADR-022 (cost governance — spend metrics and alert tiers)

## Verification

- Synthetic test: inject a mock Graph batch response with an inner 429 wrapped in an outer 200; assert `provider_throttle_events_total{nested="true"}` increments and the outer-status-only path does not falsely report success.
- Load test: drive one mailbox's queue depth past 10,000 and confirm the warn→page escalation fires at the documented time windows.
- Compliance check (automated, run in CI against log/trace schemas): scan emitted log and span field names against the content-minimization allowlist; fail the build if any body/header-content field is added without an explicit allowlist update and reviewer sign-off.
- Quarterly dashboard review comparing alert-threshold false-positive/false-negative rates against actual incidents, feeding threshold recalibration.
