# ADR-017: Disaster Recovery, Backup & RPO/RTO Targets

- **Status**: proposed
- **Date**: 2026-09-16
- **Deciders**: Architecture Team
- **Tags**: disaster-recovery, backup, rpo, rto, resilience

## Context

Per §7 Stage 6, the state store is the single source of truth for message-identity mappings, classification/confidence/source history, VIP lists, sender reputation, corrections, and watch/subscription renewal schedules — its loss is not equivalent to losing a cache, since none of this state can be perfectly reconstructed from the providers alone (Gmail/Graph do not store "which layer classified this message" or "user's manual VIP overrides").

However, per §1/§2, both Gmail and Microsoft Graph remain the **authoritative source for the mail itself** — the system never owns primary copies of message content (per the content-minimization posture in [[adr-014-data-retention-gdpr-ccpa-compliance]], body content is not persisted at all). This materially changes the DR calculus versus a typical stateful system: recovering from a disaster does not require restoring lost email, only restoring the *classification state and configuration* built on top of it, with the provider APIs available as a resynchronization source once ingestion resumes (§7 Stage 7's "push notifies, pull reconciles" pattern is directly reusable as a post-recovery resync mechanism).

Per §1/§2's renewal deadlines (Gmail 7-day watch, Graph ~3-day subscription), any DR event long enough to miss a renewal window requires an explicit re-establishment procedure with the provider, not just a state-store restore — this is a provider-specific recovery step this ADR must define, distinct from generic backup/restore.

## Decision

### Backup strategy
- **Postgres state store**: continuous WAL archiving to object storage + daily full base backups, retained 35 days (point-in-time recovery window). Cross-region backup replication to a secondary region's object storage bucket for the region-pinned V2+ topology in [[adr-016-deployment-topology-infrastructure]], so a full regional loss does not also destroy the only backup copy.
- **OAuth token vault** ([[adr-004-oauth-token-storage-credential-vault]]): backed up as part of the same Postgres backup if co-located, or via the secrets manager's own native backup/versioning (GCP Secret Manager / Azure Key Vault both provide version history) if stored separately — recovery procedure must restore both consistently (a restored state-store row referencing a token version that no longer exists in the vault is an explicit failure mode to test against).
- **Tenant routing/control-plane table** ([[adr-016-deployment-topology-infrastructure]]): small, low-write — backed up at higher frequency (hourly) given its outsized blast radius if lost (misrouting all tenants).

### RPO/RTO targets

| Component | RPO | RTO | Rationale |
|---|---|---|---|
| Classification state / corrections / VIP lists | 5 minutes (WAL-based PITR) | 4 hours | Losing up to 5 min of corrections is tolerable (user can re-correct); full state-store rebuild from backup within a business-hours-scale window |
| OAuth token vault | 5 minutes | 1 hour | Prioritized faster RTO than general state — without valid tokens, ingestion cannot resume at all, compounding data loss via missed watch renewals |
| Tenant routing/control-plane | 15 minutes | 30 minutes | Smallest dataset, highest blast radius if stale/unavailable — fastest RTO target in the system |
| Provider-side mail content | N/A (not owned) | N/A | Gmail/Graph's own SLA governs; out of scope — recovery is "resume ingestion," not "restore mail" |

These targets are **initial proposals pending business sign-off** on acceptable downtime/data-loss windows — the research report does not supply industry-specific RPO/RTO benchmarks for this product category, so these are derived from general practice (5-minute PITR is a standard Postgres WAL-archiving capability, not a novel target) rather than a cited external source; flagged here as engineering judgment, consistent with the report's caution (§7) about unvalidated tech-stack-adjacent claims.

### Regional failover (V1 vs V2+)
- **V1** (single-region, per [[adr-016-deployment-topology-infrastructure]]): no live failover. A full regional outage is a full service outage; recovery is restore-from-cross-region-backup into a newly provisioned region, targeting the RTOs above measured from declared-disaster to service-restored, not from the outage's true start (detection time is tracked separately as an observability SLA, [[adr-015-observability-slos]]).
- **V2+** (region-pinned multi-tenant): a tenant's region failure still requires restore-into-new-region (no active-active failover, per [[adr-016-deployment-topology-infrastructure]]'s explicit deferral), but the blast radius is limited to tenants pinned to the failed region rather than the whole fleet — a structural improvement over V1 even without live failover.

