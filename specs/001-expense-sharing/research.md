# Research and Architecture Decisions: SplitSmart

## Consolidated decision

Adopt the **Hybrid Java architecture** detailed in `architecture-decision.md`. The backend is a Java 21 modular monolith exposing a shared, versioned REST API. Browser, Android/iOS, and Java Swing clients use that API. PostgreSQL is the transactional source of truth; receipt binaries use private object storage. The backend owns authorization, expense approval, split calculations, ledger events, payments, balances, reports, and notification intent.

This consolidates the original Spec Kit product architecture (shared backend API plus distinct browser/mobile clients) with the alternative Java architecture (inward dependencies, domain-focused modules, and Ports and Adapters). It replaces the prior .NET/React technology selection; it does not change the product’s group expense scope or single-approver v1 rule.

## Decisions and rationale

### Backend shape and client topology

Use a modular monolith so approval, audit recording, ledger posting, and notification intent can share a clear transactional boundary. A microservice split is not justified by the current small-to-medium group scope. Establish modules and dependency rules now; extract services only if operational evidence later warrants it.

The REST API is the only client-facing business boundary. Web, mobile, and Swing clients may provide local input feedback, but they do not connect to PostgreSQL/object storage or make authoritative financial decisions. This preserves consistent behavior across platforms.

### Domain and persistence model

Balances are derived from approved expenses and payment events. Source financial records and approval decisions are append-only; a materialized balance view is allowed only as a rebuildable projection. Expense approval, approval audit record, ledger source event, and outbox intent commit atomically. Idempotency and a unique source-event reference prevent duplicate postings.

### Expense approval

Use one designated approver per group in v1. A submitter cannot approve their own expense. Pending and rejected expenses have no ledger effect; rejection retains its actor, decision, timestamp, and notes.

### Splits and money

Support equal and percentage splits. Use decimal money with an explicit currency and rounding rule. Equal split remainder assignment is deterministic; percentage splits must total exactly 100% at the defined precision; participant shares must total the expense amount.

### Receipts and notifications

Store receipt binaries in private S3-compatible object storage and metadata in PostgreSQL. Expose scoped, short-lived access rather than general storage credentials. Persist notification intent in a transactional outbox and deliver asynchronously with retries; delivery failure does not undo a committed financial operation.

## Alternatives considered

- **Original .NET/React baseline:** sound shared-API and multi-client topology, but conflicts with the selected Java/Swing direction and has less explicit domain-level Java structure. Its product requirements and client separation are retained; its stack is superseded.
- **Java architecture without Swing:** provides good Java backend boundaries and web/mobile API clients, but does not define the requested desktop client. Swing is added as another API client.
- **Direct database access from desktop/mobile clients:** rejected because it bypasses authorization and makes client behavior authoritative.
- **Microservices:** rejected for v1 because they complicate atomic approval/ledger operations without a demonstrated independent-scaling need.
- **Editable balance fields:** rejected because they break auditability and reconciliation.
- **Multi-level or absent approval:** rejected; clarified product decision is a single approver.

## Open product/operational decisions

Before implementation, specify policies where the requirements leave choices open: group member removal and historical access, overpayment/partial settlement behavior, approver timeout/reassignment, receipt retention and size/type limits, supported currency and rounding precision, identity provider, notification channel priorities, and report export formats. These do not change the architecture decision.
