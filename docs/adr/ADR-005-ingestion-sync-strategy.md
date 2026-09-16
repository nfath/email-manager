# ADR-005: Ingestion Sync Strategy — Push + Reconciliation Sweep

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: ingestion, sync, gmail, microsoft-graph, reliability

## Context

Per §1, Gmail's push mechanism (`users.watch` + Cloud Pub/Sub) delivers `{emailAddress, historyId}` notifications, but the watch **expires after 7 days** with Google recommending **daily renewal**, and the report notes "because push is not 100% guaranteed, pair with periodic `history.list` calls as a reconciliation fallback for gaps." Per §2, Graph's webhook subscriptions are notably shorter-lived — **~4,230 minutes (~3 days)** — "notably shorter-lived than Gmail's 7-day watch," requiring more aggressive renewal (recommended every 24h), and Microsoft's own documented pattern explicitly combines change notifications (push) with delta queries (pull): "push tells you when to pull, delta tells you what changed cheaply" (§2). §7 Stage 1 and Stage 7 both describe this as the canonical "push-driven primary trigger + periodic reconciliation sweep as safety net" pattern, calling it "explicitly Microsoft-recommended and implicitly Google's own design."

Without an explicit renewal schedule and reconciliation cadence, watches/subscriptions will silently expire (Gmail after 7 days, Graph after ~3 days) and the system will stop receiving new-mail notifications with no application-visible error — a silent-failure mode the design must actively guard against.

## Decision

**Push layer**:
- **Gmail**: `users.watch` registered per mailbox against a per-tenant (or shared, access-controlled) Cloud Pub/Sub topic. A renewal cron job runs **daily** (per Google's recommendation, well inside the 7-day expiry) and re-issues `watch` for every active account; renewal failures are retried with exponential backoff and alert if a watch is within 24h of expiry with no successful renewal (ADR-015 alerting, batch 012-022).
- **Graph**: Webhook subscription renewal cron runs **every 12 hours** (inside the ~4,230-minute/~3-day expiry, more aggressive than Gmail given the shorter window per §2) using the `PATCH /subscriptions/{id}` renewal call. Subscriptions failing renewal twice consecutively trigger full subscription re-creation rather than continued renewal attempts (guards against a subscription stuck in a bad state).
- Push notification payloads (`{emailAddress, historyId}` for Gmail; change-notification envelope for Graph) are treated as **wake-up signals only** — they trigger a pull (`history.list` / delta query), never carry authoritative message content themselves, avoiding trust in payload integrity beyond "something changed, go check."

**Pull/reconciliation layer**:
- **Gmail**: `history.list` calls from the last-processed `historyId`, cursor persisted per account in `sync_state` table. On push notification arrival, immediately pull from cursor. Independently, a reconciliation sweep runs **every 15 minutes** per active account (config-tunable) to catch missed pushes, calling `history.list` regardless of whether a push was recently received.
- **Graph**: `GET /me/mailFolders/inbox/messages/delta` using persisted `@odata.deltaLink`, same dual-trigger pattern (on webhook arrival + independent 15-minute sweep).
- **Cursor recovery**: If a `history.list`/delta call returns an expired-cursor error (history ID too old / deltaLink invalid — both providers can invalidate cursors after extended gaps), fall back to a full resync bounded to a configurable lookback window (default 30 days) rather than an unbounded full-mailbox scan, to bound both latency and Gmail quota cost (§1: `messages.get` at 20 units each — a 10k-message full resync is 200,000 units, within per-minute limits per §1 but only if throttled deliberately).

**Sync state schema**:
```sql
CREATE TABLE sync_state (
  account_id UUID PRIMARY KEY REFERENCES accounts(account_id),
  provider TEXT NOT NULL,
  push_subscription_id TEXT,        -- Pub/Sub watch token or Graph subscription ID
  push_expires_at TIMESTAMPTZ NOT NULL,
  pull_cursor TEXT,                 -- historyId or @odata.deltaLink
  last_reconciliation_at TIMESTAMPTZ,
  last_push_received_at TIMESTAMPTZ,
  consecutive_renewal_failures INT NOT NULL DEFAULT 0
);
```

**Ordering guarantee**: Both push and reconciliation paths feed the same per-mailbox work queue (ADR-006); the message-identity layer (ADR-007) ensures a message discovered twice (once via push-triggered pull, once via sweep) is deduplicated before entering Stage 2 of the classification pipeline (ADR-001), so double-delivery is safe by construction rather than requiring the sync layer itself to dedupe.

## Consequences

### Positive
- No silent sync death: renewal cadence is comfortably inside both providers' expiry windows, with alerting on renewal failure before expiry is reached.
- Reconciliation sweep bounds the "missed push" failure mode to at most one sweep interval (15 minutes) of latency, not indefinite silence.
- Bounded-lookback resync avoids unbounded quota burn on cursor-invalidation edge cases.

### Negative
- Two renewal cadences (daily for Gmail, 12-hourly for Graph) and two reconciliation mechanisms (history.list vs delta query) mean the ingestion layer cannot be fully unified — some provider-specific scheduling logic is unavoidable (contained within the ADR-002 adapters, however).
- 15-minute reconciliation sweeps across many mailboxes require careful staggering (jitter) to avoid a thundering-herd of simultaneous API calls at sweep boundaries.

### Neutral
- The reconciliation interval (15 minutes) and Graph renewal cadence (12h) are configurable operational parameters, not hardcoded constants — expect tuning based on observed push reliability in production (no public data on Gmail/Graph push miss rates was found in the research, per §9 gap discussion adjacent to sync).

## Links
- Related: ADR-001 (ingestion feeds Stage 1), ADR-002 (adapters implement `fetchMessage`/`listChangedSince` used here), ADR-006 (per-mailbox queue receives both push- and sweep-triggered work), ADR-007 (dedup relies on identity design), ADR-015 (alerting on renewal-failure and push-miss-rate metrics, batch 012-022), ADR-017 (disaster recovery — cursor loss / bounded resync ties into RPO targets, batch 012-022)

## Verification

- Integration test against provider sandboxes (ADR-019, batch 012-022): register a watch/subscription, simulate expiry-adjacent renewal, assert no gap in coverage.
- Chaos test: deliberately drop N consecutive push notifications and assert the reconciliation sweep still surfaces the missed messages within one sweep interval.
- Alerting test: assert an alert fires when `consecutive_renewal_failures` crosses a threshold (e.g., 2) and when `push_expires_at` is within 24h with no successful renewal logged.
- Quota-usage dashboard tracking actual Gmail quota-unit consumption per reconciliation sweep against the 250 qu/s per-user ceiling (§1).
