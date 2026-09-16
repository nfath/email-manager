# ADR-019: Testing Strategy — Contract, Golden-Set, and Regression Gating

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: testing, quality, contract-tests, evaluation, ci

## Context

Per §9 of the research report (research gap #5), "no public reference dataset of labeled emails for this exact 11-category taxonomy exists" — the rule-scoring weights and thresholds throughout [[adr-008-deterministic-rule-scoring-engine]], [[adr-012-phishing-detection-subsystem]], and others are explicitly called out as requiring "empirical tuning against real mailbox data." This means the testing strategy cannot rely on an off-the-shelf benchmark and must build its own golden set as a first-class deliverable, not an afterthought.

Per §2, Graph's batch/throttling behavior includes the documented nested-429 pitfall, and per §1/§2 both providers have distinct quota/rate-limit/push mechanics — meaning provider-facing code needs contract tests against real (sandbox) provider behavior, not mocks alone, to catch integration drift as Google/Microsoft evolve their APIs (§2 explicitly flags Microsoft's Entra/admin surfaces as changing "frequently"). Per §7, the LLM adjudication layer's output quality is central to the whole cost/accuracy tradeoff (§5) — classification accuracy regressions must be caught before reaching production, especially since prompt or taxonomy changes could silently degrade accuracy for categories that previously worked well.

## Decision

### Test pyramid layers

**1. Unit tests** (fastest, run on every commit)
- Per-symbol rule tests (each named signal in [[adr-008-deterministic-rule-scoring-engine]] and [[adr-012-phishing-detection-subsystem]] gets isolated fixture-based tests: header parsing, List-Unsubscribe detection, schema.org JSON-LD parsing, iCalendar METHOD:CANCEL parsing, SPF/DKIM/DMARC verdict parsing).
- Provider abstraction/anti-corruption layer unit tests ([[adr-002-provider-abstraction-anti-corruption-layer]]) using recorded fixture payloads for both Gmail and Graph message shapes.

**2. Contract tests against provider sandboxes** (run in CI on a schedule, e.g., every 6 hours, not on every commit, to respect provider quota)
- Dedicated Gmail test Workspace account(s) and a Microsoft 365 developer tenant, provisioned specifically for CI (never production tenant credentials).
- Contract suite validates: `users.watch` registration/renewal round-trip, `history.list` reconciliation correctness after a simulated gap, Graph delta-query round-trip, Graph batch requests specifically exercising the nested-429 case (§2) by deliberately triggering throttling and asserting the client correctly detects and retries the inner failure rather than treating the outer 200 as success.
- Contract tests are the explicit mechanism for catching provider-API drift (per §2's caution about Microsoft's admin surfaces changing frequently) — a scheduled run, not just a pre-release check, so drift is caught within hours, not at the next release.

