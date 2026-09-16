# Domain Model — Email Triage System

Context map for the cross-platform (Gmail + Microsoft Graph/Outlook) email
triage system's 10 bounded contexts. Each context doc is grounded in
[`.plans/email-sorting-system-research.md`](../../.plans/email-sorting-system-research.md)
and follows the shared per-context source layout:

```
src/<context-name>/
  domain/
    entities/         # Entities and aggregate root
    value-objects/    # Value objects
    events/           # Domain events
    services/         # Domain services
    repositories/      # Repository interfaces
  application/        # Use cases / application services
  infrastructure/      # Repository implementations, ACL adapters
  index.ts             # Public API of the context
```

## Bounded contexts

| # | Context | Core aggregate(s) | Doc |
|---|---|---|---|
| 01 | Identity & Provider Access | `MailboxConnection`, `CredentialVault` | [01-identity-provider-access.md](01-identity-provider-access.md) |
| 02 | Mail Ingestion | `RawMessage`, `WatchSubscription`, `ReconciliationSweep` | [02-mail-ingestion.md](02-mail-ingestion.md) |
| 03 | Classification Engine | `MessageClassification`, `ClassificationRun` | [03-classification-engine.md](03-classification-engine.md) |
| 04 | LLM Adjudication | `AdjudicationBatch` | [04-llm-adjudication.md](04-llm-adjudication.md) |
| 05 | Relationship Graph | `ContactRelationship` | [05-relationship-graph.md](05-relationship-graph.md) |
| 06 | Action & Provider Sync | `MessageSyncState`, `ProviderCategoryMapping` | [06-action-provider-sync.md](06-action-provider-sync.md) |
| 07 | Feedback & Learning | `Correction`, `RuleWeightProfile`, `FewShotExampleSet` | [07-feedback-learning.md](07-feedback-learning.md) |
| 08 | Tenant & Account Management | `Tenant`, `MailboxConnection` (ref), `UsageCounter` | [08-tenant-account-management.md](08-tenant-account-management.md) |
| 09 | Audit & Compliance | `AuditEntry`, `DataSubjectRequest`, `RetentionPolicy` | [09-audit-compliance.md](09-audit-compliance.md) |
| 10 | Notification & Reporting | `DigestSchedule`, `AlertRule`/`Alert`, `MetricSnapshot` | [10-notification-reporting.md](10-notification-reporting.md) |

## Context relationships (high level)

```
Tenant & Account Mgmt (08) ──Partnership──> Identity & Provider Access (01)
Identity & Provider Access (01) ──ACL──> Gmail OAuth / Microsoft Entra
Mail Ingestion (02) ──ACL──> Gmail Messages API / Graph Mail API
Mail Ingestion (02) ──Customer-Supplier──> Classification Engine (03)
Classification Engine (03) ──Customer-Supplier──> LLM Adjudication (04)
Classification Engine (03) ──Open Host Service──> Relationship Graph (05)
Classification Engine (03) + LLM Adjudication (04) ──Customer-Supplier──> Action & Provider Sync (06)
Action & Provider Sync (06) ──ACL──> Gmail batchModify / Graph category & folder APIs
Feedback & Learning (07) ──Published Language (events)──> Classification Engine (03), LLM Adjudication (04)
Audit & Compliance (09) ──Open Host Service (DataLifecycleParticipant port)──> all contexts holding message/PII data
Notification & Reporting (10) ──Conformist (consumes events)──> all contexts
```

Each context doc's own "Relationships to other bounded contexts" section is the
authoritative source for the exact relationship type per pair — this diagram is
a navigation aid, not a substitute.

## Cross-reference to ADRs

| Context | Primary ADRs |
|---|---|
| 01 Identity & Provider Access | ADR-004, ADR-020 |
| 02 Mail Ingestion | ADR-005, ADR-006, ADR-007 |
| 03 Classification Engine | ADR-008, ADR-010, ADR-012, ADR-021 |
| 04 LLM Adjudication | ADR-009, ADR-022 |
| 05 Relationship Graph | ADR-011 |
| 06 Action & Provider Sync | ADR-002, ADR-007, ADR-010 |
| 07 Feedback & Learning | ADR-013 |
| 08 Tenant & Account Management | ADR-003, ADR-018 |
| 09 Audit & Compliance | ADR-014, ADR-017 |
| 10 Notification & Reporting | ADR-015 |

## Open questions surfaced by the domain model

- **LLM Adjudication fallback**: `04-llm-adjudication.md` defines an explicit
  rule-only/manual-review fallback for LLM failure or budget exhaustion,
  resolving a gap the research report left open (see ADR-013, ADR-022).
- **Phishing verdicts are never applied directly from the LLM**: `03-classification-engine.md`
  and ADR-012 both enforce that LLM phishing signals are converted to a
  weighted symbol re-summed against rule scores, never a sole determinant.
- Domain modeling otherwise inherits the same research-gap caveats listed in
  [`docs/adr/README.md`](../adr/README.md#known-open-risks-carried-over-from-the-research-report).
