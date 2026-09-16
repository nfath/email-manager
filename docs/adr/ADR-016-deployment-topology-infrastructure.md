# ADR-016: Deployment Topology & Infrastructure

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: infrastructure, deployment, autoscaling, multi-region, containerization

## Context

Per §3 of the research report, rate limits are mailbox-scoped (Gmail 250 quota units/sec/user; Graph ~4 concurrent requests/mailbox effective concurrency), meaning "horizontal scale-out (many users) is naturally parallel-safe. A single very active mailbox is the bottleneck, arguing for a per-mailbox worker/queue model with backoff on 429, not global concurrency limiting." This directly shapes the deployment topology: the system must scale by adding workers across many mailboxes, not by increasing per-mailbox concurrency, which has a hard external ceiling regardless of infrastructure scale.

Per §7's suggested tech stack, the report proposes Node/TypeScript or Python for the async I/O-heavy ingestion/webhook layer, Postgres for the state store, and per-mailbox work queues — explicitly flagged as "engineering judgment calls, not web-research-validatable facts... a starting proposal for the team to confirm" (Confidence: Lower/recommendation-only). This ADR treats that proposal as the accepted starting point given no research-backed alternative is presented, while making the resulting deployment topology concrete.

Per §1/§2, watch/subscription renewal has hard external deadlines (Gmail 7-day watch, Graph ~3-day subscription) — this is a liveness requirement the deployment topology must guarantee independent of deploys or rolling restarts.

## Decision

