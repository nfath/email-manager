# ADR-012: Phishing Detection Subsystem Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: security, phishing, classification, rule-engine, llm

## Context

Per §4 of the research report, phishing/security is the category with the widest signal surface: SPF/DKIM/DMARC authentication results are "necessary but insufficient" because lookalike/cousin domains routinely pass all three checks legitimately. Per §8, phishing detection also drives the *broadest data-access requirement* of any category in the system — it needs full headers (for `Authentication-Results`) and full body content (for lookalike/urgency-language detection), in direct tension with the metadata-first content-minimization posture established in [[adr-014-data-retention-gdpr-ccpa-compliance]].

The report explicitly recommends (§4, §8) that LLM output for this category be treated as "an additional signal feeding a rule-based final gate, not sole determinant" — i.e., the LLM must never be the sole authority that marks a message as phishing or safe. This is a deliberate architectural constraint: false negatives are a security incident, false positives destroy user trust in the whole triage system, and LLMs are not deterministic or independently auditable enough to be a lone gatekeeper for a security-critical verdict.

The report also notes native provider verdicts (Gmail's built-in phishing warnings, Microsoft Defender/EOP verdicts via Graph, where licensed) should be checked as a substitute/supplement before rebuilding detection from scratch (§8), and flags this as worth re-validating at implementation time since the exact fields exposed vary by tenant licensing (Confidence: Medium-high, per §8).

## Decision

Implement phishing detection as a **4-stage sub-pipeline** inside Stage 3/4 of the overall layered architecture ([[adr-001-layered-classification-pipeline]]), producing a `PhishingVerdict` that is always rule-gated regardless of LLM involvement.

### 1. Native-verdict ingestion (cheapest, checked first)
- Gmail: read `X-Gmail-...` phishing/spam warning banners is not API-exposed directly; instead check message `labelIds` for `SPAM` and inspect `payload.headers` for `Authentication-Results` verdicts Gmail itself already computed.
- Microsoft Graph: read `singleValueExtendedProperties` for the `X-MS-Exchange-Organization-SCL` (Spam Confidence Level, -1 to 9) and, where Defender for Office 365 is licensed, the `X-Forefront-Antispam-Report` header via `internetMessageHeaders` on `GET /messages/{id}?$select=internetMessageHeaders`.
- If a native verdict marks the message as high-confidence phishing/malware (Gmail `SPAM`/`labelIds` contains system spam label with explicit phishing sub-marker, or SCL ≥ 6), short-circuit directly to `phishing: true, source: native_provider` — skip rule engine and LLM entirely. This is the cheapest and (per §8) most authoritative signal when available.

### 2. Deterministic signal extraction (Stage 2, every message)
Score components, each independently computed and stored as named symbols (Rspamd-style, per §6/§7):
- `AUTH_FAIL`: `Authentication-Results` header parsed for `spf=fail|softfail`, `dkim=fail`, `dmarc=fail|quarantine|reject`. Weight: +8 per failing mechanism (max +24).
- `AUTH_PASS_LOOKALIKE`: all three pass BUT sending domain is a Levenshtein-distance ≤2 match (or homoglyph match via `confusable_homoglyphs` normalization) against (a) the recipient's own past-contacted domains (from People API + reciprocity table, [[adr-011-vip-relationship-graph-scoring]]) or (b) a maintained top-2000 brand-domain reference list. Weight: +12 (this is the highest-value rule per §4 — it is precisely the case that passes auth but is still phishing).
- `URGENCY_LEXICON`: body/subject match against a maintained urgency/pressure lexicon ("verify your account within 24 hours", "your account has been suspended", "confirm payment now", wire-transfer language). Weight: +4 per matched phrase, max +12.
- `DISPLAY_NAME_MISMATCH`: `From` display name contains a brand/person name that doesn't match the `From` address domain (e.g., display name "PayPal Support" from a non-paypal.com domain). Weight: +10.
- `NEW_SENDER_FINANCIAL_ASK`: sender has zero prior reciprocal history (per [[adr-011-vip-relationship-graph-scoring]]) AND body contains financial-action lexicon (invoice, wire, gift card, payment). Weight: +6.

### 3. Rule-engine threshold gate (Stage 3)
- Score ≥ 20: finalize as `phishing: true, source: rule_engine`, no LLM call.
- Score ≤ 4: finalize as `phishing: false, source: rule_engine`, no LLM call (majority of mail, per §7 "large majority resolved without LLM").
- Score 5–19 (ambiguous band): escalate to Stage 4 LLM adjudication.

### 4. LLM adjudication as advisory input to a final rule gate
- Only the ambiguous band (ADR estimates 10-20% of mail pre-native/rule resolution, to be measured empirically per open question #5 in the research report §9) is sent to Claude Haiku 4.5 with full body content, per the content-minimization carve-out in [[adr-014-data-retention-gdpr-ccpa-compliance]].
- LLM prompt requests structured output: `{message_id, phishing_likelihood: 0-100, reasoning_symbols: [...], recommended_action}`.
- **The LLM's output is never applied directly.** It is converted into an additional weighted symbol (`LLM_PHISHING_SIGNAL`, weight = `likelihood/100 * 15`) and re-summed against the Stage 3 score. Final gate: combined score ≥ 20 → `phishing: true, source: rule_gate_llm_assisted`; otherwise → `phishing: false, source: rule_gate_llm_assisted`.
- This guarantees the rule-based threshold logic is the sole authority for the final boolean verdict, satisfying the report's explicit non-negotiable ("LLM as residual-case adjudicator feeding a rule-based final gate, not sole determinant").

### Action mapping
- `phishing: true` → apply `phishing` category label/tag (per [[adr-010-category-taxonomy-action-mapping]]) AND (configurable per tenant, [[adr-018-tenant-admin-configuration-api]]) optionally move to a quarantine folder/label rather than delete — the system never auto-deletes user mail.
- Every phishing verdict, its contributing symbols, and scores are persisted for audit (per [[adr-013-user-feedback-correction-loop]] and [[adr-015-observability-slos]]).

## Consequences

### Positive
- Rule-based final gate bounds LLM unpredictability out of the security-critical decision boundary — an LLM prompt-injection or hallucination cannot unilaterally mark benign mail as phishing or vice versa.
- Native-verdict short-circuit avoids re-deriving detection the provider already computed, reducing both cost and LLM exposure for the clearest cases.
- Symbol-based scoring is auditable and independently tunable per symbol without redesigning the pipeline (mirrors Rspamd, per §6).

### Negative
- Requires full body/header access for the ambiguous band, which is the broadest privacy scope of any category (per §8) — creates DPA/legal review dependency on [[adr-014-data-retention-gdpr-ccpa-compliance]].
- Lookalike-domain detection requires maintaining and periodically refreshing a brand-domain reference list and per-tenant contact-domain corpus — ongoing curation cost, not a one-time build.
- Native Microsoft Defender/EOP header availability depends on tenant licensing (Confidence: Medium-high per §8) — must be feature-detected per tenant at onboarding, not assumed universally available.

### Neutral
- Threshold values (20/4/12/8/etc.) are initial estimates requiring empirical tuning against real labeled mailbox data, per research gap #5 in §9 of the report — expect a tuning pass in V2/V3, tracked as an explicit follow-up rather than treated as final.

## Links
- Related: ADR-001 (layered pipeline), ADR-008 (rule/signal scoring engine — phishing reuses its scoring primitives), ADR-009 (LLM adjudication service), ADR-010 (category taxonomy/action mapping), ADR-011 (VIP/relationship graph — supplies reciprocity data for lookalike/new-sender scoring), ADR-014 (data retention/compliance — governs body-content access), ADR-015 (observability), ADR-018 (tenant config — quarantine behavior)

## Verification

- Unit tests per symbol (AUTH_FAIL, AUTH_PASS_LOOKALIKE, URGENCY_LEXICON, DISPLAY_NAME_MISMATCH, NEW_SENDER_FINANCIAL_ASK) against a fixture corpus of known-phishing and known-benign headers/bodies.
- Golden-set regression suite (per [[adr-019-testing-strategy]]) with a labeled set of real phishing samples (e.g., from public phishing corpora such as APWG eCrime feeds, sanitized) — target ≥95% recall on high-confidence native/rule-gated cases, tracked separately from the ambiguous-band precision/recall.
- Chaos test: verify that a crafted LLM response with an extreme `phishing_likelihood` cannot alone cross the final gate threshold without rule-derived symbols also contributing — i.e., prove the "LLM cannot be sole determinant" invariant holds under adversarial LLM output.
- Quarterly review of the brand-domain and lookalike reference lists against a sample of production near-miss cases (score 15-19 band) to catch drift.