**3. Golden-set classification evaluation harness** (the core deliverable addressing research gap #5)
- Build and maintain a labeled corpus of real (consented, sanitized) or synthetically constructed representative emails covering all 11 categories, stratified to include known-hard cases per §4's confidence gradings: ATS/job-posting sender variety (Medium confidence per §4 — needs empirical sampling), needs-reply ambiguous threads (Medium-low confidence), phishing lookalike-domain cases (the highest-value/hardest rule per [[adr-012-phishing-detection-subsystem]]).
- Harness runs the full pipeline (rule engine + LLM adjudication) against the golden set and computes per-category precision/recall/F1, plus a confusion matrix across the 11 categories (multi-label, so measured as per-category binary precision/recall, not single-label accuracy).
- Golden-set size target: minimum 50 labeled examples per category at MVP (550 total), growing via the correction-loop pipeline in [[adr-013-user-feedback-correction-loop]] — corrections that a human reviewer confirms as high-quality are periodically promoted into the golden set (not automatically, to avoid the golden set silently drifting to match model biases).
- Golden-set evaluation is versioned against taxonomy version ([[adr-021-schema-taxonomy-versioning]]) so accuracy comparisons across taxonomy changes remain meaningful.

**4. Regression gating in CI/CD**
- Every change to rule weights, symbol logic, or the LLM system prompt/few-shot template must run the full golden-set harness before merge.
- Gate: no category's F1 score may regress more than 3 points versus the current production baseline without an explicit override + sign-off from the architecture team (a hard block by default, escapable only via documented exception, not silently skippable).
- LLM prompt changes additionally run a **cache-hit-rate regression check**: since prompt caching is central to the cost model (§5), a prompt-structure change that drops cache-hit rate is flagged even if accuracy is unaffected, feeding [[adr-022-llm-cost-governance-circuit-breaker]].

**5. End-to-end smoke tests**
- Full pipeline run (ingestion → classification → action-apply) against sandbox accounts before each production deploy, verifying labels/categories actually appear correctly translated per-provider (Gmail label vs. Graph category, per [[adr-010-category-taxonomy-action-mapping]]).

### Test data governance
- No production tenant mail content is ever used in the golden set or contract-test fixtures without explicit, documented consent and sanitization (stripping any residual PII beyond what's needed for the specific signal under test) — consistent with the content-minimization posture in [[adr-014-data-retention-gdpr-ccpa-compliance]].
- Synthetic fixture generation (templated variations of known patterns: List-Unsubscribe variants, schema.org Order JSON-LD variants, phishing lookalike-domain variants) is preferred over real-mail sourcing wherever it can adequately cover a signal, precisely because no canonical dataset exists (§9 gap #5) and synthetic data sidesteps consent/privacy overhead entirely.

## Consequences

### Positive
- Directly closes the most consequential research gap flagged in the report (§9 #5: no reference dataset) by making golden-set construction an explicit, tracked engineering deliverable rather than an implicit assumption.
- Contract tests against real sandbox accounts, run on a schedule independent of releases, catch provider-API drift (explicitly flagged as a risk for Microsoft's Entra surfaces, §2) proactively rather than via a production incident.
- Nested-429 contract test specifically operationalizes a documented, non-obvious failure mode (§2) into a concrete, automatable check rather than leaving it as tribal knowledge.

### Negative
- Building and maintaining a 550+ example golden set (growing over time) is real, ongoing labeling effort — not a one-time cost, and requires a human-review step for correction-to-golden-set promotion that must be staffed.
- Scheduled contract tests against provider sandboxes consume real API quota and require maintaining dedicated non-production Gmail/M365 tenant accounts indefinitely, including keeping their OAuth grants alive and monitored.
- The 3-point F1 regression gate is a somewhat arbitrary initial threshold (no industry-standard figure was found in the research for this exact taxonomy) and may need loosening/tightening once real variance in golden-set evaluation is observed.

### Neutral
- Golden-set growth via correction-loop promotion creates a feedback dependency between [[adr-013-user-feedback-correction-loop]] and this ADR — the two systems must be built with awareness of each other, not independently.

## Links
- Related: ADR-002 (provider abstraction — contract test target), ADR-008 (rule engine — unit test target and regression-gated), ADR-009 (LLM adjudication — prompt regression and cache-hit checks), ADR-010 (taxonomy/action mapping — e2e smoke test target), ADR-012 (phishing detection — hardest golden-set stratum), ADR-013 (feedback loop — golden-set growth source), ADR-014 (compliance — test-data governance), ADR-016 (deployment topology — staging environment with sandbox accounts), ADR-021 (taxonomy versioning — golden-set versioning), ADR-022 (cost governance — cache-hit regression check)

## Verification

- CI dashboard showing golden-set F1 per category per build, with historical trend line, reviewed at each release readiness check.
- Scheduled contract-test suite green/red status tracked as its own uptime metric (per [[adr-015-observability-slos]]); a sustained red status blocks the next scheduled release until root-caused.
- Quarterly golden-set audit: sample-review a subset of golden-set entries for label quality/staleness, and check corpus category balance hasn't drifted (e.g., no category should silently shrink below the 50-example minimum as data ages out).
- Post-incident process: any production misclassification incident results in the offending example being added to the golden set (after review) so the same failure mode is regression-tested going forward.