### Compute topology
- **Containerized services** (Docker images, OCI-compliant), orchestrated on Kubernetes, split into independently scalable deployments:
  - `ingestion-gateway`: receives Gmail Pub/Sub push and Graph webhook callbacks; stateless, horizontally autoscaled on request rate.
  - `mailbox-worker`: one logical worker-group per mailbox shard, pulls from that mailbox's queue (per [[adr-006-per-mailbox-worker-queue-backpressure]]), performs signal extraction + rule scoring; horizontally autoscaled on aggregate queue depth, not per-mailbox (a single mailbox cannot be sped up further per §3's external ceiling, but the total worker pool scales with mailbox count).
  - `llm-adjudicator`: stateless service wrapping the Claude API client, handles both sync and Batch API paths ([[adr-009-llm-adjudication-service]]); autoscaled on inbound adjudication request rate, with the Batch API path additionally run as scheduled Kubernetes CronJobs rather than long-lived pods, since batch submissions are async and poll for completion rather than holding a connection.
  - `reconciliation-sweeper`: scheduled job (CronJob) running `history.list`/delta-query reconciliation per §7 Stage 7; runs per-account on a rolling schedule, not a single fleet-wide sweep, to bound worst-case staleness per mailbox.
  - `renewal-scheduler`: a small, high-priority-class, dedicated deployment (not sharing a node pool with bursty workers) solely responsible for Gmail watch renewal (daily cadence, per §1 recommendation) and Graph subscription renewal (every 24h, per §2's ~3-day expiry) — isolated so that noisy-neighbor resource contention on shared workers cannot cause a renewal miss that silently breaks push ingestion for a mailbox.

### Message queue
- Per-mailbox work queues (as recommended in §7) implemented on a durable, ordered queue backend (e.g., a partitioned topic keyed by `mailbox_id`) so that all work for one mailbox is processed in order by one consumer at a time, naturally enforcing the per-mailbox concurrency ceiling from §3 without additional application-level locking.

### State store
- Postgres (per §7 recommendation) as the primary state store (message identity, classification state, VIP lists, corrections, per [[adr-007-message-identity-idempotency]] and [[adr-013-user-feedback-correction-loop]]), deployed as a managed, highly-available instance (primary + synchronous standby minimum) per region.
- Read replicas for the tenant admin API's read-heavy dashboard queries ([[adr-018-tenant-admin-configuration-api]]) to isolate reporting load from the classification hot path.

### Autoscaling model
- `mailbox-worker` and `llm-adjudicator` scale on custom metrics (`queue_depth`, `llm_adjudication_request_rate` respectively, per [[adr-015-observability-slos]]) via Kubernetes HPA with custom-metrics adapter, not just CPU/memory — CPU-based autoscaling would not reflect the true bottleneck (external API rate limits), so it is used only as a secondary/fallback signal.
- `ingestion-gateway` scales on request rate/CPU since it is a thin, low-latency webhook receiver with no external rate-limit exposure of its own.

### Multi-region posture
- **V1**: single-region deployment (one primary region), chosen for initial launch simplicity — explicitly a disclosed limitation, not a silent gap, consistent with [[adr-014-data-retention-gdpr-ccpa-compliance]]'s V1 residency caveat.
- **V2+**: region-pinned multi-tenant deployment — a tenant is provisioned into exactly one region at onboarding (its state store row, OAuth token vault entries, and mailbox workers all live in that region), rather than a single global active-active deployment. This is chosen over active-active because (a) message classification has no cross-tenant coupling requiring global consistency, and (b) it directly satisfies data-residency requirements ([[adr-014-data-retention-gdpr-ccpa-compliance]]) as a side effect of the topology rather than needing separate residency engineering.
- Each region runs the full stack independently; only a thin global control-plane (tenant routing table: which region owns which tenant) is shared, kept in a low-write, high-read datastore separate from per-region Postgres.

### Environments and rollout
- Standard three-tier environment promotion: dev → staging (with sandbox Gmail/Graph test accounts per [[adr-019-testing-strategy]]) → production.
- Production rollouts use rolling deploys with readiness probes gated on: (a) successful DB migration check, (b) `renewal-scheduler` health check confirming no imminent watch/subscription expiry within the deploy window, to avoid a rollout coinciding with a missed renewal.

## Consequences

### Positive
- Autoscaling on queue depth and adjudication rate (rather than CPU alone) reflects the true bottleneck identified in the research (external rate limits, §3), avoiding wasted infrastructure spend on scaling the wrong dimension.
- Isolating `renewal-scheduler` onto a dedicated, high-priority deployment directly protects against the single highest-consequence liveness failure mode identified in the research (missed watch/subscription renewal silently breaking push ingestion, §1/§2).
- Region-pinned V2+ topology satisfies data residency as a structural property rather than bolted-on logic.

### Negative
- Region-pinning defers full multi-region availability (failover across regions) to a later phase — a region outage in V1/early V2 is a full outage for tenants in that region, not a degraded-but-available state; this is addressed operationally via [[adr-017-disaster-recovery-backup-rpo-rto]] rather than via live failover.
- Per-mailbox ordered-queue partitioning requires careful partition-key design to avoid hot-partition skew if a small number of tenants have disproportionately high mail volume — needs monitoring (`queue_depth` per mailbox, [[adr-015-observability-slos]]) and a resharding plan as a follow-up, not solved by this ADR alone.
- Running Batch API polling as CronJobs rather than long-lived workers adds latency (poll interval) to the batch classification path — acceptable per §5's framing of Batch API as for "non-latency-sensitive work," but must not be used for the synchronous/time-sensitive adjudication path.

### Neutral
- The specific technology choices here (Kubernetes, Postgres, partitioned queue) are, per the research report's own caveat (§7), engineering judgment calls rather than research-validated facts — this ADR documents the team's accepted starting point, open to revision if operational experience contradicts it.

## Links
- Related: ADR-005 (ingestion sync strategy — drives ingestion-gateway and reconciliation-sweeper design), ADR-006 (per-mailbox worker/queue — core scaling unit), ADR-007 (message identity — state store schema), ADR-009 (LLM adjudication service — llm-adjudicator deployment), ADR-014 (compliance/residency — motivates region-pinning), ADR-015 (observability — autoscaling metrics source), ADR-017 (disaster recovery — region-outage handling), ADR-018 (tenant admin API — read-replica consumer), ADR-019 (testing strategy — environment promotion path)

## Verification

- Load test: simulate 10,000 mailboxes across a range of mail volumes, confirm autoscaling responds to `queue_depth`/adjudication-rate signals within the target scale-up latency (define target: 95th percentile scale-up trigger-to-ready under 3 minutes).
- Chaos test: kill `renewal-scheduler` pods during a rolling deploy and confirm readiness-probe gating prevents a deploy from proceeding if a watch/subscription is within its renewal danger window.
- Partition-skew test: seed one synthetic high-volume mailbox and confirm its queue partition does not starve other mailboxes' processing (bounded per-partition worker fairness).
- Region-provisioning test (V2+): provision a tenant into a specific region and confirm all its state-store rows, token-vault entries, and worker assignments remain within that region (data-residency structural test, cross-checked against [[adr-014-data-retention-gdpr-ccpa-compliance]]).
