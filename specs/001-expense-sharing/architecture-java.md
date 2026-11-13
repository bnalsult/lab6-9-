# SplitSmart Hybrid Java Architecture Design

**Status:** Accepted; consolidated with the shared-API baseline
**Date:** 2026-11-12  
**Scope:** Architecture for the expense-sharing feature in `specs/001-expense-sharing/`  
**Technology baseline:** Java 21, Maven

## 1. Purpose

This document is the detailed technical design for the accepted Hybrid architecture. It retains the baseline shared-API topology while replacing the .NET stack with a Java 21 modular monolith and adding a Java Swing desktop client alongside browser and Android/iOS clients. All clients use the same versioned JSON API. It emphasizes package boundaries, object-oriented domain rules, and strict separation between presentation and financial logic.

The architecture supports independent web, mobile, and Swing experiences while centralizing authorization, approvals, split calculations, balances, payments, and reporting in one backend. It does not change product scope or the single-approver v1 decision.

## 2. Architectural drivers and decisions

| Driver | Design response |
|---|---|
| Web, Android/iOS, and Swing with consistent behavior | One versioned REST API and one backend domain model; clients contain presentation and transport code only. |
| Expense approval before balance impact | Explicit expense state machine and transactional approval use case; ledger effects occur only on approval. |
| Reconstructable, auditable balances | Immutable financial events and approval records are the source of truth; balance views are derived projections. |
| User and group privacy | Authentication at the edge, authorization in application use cases, and membership-scoped repository queries. |
| Reliable notifications | Persist notification intent in the same transaction using an outbox, then deliver asynchronously. |
| Maintainable Java structure | Maven multi-module build with inward dependencies and domain-focused packages. |

### Chosen style

Use a **modular monolith** for the backend. The feature set needs strong transactional boundaries around approval and ledger postings; a single deployable service keeps those boundaries straightforward. Modules establish compile-time ownership and can be extracted later if operational evidence calls for it. Microservices are not required for the stated small-to-medium group scope.

Use **Ports and Adapters (Hexagonal Architecture)** within the backend. Domain and application code do not depend on HTTP, database, object-storage, or push-provider libraries. Adapters implement ports declared inward. A web MVC client and mobile applications call the same API; neither owns authoritative business rules.

## 3. System context

    +-------------------+   +------------------+   +------------------+
    | Browser web client|   | Android / iOS    |   | Java Swing       |
    | presentation      |   | clients          |   | desktop client   |
    +---------+---------+   +--------+---------+   +--------+---------+
              |                      |                      |
              +----------------------+----------------------+
                                     | HTTPS, versioned JSON API
                                     v
                     +-------------------------------+
                     | Java 21 SplitSmart backend    |
                     | REST/API -> Application       |
                     |              -> Domain        |
                     | Adapters implement ports     |
                     +-----------+---------+---------+
                                 |         |
                          PostgreSQL   S3-compatible
                          source data  private receipt store
                                 |
                          Outbox worker -> Push/email providers

The diagram shows logical dependencies, not required deployment units. The web UI and API may initially run in the same backend deployment. All clients use the API over TLS. Receipt access is authorized and time limited; clients never receive general object-store credentials.
## 4. Maven repository layout

```text
splitsmart/
├── pom.xml                         # Parent; Java 21 and dependency/plugin versions
├── splitsmart-domain/              # Pure domain model and domain services
├── splitsmart-application/         # Use cases, ports, commands, queries, DTOs
├── splitsmart-adapters/            # PostgreSQL, object storage, notification adapters
├── splitsmart-web/                 # REST API, auth, validation mapping, web UI
├── splitsmart-desktop/             # Swing UI and API client; no direct DB access
├── clients-mobile/                 # Android/iOS presentation and API client
├── splitsmart-boot/                 # Composition root and executable application
└── splitsmart-tests/               # Cross-module architecture and integration tests
```

Recommended dependency direction:

```text
splitsmart-domain
       ^
splitsmart-application
       ^
splitsmart-adapters     splitsmart-web
         ^                  ^
         +---- splitsmart-boot
```

`domain` has no framework dependencies. `application` depends on `domain` and defines inbound use cases and outbound ports. `adapters` depends on application/domain and implements outbound ports. `web` depends on application and translates HTTP/UI interactions into use-case calls. `boot` assembles implementations using dependency injection. The tests module depends on the modules under test. No dependency points from domain or application toward a framework adapter.

### Module contents

