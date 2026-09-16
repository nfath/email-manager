# Cross-Platform Email Triage System — Research & Planning

## Executive Summary

This document outlines a comprehensive design for a cross-platform email sorting and triage system capable of automatically classifying incoming emails into eleven categories: newsletters, job postings, social notifications, ecommerce/receipts, sales & deals, LinkedIn notifications, meeting cancellations, needs-reply, priority/VIP, phishing, and personal contacts. The system will integrate both Gmail (via Google Workspace API) and Outlook/Microsoft 365 (via Microsoft Graph) with a unified internal taxonomy and a layered classification pipeline combining deterministic rule-based signals, header/metadata analysis, and LLM-assisted adjudication for ambiguous cases. This approach balances cost, latency, privacy, and classification accuracy by processing messages through cheap deterministic layers first and only escalating to LLM classification when high-confidence rule-based signals are insufficient.

---

## 1. Gmail API — Capabilities Relevant to Sorting

### Labels, Filters, and Categories (Three Distinct Primitives)

**Labels** are Gmail's primary organizing primitive (combining the concepts of folders and tags):
- Created via `users.labels.create` (5 quota units)
- Hard limit: **500 user-created labels** per account; **10,000 total labels** including system labels
- Operations: list, update, delete via the [Gmail labels API reference](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.labels)

