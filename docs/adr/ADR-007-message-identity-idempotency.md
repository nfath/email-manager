# ADR-007: Internal Message Identity & Idempotency Key Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: data-model, idempotency, deduplication

## Context

Per §3 point 4 of the research report: "Both APIs are eventually-consistent and message IDs differ between IMAP/REST/native access. The system needs internal message identity (e.g., hash of Message-ID header + account) mapped to provider-specific IDs in a local datastore. This is also where 'already classified, don't reclassify' state and user corrections/feedback live." §7 Stage 6 reiterates this as the "internal message-identity table: Message-ID hash ↔ provider IDs" and notes it must record "per-category confidence + which layer decided it (rule vs LLM) for auditability/retraining."

This is compounded by ADR-005's dual-delivery-by-design ingestion (push + reconciliation sweep can both discover the same message) and ADR-006's batch/retry semantics (a batch apply may be retried after a partial/ambiguous failure, per Graph's nested-429 issue in §2) — both require the apply layer (ADR-001 Stage 5, ADR-002) to be safely re-runnable without double-labeling or corrupting state.

## Decision

**Identity key**: The internal message identity is a deterministic hash of `(tenant_id, account_id, normalized Message-ID header)`:

```typescript
function internalMessageId(tenantId: string, accountId: string, messageIdHeader: string): string {
  // normalize per RFC 5322: strip surrounding <>, lowercase domain part only
  const normalized = normalizeMessageIdHeader(messageIdHeader);
  return sha256Hex(`${tenantId}:${accountId}:${normalized}`);
}
```

`Message-ID` is used (not the Gmail `id`/Graph `id` provider fields) because those provider IDs are opaque, differ across API surfaces the same message might be accessed through (§3 point 4 explicitly calls out IMAP/REST/native divergence), and are not guaranteed stable across all access patterns, whereas `Message-ID` is an RFC 5322 header intended to be globally unique per message. Messages lacking a `Message-ID` header (rare, some malformed/legacy mail) fall back to a hash of `(tenant_id, account_id, From, Date, Subject, first-2KB-of-body)` as a best-effort synthetic identity, flagged with an `identity_synthetic: true` marker for lower-confidence dedup.

**Identity/state table**:
```sql
CREATE TABLE messages (
  internal_message_id TEXT PRIMARY KEY,
  tenant_id UUID NOT NULL,
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  provider_message_id TEXT NOT NULL,     -- current known provider ID (Gmail id / Graph id)
  message_id_header TEXT,
  identity_synthetic BOOLEAN NOT NULL DEFAULT FALSE,
  first_seen_at TIMESTAMPTZ NOT NULL,
  last_classified_at TIMESTAMPTZ,
  classification_state JSONB NOT NULL,   -- CategoryVerdict[] per ADR-001, incl. decidedBy + ruleIds
  apply_state TEXT NOT NULL DEFAULT 'pending' CHECK (apply_state IN ('pending','applied','failed')),
  apply_attempt_count INT NOT NULL DEFAULT 0,
  last_apply_error TEXT,
  UNIQUE (account_id, provider_message_id)
);
```

**Idempotency guarantees**:
1. **Ingestion dedup**: Before a discovered message enters Stage 2 (ADR-001), the pipeline computes `internal_message_id` and checks for an existing row. If found with `last_classified_at` set and no user correction since, the message is skipped (already classified) rather than reprocessed — this is what makes push+sweep double-discovery (ADR-005) safe by construction.
2. **Apply-layer idempotency**: `apply_state` transitions `pending → applied` only after the provider adapter (ADR-002) confirms success. A retried apply (e.g., after a nested-429 partial batch failure, ADR-006) re-reads `classification_state` and re-issues the *same* verdict set — because Gmail label-add and the Outlook category "ensure present" semantics (ADR-002) are themselves idempotent, re-application is always safe, never double-applies or duplicates labels.
3. **Provider ID churn**: If a provider ID changes (rare, but possible after certain move/copy operations) the `UNIQUE (account_id, provider_message_id)` constraint combined with a lookup-by-`message_id_header` fallback allows the row to be updated in place rather than creating a duplicate identity — a reconciliation job periodically verifies `provider_message_id` still resolves and updates it if the provider has re-assigned it.
4. **User-correction supersedes cache**: When a user manually recategorizes a message (feeding ADR-013's feedback loop, batch 012-022), `classification_state` is updated with a `decidedBy: 'user_correction'` entry and `last_classified_at` is intentionally NOT treated as "already classified, skip" for the purpose of future re-processing — corrections are terminal for that message unless the user changes it again, and are excluded from automatic reclassification sweeps.

**Batch reapplication safety**: `batchApplyCategories` (ADR-002) is called with a full `classification_state` snapshot per message, not a delta — so a retried batch after partial failure re-sends complete state for every message in the batch, relying on provider-side idempotent semantics rather than trying to compute a diff of "what actually got applied vs what didn't," which the nested-429 problem (§2) makes unreliable to determine precisely from the response alone.

## Consequences

### Positive
- Push+sweep double-discovery (ADR-005) and batch-retry-after-partial-failure (ADR-006) are both handled by the same simple mechanism: deterministic identity + idempotent re-apply, rather than needing separate special-case logic for each.
- `Message-ID`-based identity survives provider ID churn and is stable across Gmail/IMAP/Graph access patterns per §3 point 4.
- Full audit trail per message (`classification_state` as an array of verdicts with `decidedBy`) directly supports the feedback/retraining loop (ADR-013, batch 012-022).

### Negative
- Synthetic identity fallback (for messages missing `Message-ID`) is inherently lower-confidence and could theoretically collide or fail to dedupe correctly for near-identical bulk mail lacking headers — accepted as a rare-edge-case limitation, monitored via `identity_synthetic` rate metrics.
- Storing full classification state as JSONB trades some query-ability (can't easily SQL-filter "all messages where rule X fired" without JSONB operators/indexes) for schema flexibility as the taxonomy evolves (ADR-021, batch 012-022) — mitigated with a GIN index on `classification_state` for the fields expected to be queried (category, decidedBy).

### Neutral
- This ADR fixes the identity/idempotency mechanism but not the taxonomy versioning strategy — see ADR-021 (batch 012-022) for how `classification_state` shape changes are migrated over time.

## Links
- Related: ADR-001 (classification_state populated by pipeline stages), ADR-002 (idempotent apply semantics this design relies on), ADR-003 (tenant_id scoping on this table), ADR-005 (dedup safety net for push+sweep double-discovery), ADR-006 (batch retry safety relies on idempotent reapplication), ADR-013 (feedback loop reads/writes classification_state, batch 012-022), ADR-021 (schema/taxonomy versioning for classification_state shape, batch 012-022)

## Verification

- Property-based test: feed the same raw message through the pipeline twice (simulating push+sweep double discovery) and assert exactly one row is created and Stage 2-5 execute only once.
- Chaos test: truncate a batch apply response mid-parse (simulating partial Graph batch failure) and assert retry produces the same final provider state as a clean single-pass apply (no duplicate labels, no missing labels).
- Data-quality metric: track `identity_synthetic` rate in production; alert if it exceeds an expected baseline (indicates unexpected upstream Message-ID stripping, e.g. by a corporate gateway).