```text
splitsmart-domain/src/main/java/com/splitsmart/domain/
├── account/       # UserAccount, credentials value types (not password handling)
├── group/         # ExpenseGroup, Membership, GroupRole
├── expense/       # Expense, ExpenseParticipant, Split, ExpenseStatus
├── approval/      # ApprovalDecision, ApprovalRecord
├── ledger/        # LedgerEntry, BalanceSnapshot, posting rules
├── payment/       # Payment, settlement rules
├── notification/  # Notification event/value types
└── shared/        # Money, CurrencyCode, DomainEvent, domain exceptions

splitsmart-application/src/main/java/com/splitsmart/application/
├── port/in/       # Use-case interfaces
├── port/out/      # Repository, clock, receipt, notification, transaction ports
├── expense/       # SubmitExpense, ApproveExpense, RejectExpense
├── group/         # CreateGroup, ManageMembership
├── payment/       # RecordPayment
├── balance/       # GetBalances
├── reporting/     # GenerateReport
├── notification/  # ReadNotifications
└── security/      # CurrentActor and authorization policies

splitsmart-adapters/src/main/java/com/splitsmart/adapter/
├── persistence/jpa/       # ORM mappings and repository implementations
├── persistence/migration/  # Database migrations
├── receipt/s3/             # Object storage adapter
├── notification/           # In-app and push/email delivery
└── clock/                  # Production clock adapter

splitsmart-web/src/main/java/com/splitsmart/web/
├── api/v1/         # REST controllers, request/response models
├── ui/             # Web controllers, view models, templates/static assets
├── auth/           # Authentication filters and principal mapping
├── error/          # Stable API/UI error translation
└── validation/     # Transport-level input validation
```

Package names can be adjusted to the selected Spring Boot or other Java web stack. Framework annotations and ORM entities stay outside the domain module. The Android/iOS client is a separately built client project, not a Maven backend module; it consumes the same OpenAPI contract.

## 5. GUI and business-logic separation

### Web presentation

The web layer is responsible for routing, rendering, form/input validation, session or token handling, and mapping requests into application commands. Controllers are intentionally thin:

1. Resolve the authenticated actor and request data.
2. Construct a command or query.
3. Invoke an application use case.
4. Map the result or a known application error to a page/API response.

Templates and browser scripts format and display data. They must not calculate shares, decide approval eligibility, mutate balances, or infer group permissions. UI validation improves feedback but is repeated authoritatively in application/domain code.

### Mobile presentation

Android and iOS clients use screens, navigation, local form state, and API clients. They may offer immediate input feedback, but the server owns the canonical rules and returned state. No client connects directly to PostgreSQL or object storage. Shared OpenAPI schemas can generate or inform typed client models without sharing domain implementation code across platforms.

### Swing desktop presentation

The Swing client is an API client, not a desktop-hosted backend. Organize it around views, controllers/presenters, view models, and an API client. GUI event handlers collect input, perform immediate usability checks, invoke API operations, and render responses. They do not calculate canonical shares, approve expenses, set balances, authorize group access, or access persistence/object storage directly. Keep request/response DTOs client-owned or generated from the OpenAPI contract; do not import backend domain or ORM classes into the desktop module.

### Application layer

Application services coordinate a user goal, enforce actor and resource authorization, load aggregates through ports, invoke domain behavior, persist changes, and publish resulting events through ports. They contain orchestration, not presentation or transport concerns.

### Domain layer

Domain objects enforce invariants regardless of which client initiated an operation. The domain layer has no knowledge of HTTP, web pages, JSON, database schemas, cloud storage, or push vendors. Monetary values use `BigDecimal` behind a `Money` value type with explicit currency and rounding rules; never use binary floating point for amounts.

## 6. Object-oriented design and patterns

Apply patterns where they make a requirement explicit and testable:

- **Aggregate / Aggregate Root:** `Expense` owns participants, split details, status, and submission/approval transitions. Callers cannot change participant shares independently after submission. `ExpenseGroup` owns membership and designated approver policy. Aggregate methods guard invariants.
- **Repository:** Ports such as `ExpenseRepository`, `GroupRepository`, and `LedgerRepository` express persistence needs in domain/application terms. JPA implementations remain adapters.
- **Strategy:** `SplitStrategy` has `EqualSplitStrategy` and `PercentageSplitStrategy` implementations. It returns exact per-participant `Money` shares using a deterministic rounding/remainder rule; the sum must equal the expense amount. Select strategies from a validated split type, not arbitrary client class names.
- **State machine / State transitions:** `ExpenseStatus` and explicit methods constrain `DRAFT -> SUBMITTED -> APPROVED | REJECTED`. Approval/rejection requires an authorized designated approver, cannot self-approve, and cannot be repeated to post twice. Rejected or pending expenses have no ledger effect.
- **Domain events:** Approval emits `ExpenseApproved`; payment recording emits `PaymentRecorded`. Events express completed domain facts, not requests. They drive ledger posting, audit records, and notification intent after applicable invariants pass.
- **Transactional application service:** `ApproveExpense` coordinates approval, immutable approval record creation, ledger posting/projection update, and outbox insert in one database transaction. A uniqueness/idempotency constraint prevents duplicate posting on retries.
- **Outbox and asynchronous handler:** The transaction stores notification/outbox records atomically with the approved business action. A worker sends push/in-app notifications and marks delivery attempts. This avoids losing notifications when a process stops after commit.
- **Specification / policy object:** `ExpenseApprovalPolicy` checks designated approver and self-approval rules; `GroupAccessPolicy` checks membership and role. Policies are called by use cases and tested independently.
- **CQRS-lite:** Commands mutate state; query services provide dashboard, balance, and report read models. This is a code organization choice, not separate services or databases. Projections can be rebuilt from immutable financial events.
- **Adapter / Anti-corruption boundary:** Receipt storage and notification providers are behind ports, so vendor APIs and failure behavior do not leak into core rules.

