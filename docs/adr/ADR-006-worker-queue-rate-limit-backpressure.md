# ADR-006: Per-Mailbox Worker/Queue Model & Rate-Limit Backpressure Strategy

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: scalability, rate-limiting, queueing, backpressure

## Context

Per §1, Gmail enforces a **hard per-user ceiling of 250 quota units/second** and a max of **50 concurrent in-flight requests per mailbox**, on top of per-project (1,200,000 units/min) and per-user (15,000 units/min) quotas. Per §2, Graph's batch requests are capped at **20 sub-requests** and **4MB**, with a documented default of only **4 individual requests from a batch dispatched to the Outlook backend concurrently, regardless of target mailbox** (the "MailboxConcurrency limit"), plus a general cap of **10,000 requests/10min/app** and 429 responses carrying `Retry-After`. Critically, §2 flags a specific pitfall: "a throttled (429) sub-request is nested inside a 200 batch envelope, so naive code checking only the outer status will silently swallow failures."

§3 point 5 draws the key architectural conclusion: "Gmail's per-user 250 qu/s and Graph's ~4 req/s effective concurrency per mailbox are mailbox-scoped — horizontal scale-out (many users) is naturally parallel-safe. A single very active mailbox is the bottleneck, arguing for a per-mailbox worker/queue model with backoff on 429, not global concurrency limiting."

## Decision

**Queue topology**: One logical work queue per `account_id` (mailbox), not a single global queue. Each queue is consumed by a token-bucket-limited worker that enforces the provider's per-mailbox limits independently of all other mailboxes' queues:

```typescript
interface MailboxRateLimiter {
  accountId: string;
  provider: 'gmail' | 'graph';
  // Gmail: token bucket refilling at 250 units/sec, capacity 250, cost per operation from §1 cost table
  // Graph: concurrency semaphore of 4 concurrent sub-requests, additional 10,000/10min/app-wide budget shared across all Graph accounts
  tryAcquire(cost: number): Promise<boolean>;
  onThrottled(retryAfterSeconds?: number): void; // triggers exponential backoff for this account's queue only
}
```

- **Gmail limiter**: token-bucket, refill rate 250 units/sec (§1 hard ceiling), request costs read from a static cost table (`messages.get`=20u, `drafts.create`=10u, `labels.create`=5u, `batchModify` costed per Google's published batch pricing) — never issue a request whose cost would exceed the current bucket balance; queue the request instead of firing and hoping.
- **Graph limiter**: bounded semaphore of concurrency 4 per mailbox (matching the documented MailboxConcurrency default) plus a shared app-wide sliding-window counter capped under 10,000 requests/10min, shared across all Graph accounts in the deployment (this one is NOT per-mailbox — it's an app-wide ceiling per §2 — so it is implemented as a separate, cross-account limiter distinct from the per-mailbox semaphore).

**Nested-429 handling (Graph-specific correctness requirement)**: The batch-response parser MUST inspect each sub-response's individual status code inside a 200-status batch envelope, never trusting the outer HTTP status alone. Any sub-response with status 429 is treated as a per-item throttle: that specific item is re-queued with backoff (using its `Retry-After` if present, else a default backoff schedule), while sibling items in the same batch that succeeded are committed immediately — a single throttled item in a batch of 20 must not cause the other 19 to be needlessly retried.

**Backoff on 429/403 (rate-limit-class errors)**: Exponential backoff with jitter, scoped to the specific mailbox's queue only (per §3 point 5 — a rate-limited mailbox does not throttle unrelated mailboxes' queues): base delay 1s, doubling to a cap of 5 minutes, full jitter (random between 0 and computed delay) to avoid synchronized retry storms across many workers. After 6 consecutive backoff cycles for one mailbox, the queue is marked degraded and surfaced to observability (ADR-015, batch 012-022) rather than retried indefinitely — protects against a persistently misconfigured or revoked account spinning forever.

**Concurrency and scale-out**: Queue workers are horizontally scalable — many worker processes/pods can each own a shard of mailbox queues (e.g., consistent-hash `account_id` to worker), since mailbox-level rate limits are independent of each other (§3 point 5). This is the system's primary horizontal scaling lever: adding tenants/mailboxes scales linearly by adding workers, with no cross-mailbox contention by design.

**Priority within a mailbox's queue**: Within one mailbox's queue, prioritize (a) user-interactive actions (e.g., a manual reclassification triggering immediate re-apply) over (b) push-triggered new-mail classification over (c) reconciliation-sweep-discovered backlog — implemented as three priority sub-queues per mailbox, drained in strict priority order with starvation protection (backlog sub-queue guaranteed some minimum throughput even under sustained interactive load).

## Consequences

### Positive
- No cross-mailbox head-of-line blocking: one throttled or misbehaving mailbox cannot starve other tenants' processing (directly satisfies the multi-tenant isolation goals of ADR-003 at the queueing layer).
- Correctly parsing nested Graph batch statuses avoids a documented, easy-to-miss class of silent data loss (messages appearing "processed" when they were actually throttled).
- Horizontal scale-out story is simple and linear: more mailboxes = more queues = more workers, without a global bottleneck.

### Negative
- Per-mailbox token-bucket/semaphore state must itself be stored somewhere workers can share (e.g., Redis) if more than one worker process could ever serve the same mailbox concurrently — introduces a shared rate-limiter state dependency, not purely in-process.
- Three-tier priority sub-queues per mailbox add scheduling complexity versus a single FIFO queue.
- App-wide Graph 10,000/10min budget is a shared resource across all mailboxes on that Graph app registration — at very large tenant counts this could become a real ceiling requiring multiple app registrations (multi-tenant Graph app partitioning), which is not addressed further here and should be revisited if usage approaches that limit.

## Links
- Related: ADR-001 (Stage 5 apply-layer consumes this queue model), ADR-002 (adapters raise the shared error taxonomy this backoff logic keys off), ADR-003 (per-mailbox queue isolation reinforces tenant isolation), ADR-005 (both push- and sweep-discovered work enters these queues), ADR-016 (deployment topology/autoscaling of worker pools, batch 012-022)

## Verification

- Unit tests simulating a Graph batch response with mixed 200/429 sub-statuses; assert only the 429 items are re-queued and succeeded items are never retried.
- Load test: saturate one mailbox's queue to induce sustained 429s and assert sibling mailboxes' queues show no measurable latency increase.
- Token-bucket correctness test: replay a recorded sequence of Gmail operation costs and assert the limiter never permits cumulative cost to exceed 250 units in any 1-second sliding window.
- Production metric: per-mailbox queue depth and per-mailbox backoff-cycle count, alerting when a mailbox crosses the 6-consecutive-backoff degraded threshold (feeds ADR-015, batch 012-022).
