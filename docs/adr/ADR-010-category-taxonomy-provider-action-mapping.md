# ADR-010: Category Taxonomy & Provider Action-Mapping Table

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: taxonomy, data-model, gmail, microsoft-graph

## Context

The research report's Executive Summary defines the target taxonomy as eleven categories: "newsletters, job postings, social notifications, ecommerce/receipts, sales & deals, LinkedIn notifications, meeting cancellations, needs-reply, priority/VIP, phishing, and personal contacts." §3's "Unified Approach" (point 1) proposes representing this as "a provider-agnostic taxonomy (e.g., `category: newsletter | job_posting | social | ecommerce_receipt | promo_deal | linkedin | meeting_cancelled | needs_reply | priority_vip | phishing | personal`)" and (point 2) an "action translator per provider" mapping each internal category to a Gmail label vs an Outlook Category. §2 notes Outlook categories have a hard **25-color** limit with unlimited names per color and an **immutable `displayName`** once created (rename = delete+recreate), which directly constrains how the taxonomy can safely evolve post-launch. §1 notes Gmail's **500 user-created label** cap, which is not a binding constraint for an 11-category taxonomy but matters if per-category sub-labels or per-tenant custom labels are added later.

This ADR is the single source of truth for the taxonomy's canonical definition and its concrete mapping to both providers' native primitives, referenced by ADR-001 (pipeline), ADR-002 (adapters), ADR-008 (rule engine), and ADR-009 (LLM prompt).

## Decision

**Canonical taxonomy** (11 values, multi-label — a message may carry zero to N):

```typescript
type Category =
  | 'newsletter'
  | 'job_posting'
  | 'social'
  | 'ecommerce_receipt'
  | 'promo_deal'
  | 'linkedin'
  | 'meeting_cancelled'
  | 'needs_reply'
  | 'priority_vip'
  | 'phishing'
  | 'personal';

interface CategoryDefinition {
  category: Category;
  displayName: string;           // human-facing label text, versioned separately from `category` key
  llmRequired: boolean;          // per ADR-008 escalation gate
  defaultProviderAction: 'label_only' | 'label_and_optional_move';
}
```

**The `category` enum key is immutable post-launch**; only `displayName` may change freely. This directly accommodates Outlook's `displayName`-immutable-once-created constraint (§2) by never actually renaming the *provider-side* category — instead, `displayName` changes are implemented as a new Outlook category creation + old one's messages re-tagged + old one hidden from future use, orchestrated by ADR-021's migration tooling (batch 012-022), never a live in-place rename attempt against Graph's API.

**Full action-mapping table**:

| Internal Category | `llmRequired` | Gmail Label | Outlook Category (color slot) | Default Action |
|---|---|---|---|---|
| `newsletter` | no | `Triage/Newsletter` | "Newsletter" (color 1) | label_only |
| `job_posting` | no | `Triage/Job Posting` | "Job Posting" (color 2) | label_only |
| `social` | no | `Triage/Social` | "Social" (color 3) | label_only |
| `ecommerce_receipt` | no | `Triage/Receipt` | "Receipt" (color 4) | label_only |
| `promo_deal` | no | `Triage/Promo` | "Promo" (color 5) | label_and_optional_move (user-opt-in archive folder) |
| `linkedin` | no | `Triage/LinkedIn` | "LinkedIn" (color 6) | label_only |
| `meeting_cancelled` | no | `Triage/Meeting Cancelled` | "Meeting Cancelled" (color 7) | label_only |
| `needs_reply` | **yes** | `Triage/Needs Reply` | "Needs Reply" (color 8) | label_only (never moved — must stay visible in inbox) |
| `priority_vip` | no (ADR-011 is graph-based, not content-LLM) | `Triage/VIP` | "VIP" (color 9, also feeds `inferenceClassificationOverride`) | label_only |
| `phishing` | partial (deterministic gate first, LLM only for residual per ADR-008) | `Triage/Phishing Warning` | "Phishing Warning" (color 10) | label_only (never auto-moved/deleted — user must review) |
| `personal` | no | `Triage/Personal` | "Personal" (color 11) | label_only |

11 categories consume 11 of Outlook's 25 available color slots (§2), leaving headroom for future categories without hitting the hard limit. Gmail label count (11, potentially prefixed under one `Triage/` parent) is far under the 500-label cap (§1).

**Label/category provisioning**: On first classification action for a newly onboarded account, the provider adapter (ADR-002) ensures all 11 labels/categories exist (idempotent `labels.create`/master-category-list check), caching the resulting provider-specific IDs in a `provider_category_map` table so subsequent applies don't re-resolve by name every time:

```sql
CREATE TABLE provider_category_map (
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  category TEXT NOT NULL,          -- Category enum value
  provider_native_id TEXT NOT NULL, -- Gmail label ID or Outlook category displayName
  PRIMARY KEY (account_id, category)
);
```

**Safety-critical categories never auto-archive/delete**: `needs_reply` and `phishing` are explicitly excluded from any auto-move/auto-archive action (`defaultProviderAction` never includes a folder move for these two) — a false-positive phishing or needs-reply miscategorization must never remove the message from the user's normal inbox view; label-only ensures the user still sees it in Primary/Focused Inbox and can react.

## Consequences

### Positive
- Single enum + mapping table is the one place taxonomy meaning is defined; every other ADR (001, 002, 008, 009) references this rather than re-deriving category semantics.
- `displayName`-immutability accommodation (never live-renaming Outlook categories) avoids a whole class of Graph API errors and data-loss risk from attempting an unsupported rename.
- Explicit non-archival policy for `needs_reply`/`phishing` is a safety backstop independent of classification accuracy — even a wrong LLM verdict cannot cause a needs-reply or phishing message to disappear from the inbox.

### Negative
- 11 fixed color slots on Outlook leaves only 14 of 25 remaining for any future category additions or per-tenant custom categories — a real ceiling to plan around (ADR-021, batch 012-022, must address what happens if a tenant wants custom categories beyond this fixed 11).
- The provisioning step (ensure-labels-exist on first use) adds an extra round-trip/latency on account onboarding, and must handle partial-failure (e.g., 6 of 11 labels created before an error) idempotently.

### Neutral
- `displayName` text (the human-facing string) is left as a product/localization decision outside this ADR's scope — this ADR fixes only the stable `category` enum keys and their provider-mapping behavior.

## Links
- Related: ADR-001 (CategoryVerdict.category references this enum), ADR-002 (action-translator table implemented here is consumed by the adapters), ADR-008 (rule engine targets these categories), ADR-009 (LLM prompt taxonomy definitions), ADR-011 (priority_vip scoring detail), ADR-021 (taxonomy versioning/migration strategy for future category changes, batch 012-022)

## Verification

- Schema/enum lint: CI check that `Category` values used across ADR-008 rules, ADR-009 prompts, and `provider_category_map` never reference a value outside this canonical 11-item enum.
- Onboarding integration test: fresh account provisioning creates all 11 labels/categories idempotently, re-running provisioning twice produces no duplicates.
- Manual QA checklist item: confirm `needs_reply` and `phishing` labeled messages remain visible in Primary/Focused Inbox after classification (never silently archived), for both providers.
