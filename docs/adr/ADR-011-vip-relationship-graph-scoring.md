# ADR-011: VIP / Relationship-Graph Scoring Design

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: vip-scoring, relationship-graph, classification, gmail, microsoft-graph

## Context

Per §4's signal table, "Priority / VIP" is resolved primarily through non-content signals: "Manual VIP list (user-curated) as deterministic override; reciprocity/frequency graph (sent-to-them count × replied-to-them count, normalized); calendar co-attendance (shared meetings); Outlook's `inferenceClassificationOverride` (force sender→Focused)." The table explicitly notes LLM involvement is "less relevant — fundamentally a graph/frequency computation over user's own mail history, not content classification," distinguishing this category architecturally from content-driven categories like `needs_reply` or `phishing`.

§9 gap #5 flags "priority/VIP" alongside "needs-reply" as having only "academic/patent prior art (general shape) than a spec to copy; expect to tune thresholds against real usage data" — medium-low confidence on the exact scoring formula, though high confidence on the general signal types involved.

§1 notes the Gmail-side data source: the People API's `otherContacts.list`/`otherContacts:search` (scope `contacts.other.readonly`) surfaces auto-detected contacts, read-only, supporting reciprocity scoring; `contacts.readonly` accesses the explicit address book. §2 notes the Outlook-side lever: `inferenceClassificationOverride` can force a sender to Focused Inbox at the API level (up to 1,000 overrides per mailbox, keyed by SMTP address), but "an override only affects future mail; does not retroactively reclassify existing messages" — a hard constraint this design must accommodate rather than assume away.

## Decision

**VIP determination is a three-tier override cascade**, evaluated in this strict priority order (higher tiers short-circuit lower ones):

1. **Manual VIP list** (deterministic, user-curated) — a `vip_senders` table the user explicitly manages; always wins regardless of computed score.
2. **Reciprocity/frequency graph score** — computed from the user's own mail history (sent + received), described below.
3. **Calendar co-attendance signal** — a lesser-weighted contributing signal into the same score, not a separate override tier.

```sql
CREATE TABLE vip_senders (
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  sender_email TEXT NOT NULL,
  source TEXT NOT NULL CHECK (source IN ('manual', 'computed')),
  added_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (account_id, sender_email)
);

CREATE TABLE sender_reputation (
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  sender_email TEXT NOT NULL,
  sent_count_90d INT NOT NULL DEFAULT 0,       -- messages user sent TO this sender
  received_count_90d INT NOT NULL DEFAULT 0,   -- messages FROM this sender
  replied_count_90d INT NOT NULL DEFAULT 0,    -- received messages the user replied to
  calendar_coattendance_count_90d INT NOT NULL DEFAULT 0,
  reciprocity_score REAL NOT NULL DEFAULT 0,   -- computed, see formula below
  last_computed_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (account_id, sender_email)
);
```

