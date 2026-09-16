# Architecture Decision Records — Email Triage System

Index of all ADRs for the cross-platform (Gmail + Microsoft Graph/Outlook) email
triage system. Each ADR is grounded in the research findings in
[`.plans/email-sorting-system-research.md`](../../.plans/email-sorting-system-research.md).

All ADRs below are currently **Status: proposed** pending architecture review sign-off.

| # | Title | Status | Date | Tags |
|---|---|---|---|---|
| [ADR-001](ADR-001-layered-classification-pipeline.md) | Layered Classification Pipeline Architecture (Rules → LLM Adjudication → Action) | proposed | 2026-09-16 | architecture, classification, pipeline, cost-control |
| [ADR-002](ADR-002-provider-abstraction-anti-corruption-layer.md) | Provider Abstraction & Anti-Corruption Layer for Gmail vs Microsoft Graph | proposed | 2026-09-16 | architecture, integration, gmail, microsoft-graph, anti-corruption-layer |
| [ADR-003](ADR-003-multi-tenant-data-isolation.md) | Multi-Tenant Data Isolation Strategy | proposed | 2026-09-16 | multi-tenancy, security, data-isolation, postgres |
| [ADR-004](ADR-004-oauth-token-vault.md) | OAuth Token Storage & Credential Vault Design | proposed | 2026-09-16 | security, oauth, secrets-management, encryption |
| [ADR-005](ADR-005-ingestion-sync-strategy.md) | Ingestion Sync Strategy — Push + Reconciliation Sweep | proposed | 2026-09-16 | ingestion, sync, gmail, microsoft-graph, reliability |
| [ADR-006](ADR-006-worker-queue-rate-limit-backpressure.md) | Per-Mailbox Worker/Queue Model & Rate-Limit Backpressure Strategy | proposed | 2026-09-16 | scalability, rate-limiting, queueing, backpressure |
| [ADR-007](ADR-007-message-identity-idempotency.md) | Internal Message Identity & Idempotency Key Design | proposed | 2026-09-16 | data-model, idempotency, deduplication |
| [ADR-008](ADR-008-deterministic-rule-scoring-engine.md) | Deterministic Rule/Signal Scoring Engine Design | proposed | 2026-09-16 | classification, rules-engine, spamassassin, rspamd |
| [ADR-009](ADR-009-llm-adjudication-service.md) | LLM Adjudication Service Design | proposed | 2026-09-16 | llm, cost-optimization, claude, batch-api, prompt-caching |
| [ADR-010](ADR-010-category-taxonomy-provider-action-mapping.md) | Category Taxonomy & Provider Action-Mapping Table | proposed | 2026-09-16 | taxonomy, data-model, gmail, microsoft-graph |
| [ADR-011](ADR-011-vip-relationship-graph-scoring.md) | VIP / Relationship-Graph Scoring Design | proposed | 2026-09-16 | vip-scoring, relationship-graph, classification, gmail, microsoft-graph |
| [ADR-012](ADR-012-phishing-detection-subsystem.md) | Phishing Detection Subsystem Design | proposed | 2026-09-16 | security, phishing, classification, rule-engine, llm |
| [ADR-013](ADR-013-user-feedback-correction-loop.md) | User Feedback / Correction Loop & Retraining Strategy | proposed | 2026-09-16 | feedback, retraining, rule-engine, llm, learning |
| [ADR-014](ADR-014-data-retention-gdpr-ccpa-compliance.md) | Data Retention, Content Minimization & GDPR/CCPA Compliance Posture | proposed | 2026-09-16 | compliance, privacy, gdpr, ccpa, retention, data-minimization |
| [ADR-015](ADR-015-observability-slos.md) | Observability — Logging, Metrics, Tracing, Alerting | proposed | 2026-09-16 | observability, metrics, tracing, alerting, slo |
| [ADR-016](ADR-016-deployment-topology-infrastructure.md) | Deployment Topology & Infrastructure | proposed | 2026-09-16 | infrastructure, deployment, autoscaling, multi-region, containerization |
| [ADR-017](ADR-017-disaster-recovery-backup-rpo-rto.md) | Disaster Recovery, Backup & RPO/RTO Targets | proposed | 2026-09-16 | disaster-recovery, backup, rpo, rto, resilience |
| [ADR-018](ADR-018-tenant-admin-configuration-api.md) | Tenant / Admin Configuration API Design | proposed | 2026-09-16 | api, configuration, multi-tenant, admin |
| [ADR-019](ADR-019-testing-strategy.md) | Testing Strategy — Contract, Golden-Set, and Regression Gating | proposed | 2026-09-16 | testing, quality, contract-tests, evaluation, ci |
| [ADR-020](ADR-020-secrets-management-least-privilege.md) | Secrets Management & Least-Privilege Scope Governance | proposed | 2026-09-16 | security, secrets, oauth, least-privilege, scope-governance |
| [ADR-021](ADR-021-schema-taxonomy-versioning.md) | Schema / Taxonomy Versioning & Migration Strategy | proposed | 2026-09-16 | taxonomy, schema-migration, versioning, backward-compatibility |
| [ADR-022](ADR-022-llm-cost-governance-circuit-breaker.md) | LLM Cost Governance & Spend Budget Circuit Breaker | proposed | 2026-09-16 | cost-governance, llm, budget, circuit-breaker, finops |

## Reading order for implementation planning

1. **Foundational architecture**: ADR-001, ADR-002, ADR-007, ADR-010
2. **Provider integration & sync**: ADR-005, ADR-006
3. **Multi-tenancy & security**: ADR-003, ADR-004, ADR-020
4. **Classification pipeline**: ADR-008, ADR-009, ADR-011, ADR-012
5. **Learning loop**: ADR-013, ADR-021
6. **Compliance & governance**: ADR-014, ADR-022
7. **Operability**: ADR-015, ADR-016, ADR-017, ADR-019
8. **Product surface**: ADR-018

## Known open risks carried over from the research report

These are explicitly flagged as unresolved in the underlying research
(`.plans/email-sorting-system-research.md`, §9 "Known research gaps") and are
called out in the relevant ADRs rather than treated as settled:

- No canonical list of ATS (Greenhouse/Lever/Ashby/Workday) notification sending domains — ADR-008, ADR-012.
- Needs-reply and VIP scoring formulas are directional prior art, not validated algorithms — ADR-011.
- The cited LLM classification cost/accuracy benchmark is single-source — ADR-009.
- Microsoft's exact current admin-console mechanics for scoping `MailboxSettings.ReadWrite` via application access policy need re-verification at implementation time — ADR-020.
- Whether the phishing-detection design is "DPA-sufficient" is a legal determination, not an engineering one — ADR-014.