### Post-recovery provider resynchronization procedure
This is the DR step specific to this system's dual dependence on provider push mechanisms, executed automatically by the `renewal-scheduler`/`reconciliation-sweeper` services on restart, not a manual runbook step:
1. On state-store restore, immediately treat **every** mailbox's watch/subscription as presumptively expired regardless of its last-known expiry timestamp (conservative default, since the outage duration may have exceeded the renewal window).
2. Re-establish Gmail `users.watch` and Graph webhook subscriptions for all mailboxes before resuming any other processing, respecting the per-mailbox rate limits from §1/§2 (batched re-registration, not a synchronous fleet-wide burst, to avoid self-inflicted throttling during recovery).
3. Run a full `history.list`/delta-query reconciliation sweep (§7 Stage 7 mechanism, reused verbatim) per mailbox to catch any mail that arrived during the outage window before push resumes — this is the same mechanism used for routine gap-filling, requiring no DR-specific code path.
4. Only after re-subscription and initial reconciliation completes for a mailbox is it marked "recovered" and returned to normal steady-state processing; partial fleet recovery (some mailboxes recovered, others still catching up) is expected and surfaced via the observability dashboard ([[adr-015-observability-slos]]), not treated as an all-or-nothing gate.

## Consequences

### Positive
- Because message content is never the system's primary copy (per [[adr-014-data-retention-gdpr-ccpa-compliance]]), DR scope and cost are meaningfully smaller than a typical mail-storage system — backup/restore only covers classification metadata and configuration, not multi-terabyte mail archives.
- Reusing the routine reconciliation-sweep mechanism (§7 Stage 7) for post-DR catch-up avoids building and maintaining a separate DR-specific resync code path, reducing the chance that the "rarely exercised" DR path silently rots.
- Treating all watches/subscriptions as expired on restore is a simple, conservative rule that avoids a subtle bug class (trusting a possibly-stale expiry timestamp after an unknown-duration outage).

### Negative
- No live cross-region failover in V1/V2 means any regional outage is a real, user-visible service interruption for the RTO window (up to 4 hours for the primary classification-state RPO/RTO tier) — must be disclosed in SLAs offered to enterprise customers.
- Batched re-subscription after a fleet-wide outage could itself take significant wall-clock time for a large tenant base under Gmail/Graph's per-account rate limits (§1/§2) — worst-case full-fleet recovery time scales with tenant count and must be modeled/load-tested, not assumed to fit within the stated RTOs at arbitrary scale.
- Token-vault/state-store consistency-on-restore (avoiding orphaned token-version references) requires coordinated backup/restore tooling across two systems (Postgres + secrets manager) — an integration point that must be explicitly tested, not assumed correct by virtue of each system's individual backup capability.

### Neutral
- RPO/RTO figures are initial targets pending business/compliance sign-off, not derived from a cited industry benchmark in the research report — expect revision once actual customer SLA commitments are negotiated.

## Links
- Related: ADR-004 (OAuth token vault — coordinated backup/restore), ADR-005 (ingestion sync strategy — reused reconciliation mechanism), ADR-014 (data retention/compliance — explains why body content is out of DR scope), ADR-015 (observability — outage detection and partial-recovery visibility), ADR-016 (deployment topology — region-pinning and blast-radius scoping)

## Verification

- Quarterly DR game-day: restore Postgres state store and token vault from backup into an isolated environment, run the post-recovery resync procedure against sandbox Gmail/Graph accounts ([[adr-019-testing-strategy]]), measure actual RTO against the 4-hour/1-hour/30-minute targets.
- Automated backup-integrity check: nightly job restores the latest base backup + WAL into a scratch instance and runs a schema/row-count sanity check, alerting if a backup is silently corrupt (per [[adr-015-observability-slos]] alerting).
- Cross-system consistency test: simulate a restore where the state store is rolled back further than the token vault's latest version; assert the recovery procedure detects and surfaces the mismatch rather than operating on an inconsistent token reference.
- Load test: measure batched re-subscription wall-clock time at representative fleet sizes (1k, 10k, 100k mailboxes) to validate the RTO targets scale as expected, feeding back into the targets themselves if they do not.