**Filters** (`users.settings.filters`) are server-side rules Gmail evaluates at message arrival:
- Actions limited to label add/remove (`addLabelIds`/`removeLabelIds`); no body regex or ML-based logic
- Useful as a **fallback/backstop layer** (guarantee VIP senders always get a label) but not the primary classifier
- Reference: [Manage Gmail filters](https://developers.google.com/workspace/gmail/api/guides/filter_settings)

**Categories** (Promotions, Social, Updates, Forums, Personal, Primary) are Google's native ML classifier:
- Exposed as **read-only system labels**: `CATEGORY_PERSONAL`, `CATEGORY_SOCIAL`, `CATEGORY_PROMOTIONS`, `CATEGORY_UPDATES`, `CATEGORY_FORUMS`
- Cannot create custom categories; **mutually exclusive per message**
- **Recommendation**: Read as a free, zero-cost prior signal (e.g., `CATEGORY_PROMOTIONS` strongly co-occurs with sales/newsletter categories), but do not rely as sole signal since Google's taxonomy doesn't map 1:1 to all 11 target categories (missing job-postings, phishing detection, needs-reply, VIP)
- Reference: [Gmail categories overview](https://support.google.com/mail/answer/3094499)

### Batch Operations

`users.messages.batchModify` applies label add/remove to up to **1,000 message IDs per call**, and is idempotent (re-adding a label is a no-op). This is the primary mechanism for applying classifier output at scale rather than individual `modify` calls.
- Reference: [batchModify API](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/batchModify)

### Push Notifications and Incremental Sync

**Push via Pub/Sub**: `users.watch` registers a mailbox against a Cloud Pub/Sub topic. Gmail publishes `{emailAddress, historyId}` (double base64-encoded) when the mailbox changes. 
- Watch expires after **7 days**; renewal required **at least every 7 days**, Google recommends **daily**
- Because push is not 100% guaranteed, pair with periodic `history.list` calls as a reconciliation fallback for gaps
- This is the standard "watch + history.list" pattern
- Reference: [Gmail push guide](https://developers.google.com/workspace/gmail/api/guides/push); validated by [Unipile 2026 guide](https://www.unipile.com/gmail-api-push-notifications/)

### People API for Contacts

Gmail itself has no contacts endpoint; use the separate **People API**:
- `otherContacts.list`/`otherContacts:search` (scope `contacts.other.readonly`) surfaces auto-detected contacts from mail/chat — **read-only, cannot write**
- `contacts.readonly` accesses the user's actual address book
- This pairing directly supports "personal contacts" category and VIP reciprocity-graph scoring
- Reference: [Contacts API migration](https://developers.google.com/people/contacts-api-migration), [otherContacts.list](https://developers.google.com/people/api/rest/v1/otherContacts/list)

### Quotas and Rate Limits

| Metric | Limit |
|--------|-------|
| Per-project quota | 1,200,000 quota units/min |
| Per-user quota | 15,000 units/min |
| Hard per-user ceiling | **250 quota units/second** |
| Max concurrent in-flight requests | 50 per mailbox |
| `messages.get` cost | 20 units |
| `drafts.create` cost | 10 units |
| `drafts.send` cost | 100 units |
| `labels.create` cost | 5 units |

**Practical implication**: Fetching full message bodies for LLM classification can burn quota quickly — a mailbox with 10k unread messages at 20u/msg = 200,000 units, which is within per-minute limits but requires deliberate batching/throttling.

References: [Gmail quota reference](https://developers.google.com/workspace/gmail/api/reference/quota); [Unipile 2026 limits](https://www.unipile.com/gmail-api-limits/); [Macro rate-limit postmortem](https://macro.com/posts/gmail-rate-limits)

### OAuth Scopes and Security Review

Minimum viable scopes for this system:
- `gmail.modify` (read + label/archive, no delete/send) — **restricted scope requiring Google security review**
- `gmail.labels` (create/manage labels)
- `contacts.readonly` + `contacts.other.readonly` (People API)

Avoid overly broad scopes like full `mail.google.com`. The `gmail.modify` scope requires Google's CASA security assessment for production OAuth — budget time for review in the roadmap.

Reference: [EmailEngine Gmail scopes reference](https://learn.emailengine.app/docs/accounts/gmail/gmail-api-scopes); [Google OAuth best practices](https://developers.google.com/identity/protocols/oauth2/resources/best-practices)

**Confidence: High** — All Gmail mechanics corroborated by ≥2 independent sources (official Google docs + third-party API vendors).

---

## 2. Microsoft Graph (Outlook / M365) — Capabilities Relevant to Sorting

### Categories, Folders, Rules, and Focused Inbox (Four Distinct Primitives)

**Categories** (`outlookCategory`):
- Per-user color-tagged label system, up to **25 distinct colors** with unlimited category names per color
- `displayName` is **immutable once created** (only `color` is updatable) — plan taxonomy carefully; renaming = delete+recreate
- Reading/writing *other users'* categories requires **application permission** `MailboxSettings.ReadWrite`; delegated permission cannot touch other mailboxes' categories
- Important for shared-mailbox/admin scenarios
- Reference: [outlookCategory resource](https://learn.microsoft.com/en-us/graph/api/resources/outlookcategory); [master categories API](https://learn.microsoft.com/en-us/graph/api/outlookuser-list-mastercategories)

**Folders**: Standard hierarchical mail folders; messages moved via Graph's move-message action (analogous to Gmail label removal, since Gmail lacks true folders).

**Inbox Rules** (`messageRule` under `mailFolders/inbox/messageRules`): Server-side conditional rules, similar role to Gmail filters — a backstop layer, not the primary classifier.

**Focused Inbox** (`inferenceClassification` + `inferenceClassificationOverride`): Microsoft's native ML "Focused vs Other" split, analogous to Gmail's Primary tab.
- Can create sender-level overrides (up to 1,000 per mailbox, keyed by SMTP address) to force a sender always into Focused or Other — lightweight way to implement part of "priority/VIP" without owning the classifier
- **Important caveat**: an override only affects future mail; does not retroactively reclassify existing messages
- Reference: [Focused Inbox overview](https://learn.microsoft.com/en-us/graph/api/resources/manage-focused-inbox); [create override](https://learn.microsoft.com/en-us/graph/api/inferenceclassification-post-overrides)

### Delta Queries and Webhooks (Push + Pull Pattern)

**Delta queries** (`GET /me/mailFolders/inbox/messages/delta`):
- Returns `@odata.deltaLink` for pull-based incremental sync — pure diff, no need to re-scan
- Complements push notifications for reliable sync

**Subscriptions (webhooks)**: 
- Push notifications on create/update/delete
- **Max 1,000 active subscriptions per mailbox** (across all apps)
- Mail-resource subscriptions **expire after ~4,230 minutes (~3 days)** — notably shorter-lived than Gmail's 7-day watch
- Renewal must be scheduled more aggressively (e.g., every 24h)

**Recommended pattern**: Use change notifications (push, "tell me when something changed") *plus* delta query (pull, "give me everything that changed since X") together — push tells you *when* to pull, delta tells you *what* changed cheaply.

References: [Delta query overview](https://learn.microsoft.com/en-us/graph/delta-query-overview); [Outlook change notifications](https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview)

### Batch Requests and Throttling

- **Batch requests**: max **20 sub-requests**, **4MB payload cap**
- Each sub-request individually subject to throttling — **a throttled (429) sub-request is nested inside a 200 batch envelope**, so naive code checking only the outer status will silently swallow failures (documented pitfall)
- **Concurrency**: by default only 4 individual requests from a batch dispatched to the Outlook backend concurrently, regardless of target mailbox
- General cap: **10,000 requests/10min/app**; 429 responses carry `Retry-After`

References: [Graph throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits); [MailboxConcurrency limit writeup](https://gsexdev.blogspot.com/2020/09/the-mailboxconcurrency-limit-and-using.html)

### Authentication and Scopes

**Identity Platform**: Microsoft Entra ID via MSAL; delegated (user sign-in) vs application (service-to-service) permission models.

Minimum scopes:
- `Mail.ReadWrite` (read+modify, no send)
- `MailboxSettings.ReadWrite` (categories, rules, Focused Inbox — **restricted, tenant-wide-by-default** application permission; admins can scope down via **application access policies** to specific mailboxes)
- `Contacts.Read`
- `offline_access` (required for refresh tokens — without it, the grant dies after first access-token expiry ~1hr)

Reference: [Microsoft Graph permissions](https://learn.microsoft.com/en-us/graph/permissions-reference); [Nylas Microsoft OAuth scopes](https://developer.nylas.com/docs/cookbook/use-cases/build/microsoft-oauth-scopes/)

**Confidence: High** for API mechanics (official MS Learn docs, cross-checked against Unipile/Nylas guides). **Medium** on exact application-access-policy scoping details — worth validating against current tenant admin docs at implementation time, as Microsoft evolves Entra controls frequently.

---

## 3. Cross-Platform Architecture: Reconciling Two Different Primitive Models

### The Abstraction Problem

**Gmail's model**: label-based, many-to-many tags on an immutable message
**Outlook's model**: folder+category hybrid, message lives in exactly one folder plus zero-or-more categories

### Unified Approach

1. **Single internal taxonomy**: Classify each message into 0-N of our 11 categories using a provider-agnostic taxonomy (e.g., `category: newsletter | job_posting | social | ecommerce_receipt | promo_deal | linkedin | meeting_cancelled | needs_reply | priority_vip | phishing | personal`).

2. **Action translator per provider**:
   - **Gmail**: map each internal category → a label (create once via `labels.create`, cache ID), apply via `batchModify`. Multiple categories per message = multiple labels (natively supported).
   - **Outlook**: map each internal category → an Outlook Category (color-tagged, multi-valued — also natively supports multiple per message). Reserve folder-moves only for archived-out-of-inbox cases (e.g., a dedicated newsletter folder) since folders are exclusive and fighting with Focused Inbox.

3. **Leverage platform primitives**: Read Gmail's `CATEGORY_*` and Outlook's `inferenceClassification` as low-cost *signals* into the classifier, not ignored. Reserve engine compute for categories neither platform's built-in ML covers (job postings, phishing beyond basic spam, needs-reply, VIP, meeting-cancellation, LinkedIn sub-typing).

4. **State and sync layer**: Both APIs are eventually-consistent and message IDs differ between IMAP/REST/native access. The system needs internal message identity (e.g., hash of Message-ID header + account) mapped to provider-specific IDs in a local datastore. This is also where "already classified, don't reclassify" state and user corrections/feedback live.

5. **Rate-limit posture**: Gmail's per-user 250 qu/s and Graph's ~4 req/s effective concurrency per mailbox are mailbox-scoped — horizontal scale-out (many users) is naturally parallel-safe. A single very active mailbox is the bottleneck, arguing for a per-mailbox worker/queue model with backoff on 429, not global concurrency limiting.

---

## 4. Classification Signals by Category

General principle: **layer signals cheapest→most expensive**; only escalate to LLM when header/rule heuristics are ambiguous. This mirrors SpamAssassin's and Rspamd's architecture.

| Category | Rule/Header Signals (Cheap, ~Free) | When to Escalate to LLM |
|---|---|---|
| **Newsletters** | `List-Unsubscribe` header (RFC 2369, 1998) ± `List-Unsubscribe-Post` (RFC 8058, 2017); `List-Id` header; known ESP domains (`*.mcsv.net`, `list-manage.com`, Substack `*.substack.com`, ConvertKit) | Ambiguous when List-Unsubscribe exists but content looks personal (e.g., some CRM-sent 1:1 mail includes it) |
| **Sales & Deals / Promotions** | List-Unsubscribe/ESP-domain signals + promotional lexicon ("% off", "sale ends", price patterns); Gmail's `CATEGORY_PROMOTIONS` as free prior | Distinguishing "newsletter" vs "promo" needs body/subject semantics |
| **LinkedIn Notifications** | Sender domain `*.linkedin.com` / `jobs-noreply@linkedin.com` — deterministic and cheap. LinkedIn has 70+ distinct sub-types (connection requests, InMail, digests, job alerts, endorsements) configurable by recipient's LinkedIn settings | Sub-type classification (job alert vs connection request vs digest) needs subject-line/body pattern matching if subject lines aren't stable |
| **Job Postings** | Known ATS sending domains (Greenhouse, Lever, Ashby, Workday) + LinkedIn/Indeed job-alert patterns; **no canonical current list of exact notification-sending domains** — needs empirical collection | Subject/body heuristics ("new jobs matching your search", "you were invited to apply") |
| **Ecommerce / Receipts** | **schema.org `Order`/`ParcelDelivery` JSON-LD markup** (many retailers embed for Gmail action buttons) — high-confidence, zero-effort signal; subject regex ("Your order #", "shipped", "receipt") for retailers without markup | Retailers without schema.org or clear subject patterns |
| **Meeting Cancellations** | `Content-Type: text/calendar; method=CANCEL` MIME part OR body containing `BEGIN:VCALENDAR` + `METHOD:CANCEL` (iTIP/RFC 5546, iMIP/RFC 6047) — deterministic, parse `.ics` directly, no ML needed | Essentially none — this is solved at header/MIME parse level |
| **Needs Reply** | Exclude no-reply-pattern senders (`^no.?reply@`) and `Auto-Submitted: auto-generated/auto-replied` (RFC 3834) and `Precedence: bulk`; presence of `?` / interrogative or imperative sentence structure (academic research: ~80% of "needs response" sentences interrogative, 11% imperative); thread position (last message in thread addressed to user, no reply sent) | **Single most LLM-dependent category** — pure heuristics have known precision problems (rhetorical questions, CC'd threads). Best pattern: rule-based candidate generation (exclude bulk/auto senders, find unresponded threads) then LLM final judgment on filtered set |
| **Priority / VIP** | Manual VIP list (user-curated) as deterministic override; **reciprocity/frequency graph** (sent-to-them count × replied-to-them count, normalized); calendar co-attendance (shared meetings); Outlook's `inferenceClassificationOverride` (force sender→Focused) | Less relevant — fundamentally a graph/frequency computation over user's own mail history, not content classification |
| **Phishing / Security** | SPF/DKIM/DMARC pass-fail (`Authentication-Results` header) is *necessary but insufficient* — does not address lookalike/cousin domains that legitimately pass all three checks; lookalike-domain detection (edit-distance vs user's contact-domain list + common brand domains); urgency-language lexicons; native platform flags (Gmail spam/phishing warnings, Microsoft Defender/EOP verdicts via Graph if available) | High-value LLM use for residual: business email compromise / spear-phishing passing auth and not matching known lookalike — treat LLM output as *additional signal feeding a rule-based final gate*, not sole determinant |
| **Personal Contacts** | Absence of all "bulk mail" signals simultaneously: no `List-*` headers, no `Precedence: bulk`, no `Auto-Submitted`; sender in address book (People API) or prior 1:1 reciprocal exchange; not matching any category sender-domain lists | LLM as tie-breaker only for ambiguous CC-heavy or forwarded threads |
| **Social Notifications** | Sender-domain allowlist for major platforms (Facebook, Instagram, X/Twitter, Reddit); Gmail's `CATEGORY_SOCIAL` as free prior | Rare edge cases only |

**Confidence grading**:
- **High** for header/RFC mechanics (List-Unsubscribe, Precedence, Auto-Submitted, iCalendar METHOD, schema.org Order) — standards-track with multiple corroborating sources
- **Medium** for competitor sending-domain lists (Greenhouse/Lever/ATS addresses) — no canonical current list found; needs direct empirical sampling during implementation
- **Medium-low** for "needs reply" and "priority/VIP" algorithms — closer to academic/patent prior art (general shape) than a spec to copy; expect to tune thresholds against real usage data

---

## 5. LLM-Based Classification Design

### Cost and Latency Data Points

A fine-tuned smaller model can achieve **~94% accuracy at ~$0.008/email and ~0.8s latency** for email classification, versus a large general model at ~$0.035/email and ~20s latency (single-source benchmark — treat as directional, not universal truth).

**Current pricing (Sep 2026)** — Claude models preferred for this task:
- **Haiku 4.5**: $1/$5 per MTok in/out — **recommended default tier** (classification/routing is its explicit use case)
- Sonnet 5: $2/$10 per MTok in/out
- Opus 5: $5/$25 per MTok in/out

**Cost optimization strategies**:
1. **Batch API**: Provides flat **50% discount** on both input and output, *stacks with prompt caching*
2. **Prompt caching**: Cache reads cost **10% of base input price** on non-Fable models
3. **Practical combo**: Prompt-cache the fixed system prompt/taxonomy + few-shot examples, batch multiple emails' variable content into one request, use Haiku 4.5, prefer async Batch API for non-latency-sensitive work (initial sweep) and synchronous calls only for time-sensitive cases (real-time nudges)

### Batching Strategy

Literature on LLM batching shows:
- Prompt batching (multiple items, one prompt, one structured response) gives **~1.2-1.9x median per-item speedup** vs one-call-per-item
- Forcing structured/short outputs (single-token enum or JSON, not prose) is the **single biggest lever on output-token cost**

**Recommendation**: Design classification prompt to return compact JSON array of `{message_id, categories: [...], confidence}` for N emails in one call, not N individual calls.

### Privacy Consideration: Content Minimization

Several categories (phishing, needs-reply, VIP) genuinely require body/header content. However, SaneBox/Clean Email explicitly **process only headers/metadata, never body** as a stated privacy-by-design feature.

**Recommended approach**:
- Default to **metadata-only + header-based rule classification** for majority of categories (8 of 11 categories resolvable without body content per §4 table)
- Only send full body content to LLM for minority of messages that: pass initial cheap filter AND land in ambiguous bucket (needs-reply, residual phishing, ambiguous newsletter/promo split)
- This both **controls cost** and **substantially reduces privacy surface area / DPA scope** versus "send every email body to an LLM"

**Confidence: Medium** — Pricing/latency numbers are current as of Sep 2026, general batching principles well-established, but specific %-accuracy benchmark is single-source; should not be treated as a guarantee for this exact taxonomy.

---

## 6. Prior Art and Competitive Landscape

### SaneBox
- **Approach**: Headers/metadata-only (stated privacy design choice), learns from user drag-to-folder behavior, not content NLP
- **Taxonomy**: Fixed categories (SaneLater/low-priority, SaneNews/bulk, SaneBlackHole/permanent-block, SaneCC, SaneReminders/no-reply-nudges)
- **Reusable insights**: 
  - "Drag one email to retrain the sender forever" UX pattern is powerful
  - "Reminders for messages that got no reply" (our "needs-reply-adjacent" feature from the sent side) directly applicable
- Source: [SaneBox release notes](https://www.sanebox.com/release-notes)

### Clean Email
- **Approach**: Also explicitly headers/metadata-only
- **Taxonomy**: ~33 predefined "Smart Folders" grouped by sender/domain/recency rather than content classification
- **Insight**: Large fraction of useful categorization achievable without body content — validates metadata-first design
- Source: [Clean Email Smart Folders](https://clean.email/help/basics/smart-folders)

### Shortwave (Alfred)
- **Approach**: Closest prior art to this exact ask — AI "Smart Bundles" auto-sort newsletters/receipts/notifications, user-configurable per-bundle rules, AI priority scoring from sender history + thread activity + **content** (unlike SaneBox/Clean Email)
- **Limitation**: Gmail-primary with only nascent other-provider support; **no one has yet shipped the cross-platform (Gmail+Outlook) version well** — this project's differentiation opportunity
- Source: [Alfred.ai blog on email triage tools](https://get-alfred.ai/blog/best-ai-email-triage-tools)

### SpamAssassin
- **Architecture**: Canonical rule-scoring system — hundreds of independent rules, each with signed weight; sum crosses threshold → verdict
- **Directly transferable**: Additive-scoring pattern (rather than monolithic classifier) directly applicable to our multi-label problem — each category gets its own weighted-signal accumulator with independent threshold
- Source: [SpamAssassin documentation](https://spamassassin.apache.org/)

### Rspamd (Modern Successor)
- **Architecture**: 4-stage pipeline: pre-filters → parallel main filters → post-filters → action decision; Bayesian classifier as one signal among 60+ modules; **neural network as refinement layer on top of rule-symbol outputs, NOT a replacement**
- **Directly applicable**: Pre-filter→rules→ML-refinement layering is arguably **the best architectural template for this project** — cheap rules run first and produce named "symbols," ML/LLM layer only refines/adjudicates ambiguous cases rather than reprocessing everything from raw text
- Source: [Rspamd architecture](https://docs.rspamd.com/developers/architecture/)

### Unified Email API Vendors (Nylas, Unipile, EmailEngine)
- **Purpose**: Single API surface over Gmail+Outlook+IMAP
- **Trade-off analysis**: 
  - **Pro**: Handle OAuth, webhook normalization, provider quirks; reduce integration complexity
  - **Con**: Per-account fee, data-flow dependency on third party seeing all mail content, vendor lock-in
- **Recommendation for this project**: **Direct Gmail API + Graph API integration** (no intermediary) likely preferable because:
  - Phishing-detection requirement already requires full header/body access anyway
  - Privacy goals favor not funneling all mail through additional third party
  - Trade more integration code for not introducing another data processor
- Source: [StackReferee API infrastructure comparison](https://stackreferee.com/email-api-infrastructure/nylas-vs-unipile-vs-aurinko-vs-nango-vs-emailengine)

**Confidence: High** for SaneBox/Clean Email claims (stated directly as features) and SpamAssassin/Rspamd architecture (official docs). **Medium** for Shortwave's exact internal methodology (marketing-page inference, not confirmed engineering docs).

---

## 7. Recommended Architecture

### Pipeline Stages (Rspamd-Inspired Layered Model)

**Stage 1: Ingestion**
- Per-account watcher: Gmail `users.watch`+Pub/Sub with daily watch-renewal cron + `history.list` reconciliation; Graph webhooks (renew every 24h before ~3-day expiry) + delta-query reconciliation
- Normalize both providers into internal `RawMessage` (headers, MIME structure, body, provider message ID)
- This is the one place provider differences get absorbed

**Stage 2: Signal Extraction (Cheap, Deterministic, Every Message)**
- Header parsing: List-Unsubscribe/List-Id, Precedence, Auto-Submitted, Authentication-Results/SPF-DKIM-DMARC verdict, calendar METHOD, schema.org JSON-LD block
- Sender-domain lookup: maintained allowlists (ESPs, ATS platforms, social platforms)
- People/Contacts API cross-reference
- Native-category read-through: Gmail `CATEGORY_*`, Outlook `inferenceClassification`

**Stage 3: Rule Engine / Scoring Layer**
- SpamAssassin/Rspamd-style additive weighted rules per category
- Each signal contributes to per-category score
- Categories crossing high-confidence threshold finalized here without LLM
- **Expected outcome**: Large majority of messages resolved (newsletters, ecommerce receipts, meeting cancellations, social, LinkedIn, most promo/deals)

**Stage 4: LLM Adjudication Layer (Minority of Traffic)**
- Only messages ambiguous after Stage 3 (near-threshold scores, conflicting signals, inherently judgment-based categories)
- Sends to Claude Haiku 4.5 via Batch API for backlog processing or light synchronous call for time-sensitive cases
- Uses cached system prompt (taxonomy + few-shot examples) and structured JSON multi-item output per call

**Stage 5: Action/Apply Layer**
- Internal category set → provider-specific label/category mapping
- Idempotent — safe to reapply
- Gmail: `batchModify` (≤1,000 IDs/call, respecting 250qu/s)
- Graph: Batched category-assignment/move requests (≤20 sub-requests/batch, watching for nested-429s)

**Stage 6: State Store**
- Internal message-identity table: Message-ID hash ↔ provider IDs
- Per-category confidence + which layer decided it (rule vs LLM) for auditability/retraining
- User-correction feedback loop (when user manually re-labels, feed back to adjust rule weights/few-shot examples)
- VIP list; learned sender reputation (frequency/reciprocity scores)
- Watch/subscription renewal schedule

**Stage 7: Scheduling**
- Push-driven (webhook/Pub/Sub) as primary trigger for near-real-time triage
- Periodic reconciliation sweep (`history.list`/delta query) as safety net for missed pushes
- This "push notifies, pull reconciles" pattern is explicitly Microsoft-recommended and implicitly Google's own design

### Suggested Tech Stack

| Component | Recommendation | Rationale |
|-----------|---|---|
| **Backend language** | Node/TypeScript or Python | Strong async I/O for ingestion/webhook layer |
| **Message queue** | Per-mailbox work queues | Respects per-mailbox rate limits without global contention |
| **State store** | Postgres | Message-identity/state/VIP/feedback tables, ACID transactions |
| **Classification service** | Claude API (Haiku 4.5 default) | Explicit use case for Haiku; Batch API for bulk, sync for time-sensitive |
| **Secrets management** | Cloud provider's (GCP Secret Manager / Azure Key Vault) | OAuth client secrets, per-user tokens encrypted at rest |
| **Token handling** | Encrypted at rest in DB, never in browser | Short-lived access tokens, `offline_access`/refresh-token rotation |

### Phased Roadmap

**MVP**
- Gmail-only (single provider first; simpler push model)
- Rule-engine-only for 6 fully-deterministic categories: newsletters, ecommerce receipts, meeting cancellations, social, LinkedIn, basic phishing (SPF/DKIM/DMARC + lookalike detection)
- Manual VIP list (no reciprocity graph yet)
- Apply via labels only

**V2**
- Add Outlook/Graph integration (same internal taxonomy, proves abstraction layer)
- Add LLM adjudication for needs-reply and ambiguous promo/newsletter split
- Add reciprocity-graph VIP scoring

**V3**
- Full feature set: calendar co-attendance VIP signal, user-feedback-driven rule/few-shot retraining, job-postings category refined from empirical sender-domain collection, cross-account unified view/reporting

**Confidence: High** on layered rules→LLM architecture (supported by Rspamd precedent + cost data justifying LLM minimization) and push+reconcile sync pattern (both vendors document explicitly). **Lower/recommendation-only** on specific tech-stack choices — these are engineering judgment calls, not web-research-validatable facts; treat as a starting proposal for the team to confirm.

---

## 8. Security and Privacy Considerations

### Least Privilege

**Google side**:
- `gmail.modify` + `gmail.labels` + `contacts.readonly`/`contacts.other.readonly`
- Avoid full-mailbox scope `mail.google.com`

**Microsoft side**:
- `Mail.ReadWrite` + `MailboxSettings.ReadWrite` + `Contacts.Read` + `offline_access`
- Use **application access policies** to scope `MailboxSettings.ReadWrite` down from all-tenant-mailboxes to only onboarded mailboxes
- Avoid tenant-wide unscoped application permissions

### Token Handling Best Practices

- **Encrypt refresh tokens at rest**: Use secrets manager or app-level encryption over a DB column; never in client-side/browser storage
- **Rotate on refresh**: Treat as short-lived credentials
- **Access tokens**: Short-lived (default ~1hr), regenerated via refresh token
- Reference: [OAuth token storage best practices](https://developers.google.com/identity/protocols/oauth2/resources/best-practices)

### Content Minimization Strategy

- Prefer **metadata/header-only processing** wherever a category can be resolved (§4/§5 analysis shows 8 of 11 categories resolvable without body)
- When body content must reach LLM (residual phishing, needs-reply): treat as passing through a **data processor** and paper with a DPA under GDPR
- Reduce both cost and the privacy/DPA surface area

### Phishing Detection's Structural Tension

This category requires the **deepest content/header access** of any category:
- Full headers for auth-results (SPF/DKIM/DMARC)
- Full body for lookalike/urgency-language detection

**Recommendation**: Explicitly flag to user/stakeholders that phishing detection drives the broadest scope requirement. Double-check whether provider's native verdict (Gmail's built-in warnings, Microsoft Defender/EOP if licensed) can be read via API as a substitute/supplement rather than rebuilding from scratch.

**Confidence: Medium-high** — GDPR/CCPA mechanics are well-established (high confidence). The specific "is this design DPA-sufficient" determination is a legal judgment call — **get real legal review before launch**, not just engineering best-effort.

---

## 9. Known Research Gaps and Open Questions

1. **ATS Notification Sending Domains**: No canonical current list of Greenhouse/Lever/Ashby/Workday **notification-email sending domains** — needs empirical sample collection during implementation, not documentation lookup.

2. **Shortwave Internal Methodology**: Shortwave's internal classification approach is inferred from marketing copy, not verified engineering documentation.

3. **Fine-Tuned Classifier Benchmark**: The single benchmark cited ("94% accuracy at $0.008/email") is one arXiv preprint source — good directional signal, not a guarantee transferable to this exact taxonomy.

4. **Microsoft Application Access Policy Current State**: Microsoft's exact current (Sep 2026) admin-console mechanics for scoping `MailboxSettings.ReadWrite` via application access policy should be re-verified against live tenant docs at implementation time — Entra admin surfaces change frequently.

5. **Rule Weight Tuning**: The SpamAssassin/Rspamd-inspired rule-scoring model requires empirical tuning of signal weights and per-category confidence thresholds against real mailbox data — no public reference dataset of labeled emails for this exact 11-category taxonomy exists.

6. **LLM Adjudication Fallback Behavior**: When the LLM adjudication layer fails or returns low-confidence scores, what is the fallback logic? (Default to rule-engine output? Require manual review? Defer to user's existing folder structure?) — should be decided during V2 implementation.

---

## Appendix: References and Sources

- [Gmail API Reference](https://developers.google.com/workspace/gmail/api/reference)
- [Microsoft Graph API Reference](https://learn.microsoft.com/en-us/graph/api/)
- [Google OAuth Best Practices](https://developers.google.com/identity/protocols/oauth2/resources/best-practices)
- [Unipile Email Integration Guide](https://www.unipile.com/)
- [Nylas Documentation](https://developer.nylas.com/)
- [Rspamd Architecture](https://docs.rspamd.com/developers/architecture/)
- [SpamAssassin Documentation](https://spamassassin.apache.org/)
- [RFC 2369: The Use of URLs in Internet Mail](https://tools.ietf.org/html/rfc2369)
- [RFC 5546: iCalendar Transport-Independent Interoperability Protocol (iTIP)](https://tools.ietf.org/html/rfc5546)
- [RFC 6047: iCalendar Message-Based Interoperability Protocol (iMIP)](https://tools.ietf.org/html/rfc6047)
- [Schema.org Order Type](https://schema.org/Order)