Avoid a generic “manager” per entity, inheritance-heavy domain models, and a pattern for every class. Prefer composition and small explicit interfaces where substitution or isolation has a concrete benefit.

## 7. Core financial workflow

### Expense submission

1. The client sends group, amount, date, description, participants, split method and optional receipt reference.
2. Authentication identifies the actor. The use case verifies active group membership, participant eligibility, and upload authorization.
3. The selected split strategy validates inputs and calculates shares. Percentage shares must total exactly 100% at the defined precision; equal splits assign any minor-unit remainder deterministically.
4. The `Expense` aggregate is persisted as `SUBMITTED`, with immutable submitted values and audit metadata. No balance is posted yet.
5. An outbox event requests notifications to authorized relevant members.

### Approval

1. The approver invokes `ApproveExpense` with the expense ID and expected version/idempotency key.
2. The use case verifies group membership, designated approver role, and that the approver is not the expense creator.
3. In one transaction, the aggregate transitions to `APPROVED`, the approval decision is appended, and ledger entries are posted or a projection is updated from the approved expense. A unique source-event reference prevents double posting.
4. Notification intent is written to the outbox in the same transaction. A worker delivers it later.

### Payments and balances

Payment recording validates positive amount, distinct payer/receiver, group scope, and the current outstanding amount according to the selected overpayment policy. It appends a payment record and a corresponding immutable ledger event. Balances are computed as a projection of approved expenses and recorded payments; no endpoint directly sets a balance amount. The ledger projection can be reconciled/rebuilt from source records.

The source-of-truth decision is consistent with the data model: approved expense + payment + approval + relevant group membership explain the displayed balance. If a materialized balance table is used for response speed, it is a rebuildable projection, not an editable financial record.

## 8. Persistence and external services

- **PostgreSQL:** accounts, groups, memberships, expenses, participants, approval records, payment records, ledger events/projection, notification inbox, and outbox. Use transactions, foreign keys, unique constraints, and optimistic version checks for concurrent approvals/payments.
- **Receipt object storage:** store binary files outside PostgreSQL; keep object key, content type, size, checksum, and upload time in relational metadata. Enforce file type/size limits, private buckets, scoped access, and short-lived URLs. Check that the requester belongs to the group before issuing access.
- **Notifications:** persist in-app notifications; deliver push/email through replaceable adapters. Notification failure does not roll back an already committed expense or payment; retry from the outbox and expose delivery state operationally.
- **Authentication:** use a secure identity provider or a well-maintained Java security integration. Store only password hashes if local credentials are chosen; never put secrets or raw tokens in application logs. Authorization remains server-side and group-scoped.
- **Migrations:** use versioned schema migrations checked into source control. Schema changes must preserve the ability to audit and reconstruct posted balances.

## 9. API and client contract

Expose a versioned HTTPS REST API at `/api/v1` consumed by web, mobile, and Swing. Publish an OpenAPI contract. Use request/response DTOs that are separate from domain entities. Core resource groups include:

- `/accounts`, `/sessions` or identity-provider callbacks
- `/groups` and `/groups/{id}/members`
- `/groups/{id}/expenses`, `/expenses/{id}/submit`, `/expenses/{id}/approve`, `/expenses/{id}/reject`
- `/groups/{id}/balances`, `/groups/{id}/payments`
- `/groups/{id}/reports`
- `/notifications` and receipt upload authorization endpoints

Use stable error codes for validation, forbidden access, stale versions, invalid state transitions, and conflicts. Avoid returning information about groups or records the caller is not authorized to see. Use pagination and explicit date range filters for history and reports. API contract details should be reconciled with the existing `contracts/expense-api.yaml` before implementation.

## 10. Requirement traceability

