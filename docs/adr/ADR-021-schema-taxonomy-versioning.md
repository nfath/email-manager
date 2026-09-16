# ADR-021: Schema / Taxonomy Versioning & Migration Strategy

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: taxonomy, schema-migration, versioning, backward-compatibility

## Context

Per §3 of the research report, the system's core abstraction is "a single internal taxonomy" of 11 categories (`newsletter | job_posting | social | ecommerce_receipt | promo_deal | linkedin | meeting_cancelled | needs_reply | priority_vip | phishing | personal`) that both providers' native primitives (Gmail labels, Outlook categories) are mapped onto. Per §2, Outlook categories have an important immutability constraint: **`displayName` is immutable once created — only `color` is updatable — so renaming a category requires delete+recreate**, which the report explicitly calls out as something to "plan taxonomy carefully" around. This is a direct, concrete constraint this ADR must design the taxonomy-versioning/migration process against, since a naive "just rename the category" approach is not available on the Outlook side.

Per §7's roadmap, the taxonomy is expected to evolve (V3 mentions "job-postings category refined from empirical sender-domain collection"), and per §9 research gap #1, ATS sending-domain lists require ongoing empirical refinement — meaning the taxonomy and its supporting reference data are not a one-time fixed schema but require an explicit, safe evolution mechanism from day one.

## Decision

### Taxonomy versioning model
- The 11-category taxonomy (and any future additions/splits) is defined in a single versioned schema artifact: `taxonomy_v{N}.json`, containing category keys, human-readable labels, and a stable `category_id` (immutable, never reused even if a category is later removed — e.g., `cat_007_needs_reply`) distinct from the human-readable label.
- Every classification record persisted in the state store ([[adr-007-message-identity-idempotency]]) stores the `taxonomy_version` it was classified under, alongside the `category_id` — never just the current label string — so historical classifications remain interpretable even after the taxonomy evolves.
- Introducing a **new category** is a minor version bump (`v1` → `v1.1`) and is backward compatible (old classifications remain valid, new messages can additionally receive the new category). **Renaming or splitting/merging** an existing category is a major version bump (`v1` → `v2`) since it changes the meaning of existing `category_id` mappings and requires a migration pass (see below).

### Handling Outlook's category-immutability constraint
Because an Outlook `outlookCategory.displayName` cannot be renamed in place (§2):
- The internal `category_id` is never directly equal to the Outlook `displayName` string exposed to the end user; instead, a per-tenant `category_id → outlook_category_displayName` mapping table is maintained ([[adr-010-category-taxonomy-action-mapping]]).
- When a taxonomy major-version change renames a category's human-readable label, the migration creates a **new** Outlook category (new displayName) and schedules a background re-tagging job to add the new category and remove the old one from previously-tagged messages within Graph's batch limits (§2: max 20 sub-requests/batch), rather than attempting an in-place rename that Outlook's API does not support.
- The old Outlook category is left in place (not deleted) for a configurable grace period (default 90 days) in case of rollback, then deleted via a scheduled cleanup job — avoiding an irreversible delete+recreate happening synchronously with the taxonomy migration itself.

### Gmail-side migration
- Gmail labels support rename via `labels.update` (§1 — labels API supports update), so a category rename maps directly to a label rename with no equivalent immutability constraint — migrations are asymmetric across providers by design, and the provider abstraction layer ([[adr-002-provider-abstraction-anti-corruption-layer]]) must expose this asymmetry (Gmail: in-place rename; Outlook: new-create + re-tag + deferred-delete) rather than force a lowest-common-denominator behavior on both.

### Migration execution
- A taxonomy migration is executed as an explicit, versioned migration script (analogous to a DB schema migration), reviewed and approved separately from routine rule-weight tuning changes ([[adr-018-tenant-admin-configuration-api]]'s distinction between routine config and structural taxonomy change).
- Migrations run per-tenant on a rolling basis (not a global instantaneous cutover), tracked in a `tenant_taxonomy_version` table, so different tenants can be on different taxonomy versions simultaneously during a rollout window — critical because Outlook-side re-tagging is a background job bounded by batch/throttle limits (§2) and cannot be instantaneous at scale.
- The rule engine ([[adr-008-deterministic-rule-scoring-engine]]) and LLM adjudication service ([[adr-009-llm-adjudication-service]]) both read `tenant_taxonomy_version` to select the correct category schema and prompt template version for that tenant, so mixed-version operation during rollout is a first-class supported state, not an edge case.

### State-store schema migration (general, non-taxonomy)
- Standard additive-first migration discipline: new columns are nullable/defaulted, backward-incompatible changes (column removal, type changes) require a two-phase deploy (add new, dual-write, backfill, cut over reads, remove old) — standard practice, included here for completeness since the taxonomy versioning above is a special case of this general discipline applied to a specific, constrained-by-provider-API domain.

## Consequences

### Positive
- Explicitly designing around Outlook's documented displayName-immutability constraint (§2) avoids a class of migration bugs (attempted in-place rename failing silently or erroring) that would only surface once a real rename was attempted in production.
- Per-tenant rolling migration with mixed-version support avoids forcing an instantaneous global cutover that Outlook's batch/throttle limits (§2) would make operationally risky at scale.
- Stable, immutable `category_id` decoupled from the human-readable label means historical audit/analytics data ([[adr-013-user-feedback-correction-loop]], [[adr-015-observability-slos]]) remains queryable and comparable across taxonomy versions.

### Negative
- Maintaining a grace-period-deferred-delete for old Outlook categories means end users may transiently see both old and new category tags during a migration window — a UX consideration that must be communicated (e.g., via release notes or the admin UI), not just an internal implementation detail.
- Mixed-taxonomy-version operation across tenants adds real complexity to the rule engine and LLM adjudication service, which must both be version-aware rather than assuming a single global taxonomy schema at all times.
- Major taxonomy version changes (rename/split/merge) are inherently higher-friction and slower to roll out than minor additions, by design — this is a deliberate tradeoff favoring safety over migration speed, consistent with the provider constraints, but should be communicated to product stakeholders as a real cost of category renames.

### Neutral
- The 90-day default grace period for deferred Outlook category deletion is an initial engineering estimate, not derived from a cited source, and may be tuned based on observed rollback-request patterns once the system has production history.

## Links
- Related: ADR-002 (provider abstraction — exposes provider-asymmetric migration behavior), ADR-007 (message identity — classification records store taxonomy_version), ADR-008 (rule engine — version-aware category schema), ADR-009 (LLM adjudication — version-aware prompt/taxonomy template), ADR-010 (category taxonomy/action mapping — category_id-to-provider-primitive mapping table), ADR-013 (feedback loop — corrections tied to category_id, not label), ADR-015 (observability — accuracy tracked per taxonomy version), ADR-018 (tenant admin API — distinguishes routine config change from structural taxonomy migration)

## Verification

- Migration dry-run test: simulate a major-version category rename against a fixture tenant with existing Outlook-tagged messages; assert the new category is created, messages are re-tagged within batch/throttle constraints, and the old category is retained (not deleted) through the grace period.
- Regression test: verify Gmail-side rename-in-place and Outlook-side create+retag+deferred-delete both converge to the same internal `category_id` state, exercised via the provider abstraction layer's shared interface.
- Mixed-version test: run the rule engine and LLM adjudication service against two fixture tenants pinned to different taxonomy versions simultaneously, assert no cross-contamination of category schema between them.
- Audit query test: confirm historical classification records remain queryable by `category_id` across a version boundary (i.e., a report spanning a taxonomy migration date does not silently drop or misattribute pre-migration records).
