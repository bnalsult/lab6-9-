# Implementation Plan: SplitSmart Expense Sharing

**Branch**: `001-expense-sharing` | **Date**: 2026-11-12 | **Spec**: `specs/001-expense-sharing/spec.md`

## Architecture decision

Adopt the **Hybrid Java architecture** documented in `architecture-decision.md`: a Java 21 modular-monolith backend with a versioned REST API, PostgreSQL as the transactional store, and separate web, Android/iOS, and Java Swing clients. All clients call the same API. Financial rules and authorization are authoritative in the backend; no client connects directly to the database or owns balance/approval rules.

## Technical context

- **Backend language/runtime:** Java 21; Maven multi-module build.
- **Web/API:** Java web framework (Spring Boot is the default implementation candidate); REST API under `/api/v1`; OpenAPI contract in `contracts/expense-api.yaml`.
- **Persistence:** PostgreSQL with versioned migrations; immutable source records and rebuildable balance projections.
- **Receipts:** S3-compatible private object storage behind an application port.
- **Notifications:** durable in-app inbox and transactional outbox; push/email delivery through replaceable adapters.
- **Clients:** browser web app, Android/iOS app, and Java Swing desktop app. Each uses API DTOs and the same HTTPS API.
- **Verification:** Java domain/application/integration/API checks and client-specific UI checks, as required by the constitution.
- **Target platform:** Linux-hosted backend; browser and Android/iOS clients; Swing desktop client on supported Java desktop platforms.

## Constraints and quality goals

- One designated approver per group in v1; expense creator cannot approve their own expense.
- Pending and rejected expenses do not affect balances. Approval and ledger posting are atomic and idempotent.
- Balances are derived from approved expenses and payment events; users cannot directly edit them.
- Equal and percentage splits use decimal money and deterministic minor-unit rounding; participant shares sum exactly to the expense total.
- Group authorization is checked on the server for every resource; client-supplied IDs do not establish access.
- Dashboard target: under 2 seconds; expense submission and balance visibility: within 5 seconds; standard 90-day report: within 2 seconds for typical group sizes.
- Initial scope is small-to-medium groups and a single currency per group; no multi-currency conversion or external payment processing in v1.

## Backend module structure

```text
splitsmart/
├── pom.xml
├── splitsmart-domain/       # Framework-free entities, value objects, invariants
├── splitsmart-application/  # Use cases, inbound/outbound ports, commands/queries
├── splitsmart-adapters/     # PostgreSQL, object storage, notifications, clock
├── splitsmart-web/          # REST API, authentication, web presentation
├── splitsmart-boot/         # Dependency composition and executable service
├── splitsmart-desktop/      # Swing presentation and API client only
├── clients-mobile/          # Android/iOS presentation and API clients
└── splitsmart-tests/        # Cross-module architecture and integration tests
```

Dependency direction is inward: domain has no framework dependency; application depends on domain; adapters implement application ports; web/API calls application use cases; boot assembles the backend. Desktop, web, and mobile clients depend on the API contract only. Android/iOS is built with its platform client toolchain, outside the Maven backend reactor. See `architecture-java.md` and `diagrams/class-diagram.puml`.

## Core workflow decisions

1. Expense submission validates membership, participants, amount, date, split method, and receipt authorization. It persists the expense as `SUBMITTED`; it does not post a balance.
2. A designated approver approves or rejects. Approval writes the decision and one ledger source event in a transaction; rejection writes the decision without a ledger event.
3. A notification outbox record is committed with the business transaction. Delivery happens asynchronously and retries independently.
4. Payment recording appends a payment and corresponding ledger event after authorization and settlement validation.
5. Balance/report queries read authoritative records or rebuildable projections; no write endpoint sets a balance directly.

## Source documents and diagrams

- Product requirements and acceptance criteria: `spec.md`.
- Consolidated architecture decision: `architecture-decision.md`.
- Detailed technical architecture: `architecture-java.md`.
- Domain/persistence model: `data-model.md`.
- Client contract: `contracts/expense-api.yaml`.
- Local validation guide: `quickstart.md`.
- PlantUML source: `diagrams/class-diagram.puml` and `diagrams/activity-diagram.puml`.

## Constitution check

The design preserves user/group access boundaries, immutable financial history, approval before ledger impact, least-privilege receipt access, and verification of core financial flows. No exception to the SplitSmart Constitution is proposed.
