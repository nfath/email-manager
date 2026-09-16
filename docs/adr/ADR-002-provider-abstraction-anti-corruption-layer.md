# ADR-002: Provider Abstraction & Anti-Corruption Layer for Gmail vs Microsoft Graph

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: architecture, integration, gmail, microsoft-graph, anti-corruption-layer

## Context

Per §3 of the research report, Gmail and Microsoft Graph expose fundamentally different organizing primitives: "Gmail's model: label-based, many-to-many tags on an immutable message" vs "Outlook's model: folder+category hybrid, message lives in exactly one folder plus zero-or-more categories." Gmail also exposes read-only ML-derived `CATEGORY_*` system labels (§1) while Graph exposes `inferenceClassification`/Focused-Inbox overrides (§2) — structurally similar "free prior" signals but with different mutability rules (Gmail categories are mutually exclusive and read-only; Outlook categories are `displayName`-immutable once created, up to 25 colors, and require `MailboxSettings.ReadWrite` — an application permission — to touch other users' mailboxes).

Without an abstraction boundary, classification logic (ADR-001, ADR-008) would end up littered with `if (provider === 'gmail')` branches, and provider API quirks (batch size caps, nested-429s in Graph batch envelopes, Gmail's 500-label hard limit) would leak into business logic. The report's §3 "Unified Approach" section explicitly recommends a "single internal taxonomy" plus an "action translator per provider."

## Decision

Implement a strict anti-corruption layer (ACL) with three components:

**1. Internal taxonomy** (shared, provider-agnostic) — defined fully in ADR-010; referenced here only as the contract surface:
```typescript
type Category =
  | 'newsletter' | 'job_posting' | 'social' | 'ecommerce_receipt'
  | 'promo_deal' | 'linkedin' | 'meeting_cancelled' | 'needs_reply'
  | 'priority_vip' | 'phishing' | 'personal';
```

**2. Provider adapter interface** — every provider integration implements this interface; no other code in the system imports the Gmail SDK or Graph SDK directly:

```typescript
interface MailProviderAdapter {
  // Stage 1 (ADR-001): pull/normalize raw messages into RawMessage
  fetchMessage(providerMessageId: string): Promise<RawMessage>;
  listChangedSince(cursor: ProviderCursor): Promise<{ messages: RawMessage[]; nextCursor: ProviderCursor }>;

  // Stage 5 (ADR-001): apply a set of CategoryVerdicts as provider-native actions
  applyCategories(providerMessageId: string, verdicts: CategoryVerdict[]): Promise<void>;
  batchApplyCategories(batch: Array<{ providerMessageId: string; verdicts: CategoryVerdict[] }>): Promise<BatchApplyResult>;

  // read-through of provider-native "free" classification signal (§3 point 3)
  readNativeSignal(providerMessageId: string): Promise<NativeSignal>; // CATEGORY_* or inferenceClassification
}
```

**3. Action translator table** (per-provider, config-driven, not hardcoded per message):

| Internal Category | Gmail Action | Outlook Action |
|---|---|---|
| Any of the 11 | Ensure label exists (cache label ID after first `labels.create`, 5qu), apply via `batchModify` (≤1,000 IDs/call) | Ensure Outlook Category exists in master category list, apply via batched `PATCH` (≤20 sub-requests/batch, 4MB cap) |
| `newsletter`, `promo_deal` (optional archive-out) | Add label only (Gmail has no true folders — label removal ≈ Outlook folder move) | Optionally move to dedicated subfolder if user opts in; otherwise category-only to avoid fighting Focused Inbox (§3 point 2) |
| Native signal read | `CATEGORY_PROMOTIONS`/`CATEGORY_SOCIAL`/etc. read via `labels` field on `messages.get`, fed into Stage 2 signal extraction as a free prior — never authoritative alone | `inferenceClassification` read via message resource, same treatment |

Multiple categories per message map naturally to multiple labels (Gmail) or multiple categories (Outlook) — both providers natively support many-to-one (§3 point 2), so the ACL never needs to pick a single "winning" category for storage, only for any provider surface that is exclusive (e.g., Outlook folder placement, which is reserved only for explicit user-configured archive rules, never as the primary output channel).

**Idempotency contract**: `applyCategories`/`batchApplyCategories` must be safe to call repeatedly with the same verdict set (Gmail `batchModify` label-add is a no-op if already applied; Outlook category PATCH must be implemented as "ensure present" set-union semantics, not blind overwrite, to avoid clobbering categories added by other tools). This underpins the idempotency design in ADR-007.

**Failure isolation**: Adapters must translate provider-specific errors (Gmail 429/403, Graph nested-429-in-200-envelope per §2) into a shared internal error taxonomy (`RateLimited`, `AuthExpired`, `PermissionDenied`, `NotFound`, `TransientError`) so Stage 5 retry/backoff logic (ADR-006) is provider-agnostic.

## Consequences

### Positive
- Classification/scoring logic (ADR-001, ADR-008, ADR-010) never needs to know which provider a message came from.
- New providers (e.g., IMAP-generic, future Yahoo/iCloud) can be added by implementing one interface, without touching the core pipeline.
- Provider-specific quirks (nested 429s, label immutability, batch caps) are contained and tested in one place per provider.

### Negative
- The ACL itself is nontrivial engineering investment — must be maintained in lockstep with both providers' API evolutions (Graph in particular changes admin/permission mechanics frequently, per §2 confidence notes).
- Least-common-denominator risk: provider-unique features (e.g., Outlook's Focused Inbox override, which only affects *future* mail per §2) don't map cleanly to the shared interface and need explicit escape hatches rather than being silently dropped.

### Neutral
- The decision to treat platform-native ML classification (`CATEGORY_*`, `inferenceClassification`) as a Stage-2 signal rather than the source of truth is consistent with §3 point 3 and reduces provider lock-in on classification quality.

## Links
- Related: ADR-001 (pipeline stages this ACL bounds), ADR-005 (ingestion sync uses `fetchMessage`/`listChangedSince`), ADR-006 (rate-limit/backpressure consumes the shared error taxonomy), ADR-007 (idempotency contract depends on identity design), ADR-010 (internal taxonomy and full action-mapping table), ADR-016 (deployment topology, batch 012-022, may run adapters as separate deployable units per provider)

## Verification

- Contract tests run identically against both adapters (a shared test suite parameterized by provider) asserting: apply-then-reapply is idempotent, native-signal read never blocks on missing permission, batch size limits are enforced client-side before calling the provider (fail fast rather than relying on the provider to reject oversized batches).
- Static lint rule: forbid importing `googleapis`/Gmail SDK or `@microsoft/microsoft-graph-client` outside the `adapters/gmail` and `adapters/graph` directories respectively.
- Manual sandbox verification against live Gmail and Graph developer sandboxes (ADR-019, batch 012-022) before each adapter release.
