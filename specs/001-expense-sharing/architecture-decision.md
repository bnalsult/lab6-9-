# SplitSmart Architecture Decision

**Decision:** Hybrid Java modular monolith with shared API and three client surfaces  
**Status:** Accepted for project documentation and implementation planning  
**Scope:** SplitSmart expense sharing, feature `001-expense-sharing`

## Decision

The backend will be implemented in Java 21 as a modular monolith, built with Maven and organized using Ports and Adapters. It will expose a versioned HTTPS REST API documented by `contracts/expense-api.yaml`. PostgreSQL stores transactional business records. Receipt binaries and notification delivery are accessed through replaceable adapters.

The product has three presentation surfaces: browser web, Android/iOS, and Java Swing desktop. All use the shared API. Clients own screens, navigation, local state, and non-authoritative validation only. The backend owns identity and access checks, expense state transitions, split calculations, approvals, financial events, payment validation, balance/report queries, and notification intent.

## Backend module boundaries

- `splitsmart-domain`: framework-free aggregates, value objects, domain services, events, and invariants.
- `splitsmart-application`: use cases, commands/queries, actor context, authorization policies, and inbound/outbound ports.
- `splitsmart-adapters`: PostgreSQL/migrations, receipt object storage, notification/outbox delivery, and clock implementations.
- `splitsmart-web`: REST controllers, API DTOs, authentication mapping, validation, error translation, and browser presentation.
- `splitsmart-boot`: executable composition root and dependency assembly.
- Client applications (`splitsmart-desktop`, web, mobile) consume the API contract; they do not depend on backend persistence modules.

Dependency direction is inward: domain has no framework dependency; application depends on domain; adapters implement inward ports; web invokes application use cases; boot assembles modules. API and UI models remain separate from domain entities and ORM mappings.

## Domain and financial rules

- `Expense` owns submitted expense data, participants, split details, and valid state transitions: `DRAFT -> SUBMITTED -> APPROVED | REJECTED`.
- A `Money` value type uses decimal arithmetic, explicit currency, and specified minor-unit rounding. Equal and percentage `SplitStrategy` implementations produce shares that sum exactly to the total.
- Each group has one designated approver in v1. The expense creator cannot approve their own expense. Approval decisions are retained with actor, timestamp, outcome, and notes.
- Only approved expenses create ledger source events. Approval record, ledger event, and outbox notification intent commit in one transaction. Idempotency and uniqueness constraints prevent duplicate posting.
- Payments append immutable payment/ledger events after group authorization and settlement validation. Balances are derived from approved expenses and payment events; any stored balance table is a rebuildable projection, never directly editable.
- Notifications are delivered asynchronously from a transactional outbox. A notification outage does not roll back a committed expense or payment.

## API and clients

The API is versioned under `/api/v1`. It exposes stable transport DTOs and error codes; it never exposes domain or ORM objects directly. Authorization is derived from the authenticated actor and resource membership, not trusted client-supplied user IDs. History and reports support group-scoped filters and pagination. Receipt access is authorized and time limited.

Swing, web, and mobile clients call the same API over HTTPS. Swing uses MVC/MVP-style presentation boundaries (views, controllers/presenters, view models, API client); no financial calculations, authorization decisions, or persistence access belong in GUI event handlers. API failures are translated to user-facing messages by each client.

## Requirements and traceability

| Requirement | Owner |
|---|---|
| FR-001, FR-013 accounts/privacy | Authentication adapter, `CurrentActor`, policies, scoped repositories |
| FR-002 groups | Group use cases and `ExpenseGroup` aggregate |
| FR-003, FR-004, FR-014 expense data/splits/validation | Submission use case, `Expense`, `Money`, split strategies |
| FR-005, FR-009 balances | Approval/payment use cases, immutable ledger events, rebuildable projection |
| FR-006 notifications | Domain events, outbox, delivery adapters |
| FR-007, FR-008 approval/audit | Expense state machine, approver policy, append-only approval record |
| FR-010 payments | `RecordPayment` use case and payment domain rules |
| FR-011 reports | Group-scoped query services/read models |
| FR-012, FR-015 web/mobile/desktop parity | Shared API contract and three presentation clients |

## Consequences

This design keeps financial transactions straightforward and prevents client rule drift. It adds a Java desktop client and three presentation implementations to maintain. Modules and ports require dependency checks and discipline; begin with the stated boundaries and avoid premature microservices. The old .NET/React stack in the initial Spec Kit plan is superseded by this decision; product requirements remain in `spec.md`.

## Companion artifacts

- Detailed design and workflow: `architecture-java.md`
- Requirements: `spec.md`
- Implementation plan: `plan.md`
- Data model: `data-model.md`
- API: `contracts/expense-api.yaml`
- Class diagram: `diagrams/class-diagram.puml`
- Activity diagram: `diagrams/activity-diagram.puml`