| Requirement area | Architectural owner |
|---|---|
| FR-001, FR-013 account access/privacy | Web authentication adapter, `CurrentActor`, application authorization policies, scoped repositories |
| FR-002 group management | Group use cases and `ExpenseGroup` aggregate |
| FR-003, FR-004, FR-014 expense input and splits | Expense commands, value validation, `SplitStrategy` implementations |
| FR-005, FR-009 balances | Approval use case, immutable ledger events, rebuildable balance projection |
| FR-006 notifications | Domain events plus transactional outbox and notification adapters |
| FR-007, FR-008 approval/audit | Expense state machine, approval policy, append-only approval records |
| FR-010 payments | `RecordPayment` use case and payment/ledger domain rules |
| FR-011 reports | Group-scoped query services using source records/projections |
| FR-012, FR-015 web/mobile/Swing parity | Versioned shared API and separate platform-specific presentation clients |
| Constitution: immutability, security, verification | Aggregate invariants, database constraints, scoped authorization, automated domain/integration verification |

## 11. Quality attributes and operational requirements

- **Consistency:** Approval, approval record, ledger source event/projection, and outbox intent commit atomically. Use idempotency for retried commands.
- **Privacy:** Derive group scope from the authenticated actor and requested resource; do not trust client-supplied user IDs as authorization. Apply least privilege to receipts and reports.
- **Auditability:** Record actor, action, timestamps, decision, reason/notes, and source event IDs. Correct mistakes with compensating events or explicit reversals rather than editing posted history.
- **Performance:** Meet the spec plan’s goals: dashboard under 2 seconds, expense submission/balance refresh within 5 seconds, and standard 90-day reports within 2 seconds. Add indexes and projections only after profiling while retaining rebuildability.
- **Availability/failure handling:** Database transaction failure leaves no partial approval/ledger action. Notification and object storage outages have explicit retry/failure behavior; core financial operations must not depend on synchronous push delivery.
- **Observability:** Structured logs include correlation IDs and business event IDs, exclude sensitive financial/credential data, and expose transaction, outbox retry, and reconciliation metrics.
- **Portability:** Keep providers behind adapters; use environment-based configuration and secrets management. Package as a deployable Java service, with containers as a deployment option.

## 12. Verification approach

The constitution requires automated verification for core calculations and financial flows. The implementation should include:

- Domain unit tests for equal and percentage splits, exact currency rounding, zero/negative amounts, invalid percentage totals, and aggregate state transitions.
- Application tests for authorization, self-approval rejection, approval idempotency, payment limits, and no ledger effects for pending/rejected expenses.
- Persistence integration tests for transaction boundaries, unique posting references, optimistic concurrency, and projection rebuild/reconciliation.
- API-level tests for user/group isolation, receipt authorization, error mapping, and matching web/mobile contract behavior.
- End-to-end scenarios corresponding to the existing `quickstart.md`, including the 5-second balance visibility goal and report date filtering.

## 13. Trade-offs and risks

- A modular monolith simplifies atomic financial operations and deployment but requires module boundaries to be enforced through dependency checks and review.
- Web, native mobile, and Swing introduce multiple presentation implementations; the shared API and backend use-case ownership prevent business-rule divergence but do not remove client-specific UI work.
- A ledger projection improves read performance but introduces rebuild/reconciliation work. The underlying approved expenses and payment events remain authoritative.
- Outbox delivery is eventually consistent. The expense/approval commits immediately while notifications may arrive shortly afterward; the inbox and retry policy make this observable.
- Authentication provider, receipt retention policy, currency/rounding convention, group removal semantics, overpayment behavior, and approver timeout behavior still need explicit product/security decisions where existing requirements identify edge cases but do not settle policy.

## 14. Adoption sequence

1. Establish the Maven parent and module dependency rules on Java 21.
2. Implement the domain types and split strategies without framework dependencies.
3. Add application ports/use cases and persistence adapters with schema migrations.
4. Implement account/group authorization and expense submission/approval transaction flow.
5. Add payment posting, balance query/projection, reconciliation, and reporting.
6. Add outbox-based in-app/push notifications and receipt storage adapter.
7. Deliver the web, mobile, and Swing clients against the published API contract.
8. Verify the existing acceptance scenarios and performance targets before release.

## 15. Relationship to project documents

This design is the adopted technical architecture and is synchronized with `architecture-decision.md`, `plan.md`, `research.md`, `spec.md`, `data-model.md`, `quickstart.md`, and `contracts/expense-api.yaml`. `spec.md` remains authoritative for user-visible requirements; this document defines implementation boundaries. PlantUML sources in `diagrams/` show the agreed class and activity structures. Any future product behavior change must be recorded as a specification clarification and reflected in these artifacts.