**Reciprocity score formula** (initial default, explicitly flagged as tunable per §9 gap #5 — not treated as a validated constant):

```
reply_rate = replied_count_90d / max(received_count_90d, 1)
bidirectional_factor = min(sent_count_90d, received_count_90d) / max(sent_count_90d, received_count_90d, 1)
raw_score = (reply_rate * 0.5) + (bidirectional_factor * 0.3) + (log10(calendar_coattendance_count_90d + 1) * 0.2)
reciprocity_score = clamp(raw_score, 0, 1)
```

A sender crosses into computed-VIP status when `reciprocity_score >= vip_threshold` (default 0.7, tunable), at which point they are inserted into `vip_senders` with `source = 'computed'` — distinct from `source = 'manual'` so a user can see and override computed VIP designations without losing their own explicit list, and so the feedback loop (ADR-013, batch 012-022) can specifically down-weight or reset computed entries without touching manual ones.

**90-day rolling window**: reputation counts are computed over a rolling 90-day window (not all-time), recomputed on a scheduled batch job (daily), so relationship changes (e.g., a contact going quiet, or a new frequent collaborator emerging) are reflected without manual intervention, and old high-volume-then-stopped relationships decay out naturally.

**Calendar co-attendance signal**: sourced from the same OAuth-granted calendar scope where available (Google Calendar API / Graph `/me/calendar`); counts distinct meetings in the last 90 days where both the user and the sender were attendees. This is treated as a supplementary signal only (weight 0.2 in the formula above) since calendar scope may not always be granted (degrades gracefully to the two-term formula if unavailable, consistent with ADR-004's scope-degradation pattern).

**Provider action mapping (ties into ADR-010)**:
- **Gmail**: apply `Triage/VIP` label via the standard action translator (ADR-002/010) — no Gmail-native "focused inbox" equivalent exists, so labeling is the sole mechanism.
- **Outlook**: apply the `Triage/VIP` category (ADR-010) AND additionally create an `inferenceClassificationOverride` for that sender's SMTP address to force future mail to Focused Inbox — implemented as a best-effort supplementary action, not the primary signal-of-record (the category label remains authoritative for the internal taxonomy state). Because the override "does not retroactively reclassify existing messages" (§2), this design explicitly does NOT attempt to re-fetch and re-classify a sender's historical mail into Focused when they newly become VIP — that would require Graph write operations beyond the override's documented scope and is out of scope for V1 (noted here as an accepted limitation, not silently ignored).
- **Override budget management**: Outlook's 1,000-override-per-mailbox cap (§2) is tracked in `sender_reputation`/`vip_senders` application logic — if a mailbox approaches the cap, computed (non-manual) VIP overrides are evicted first (lowest `reciprocity_score` among `source='computed'` entries), preserving all manually-added VIPs unconditionally.

## Consequences

### Positive
- Manual-list-always-wins ordering gives users a reliable, predictable override regardless of any scoring quirks — critical for trust in a "who is important to me" feature.
- Reciprocity formula uses only signals the user's own mailbox already provides (no third-party data), keeping this fully within the existing OAuth scope model (ADR-004) and avoiding new privacy surface area.
- Explicit handling of the Outlook override's future-only limitation avoids a design that silently over-promises retroactive reclassification it cannot deliver.
- Computed vs manual VIP distinction lets the feedback/correction loop (ADR-013, batch 012-022) safely adjust algorithmic guesses without ever touching user-entered data.

### Negative
- The reciprocity formula's weights (0.5/0.3/0.2) and threshold (0.7) are initial estimates without a validated reference dataset (§9 gap #5) — real tuning against user corrections is required and accuracy risk exists pre-tuning.
- 90-day rolling recomputation is a daily batch job over potentially large sent/received history per account — needs to be resource-budgeted per mailbox (ties into ADR-006's per-mailbox queue model) rather than a naive full-history scan.
- Outlook's 1,000-override cap requires active budget-management logic (eviction policy) that adds operational complexity most competitors' simpler VIP-list features don't need to handle.

### Neutral
- Calendar co-attendance is optional/degradable by design (missing scope does not break VIP scoring, just removes one weighted term) — consistent with the graceful-degradation pattern established in ADR-004.

## Links
- Related: ADR-004 (OAuth scope degradation pattern reused for optional calendar signal), ADR-008 (`SENDER_IN_ADDRESS_BOOK` rule feeds into but is distinct from full reciprocity scoring), ADR-010 (VIP category provider-action mapping), ADR-002 (Outlook override is an adapter-level action alongside category apply), ADR-013 (feedback loop adjusts computed-VIP threshold/formula weights over time, batch 012-022)

## Verification

- Unit tests for the reciprocity formula against synthetic sent/received/reply/calendar count fixtures, asserting monotonicity (more reciprocal interaction never decreases score).
- Integration test: verify manual VIP entries are never evicted by the override-budget-management eviction policy even under a simulated near-1,000-override mailbox.
- Scheduled-job resource test: measure daily reputation-recompute job duration/cost per mailbox size tier, ensure it fits within the per-mailbox worker budget (ADR-006) without starving classification throughput.
- Post-launch analytics: track computed-VIP precision (via user corrections, ADR-013 batch 012-022) as the primary signal for retuning `vip_threshold` and formula weights, per §9 gap #5's explicit call for empirical tuning.
