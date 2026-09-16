# ADR-008: Deterministic Rule/Signal Scoring Engine Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: classification, rules-engine, spamassassin, rspamd

## Context

Per §6 of the research report, SpamAssassin's architecture is described as "the canonical rule-scoring system — hundreds of independent rules, each with signed weight; sum crosses threshold → verdict," and is called out as "directly transferable: additive-scoring pattern (rather than monolithic classifier) directly applicable to our multi-label problem — each category gets its own weighted-signal accumulator with independent threshold." Rspamd is identified as the modern successor and the recommended architectural template overall (§6, §7 Stage 3).

Per §4, the signal table gives concrete, gradeable signals per category with explicit confidence levels: **High confidence** signals include `List-Unsubscribe`/`List-Id` headers (RFC 2369/8058), `Precedence: bulk`, `Auth-Submitted` (RFC 3834), iTIP/iMIP `METHOD:CANCEL` (RFC 5546/6047), and schema.org `Order`/`ParcelDelivery` JSON-LD. **Medium confidence** signals include ATS sending-domain lists (no canonical current list exists — §9 gap #1) and promotional lexicon matching. §9 gap #5 explicitly states: "The SpamAssassin/Rspamd-inspired rule-scoring model requires empirical tuning of signal weights and per-category confidence thresholds against real mailbox data — no public reference dataset of labeled emails for this exact 11-category taxonomy exists." This ADR must therefore define the *mechanism* precisely while treating specific weight values as initial defaults subject to tuning, not settled constants.

## Decision

**Rule definition format**: Each rule is a named, independently testable unit that inspects `SignalSet` (Stage 2 output, ADR-001) and contributes a signed weight to one or more category accumulators:

```typescript
interface Rule {
  id: string;                    // e.g. "LIST_UNSUBSCRIBE_PRESENT"
  appliesToCategories: Category[];
  weight: number;                 // positive = evidence for, negative = evidence against
  confidenceGrade: 'high' | 'medium' | 'low'; // per §4 grading, informs weight magnitude
  evaluate(signals: SignalSet): boolean;
}
```

**Initial rule set (seeded directly from §4's high-confidence signals)**:

| Rule ID | Category | Signal | Weight | Confidence (§4) |
|---|---|---|---|---|
| `LIST_UNSUBSCRIBE_RFC2369` | newsletter | `List-Unsubscribe` header present | +8 | high |
| `LIST_UNSUBSCRIBE_POST_RFC8058` | newsletter | `List-Unsubscribe-Post` present | +3 | high |
| `LIST_ID_PRESENT` | newsletter | `List-Id` header present | +6 | high |
| `ESP_DOMAIN_KNOWN` | newsletter, promo_deal | sender domain in curated ESP allowlist (Mailchimp `*.mcsv.net`, `list-manage.com`, Substack `*.substack.com`, ConvertKit) | +7 | high |
| `PROMO_LEXICON_MATCH` | promo_deal | subject/body matches promotional lexicon ("% off", "sale ends") | +4 | medium |
| `GMAIL_CATEGORY_PROMOTIONS` | promo_deal, newsletter | native `CATEGORY_PROMOTIONS` present | +3 | high (as a prior, per §3 point 3) |
| `LINKEDIN_SENDER_DOMAIN` | linkedin | sender domain `*.linkedin.com` | +10 | high |
| `ATS_DOMAIN_KNOWN` | job_posting | sender domain in curated ATS allowlist (Greenhouse, Lever, Ashby, Workday) | +6 | medium (§9 gap #1 — needs empirical expansion) |
| `SCHEMA_ORG_ORDER_JSONLD` | ecommerce_receipt | schema.org `Order`/`ParcelDelivery` JSON-LD block present | +10 | high |
| `RECEIPT_SUBJECT_REGEX` | ecommerce_receipt | subject matches `/your order #|shipped|receipt/i` | +4 | medium |
| `ICAL_METHOD_CANCEL` | meeting_cancelled | MIME `text/calendar; method=CANCEL` or body `METHOD:CANCEL` | +12 | high (near-deterministic per §4) |
| `NOREPLY_SENDER_PATTERN` | needs_reply | sender matches `/^no.?reply@/i` | -8 | high (exclusionary) |
| `AUTO_SUBMITTED_HEADER` | needs_reply, personal | `Auto-Submitted: auto-generated\|auto-replied` (RFC 3834) | -8 | high (exclusionary) |
| `PRECEDENCE_BULK` | needs_reply, personal | `Precedence: bulk` | -6 | high (exclusionary) |
| `INTERROGATIVE_STRUCTURE` | needs_reply | body contains `?` in final paragraph / imperative sentence heuristic | +3 | medium (§4: ~80% interrogative, 11% imperative in research) |
| `AUTH_RESULTS_FAIL` | phishing | `Authentication-Results` shows SPF/DKIM/DMARC fail | +6 | high (necessary but insufficient per §4) |
| `LOOKALIKE_DOMAIN_MATCH` | phishing | sender domain within edit-distance-2 of a known contact/brand domain | +9 | medium |
| `URGENCY_LEXICON_MATCH` | phishing | body matches urgency lexicon ("verify your account immediately", "suspended") | +3 | medium |
| `SOCIAL_SENDER_DOMAIN` | social | sender domain in curated social-platform allowlist (Facebook, Instagram, X, Reddit) | +9 | high |
| `NO_BULK_SIGNALS_PRESENT` | personal | absence of List-*, Precedence:bulk, Auto-Submitted simultaneously | +5 | high |
| `SENDER_IN_ADDRESS_BOOK` | personal, priority_vip | sender in People API contacts | +6 | high |

**Scoring and threshold mechanism**: Per category, sum all applicable rule weights that fire into a raw score, then normalize to `[0,1]` via a logistic transform calibrated per category (`confidence = sigmoid((raw_score - midpoint) / scale)`, with `midpoint`/`scale` as per-category tunable parameters, not global constants — since e.g. `meeting_cancelled`'s single near-deterministic rule needs a very different curve shape than `needs_reply`'s many-weak-signals accumulation).

**Per-category threshold bands** (feeding ADR-001's Stage 3→4 escalation gate):
- `confidence >= high_threshold` (default 0.85): finalize at Stage 3, `decidedBy: 'rule'`.
- `confidence <= low_threshold` (default 0.15): finalize as **not** this category at Stage 3 (no escalation needed for exclusion).
- `low_threshold < confidence < high_threshold`: escalate to Stage 4 LLM adjudication (ADR-009).
- Categories explicitly flagged `llmRequired: true` in the taxonomy config (`needs_reply`, and `phishing` beyond the deterministic `AUTH_RESULTS_FAIL`/`LOOKALIKE_DOMAIN_MATCH` gate) always escalate regardless of score, per §4's explicit callout of needs-reply as "the single most LLM-dependent category."

**Weight tuning as a first-class operational concern**: Because §9 gap #5 states no public reference dataset exists for this exact taxonomy, all weights above are **defaults**, stored in a versioned config table (not hardcoded), and expected to be tuned against the user-correction feedback loop (ADR-013, batch 012-022) post-launch. Every rule firing is logged with its contribution to the final score (`ruleIds` in `CategoryVerdict`, ADR-001) specifically so weight-tuning can be done from real outcome data rather than re-guessing.

## Consequences

### Positive
- Additive, named-rule scoring gives full explainability per verdict (which rules fired, by how much) — directly required for the audit trail (ADR-007) and user trust ("why did this get labeled X").
- Rspamd/SpamAssassin-proven pattern scales to hundreds of rules without architectural rework — new signals are added as new `Rule` implementations, not changes to a monolithic scoring function.
- Explicit `llmRequired` escape hatch correctly routes the categories the research identifies as fundamentally judgment-based (needs_reply) to LLM adjudication regardless of rule score, avoiding false confidence from a purely heuristic approach the report itself says "has known precision problems" (§4).

### Negative
- Initial weights are best-effort defaults, not empirically validated (§9 gap #5) — expect a tuning period post-launch with real classification-quality risk until sufficient correction-feedback volume accumulates.
- ATS domain list (`ATS_DOMAIN_KNOWN`) is explicitly incomplete per §9 gap #1 and will require ongoing manual/empirical curation, not a one-time setup task.
- Per-category logistic calibration (distinct midpoint/scale per category) adds tuning-surface complexity versus a single global threshold.

### Neutral
- This ADR does not decide the retraining/weight-adjustment algorithm itself (e.g., simple manual review vs automated gradient-based weight adjustment from corrections) — that is ADR-013's concern (batch 012-022); this ADR only fixes the scoring mechanism and initial weight table the retraining loop operates on.

## Links
- Related: ADR-001 (Stage 3 of the pipeline, escalation gate consumer), ADR-004 (scope-degradation may disable People-API-dependent rules like `SENDER_IN_ADDRESS_BOOK`), ADR-009 (Stage 4 LLM adjudication for escalated/llmRequired categories), ADR-010 (taxonomy defines which categories are `llmRequired`), ADR-011 (VIP scoring extends `SENDER_IN_ADDRESS_BOOK`/reciprocity beyond this rule engine), ADR-013 (feedback loop tunes weights, batch 012-022)

## Verification

- Unit tests per rule with synthetic `SignalSet` fixtures covering fire/no-fire cases for every rule in the table.
- Regression corpus: a growing set of real (anonymized/synthetic) messages with ground-truth labels, run against the scoring engine on every rule-set change; track precision/recall per category and diff against the prior rule-set version before merge.
- Threshold-tuning dashboard showing the distribution of raw scores per category in production, used to validate/adjust `high_threshold`/`low_threshold`/`midpoint`/`scale` empirically rather than by guesswork (directly addresses §9 gap #5).
