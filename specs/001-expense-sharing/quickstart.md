# Quickstart: SplitSmart Validation Guide

## Prerequisites

- Java 21 JDK
- Maven (or use the Maven wrapper when added)
- PostgreSQL
- S3-compatible object storage for receipt uploads (for example, MinIO)
- Web, Android/iOS, or desktop tooling only for the client being run

## Local setup

1. Configure PostgreSQL and object storage using local environment configuration. Keep secrets out of source control.
2. From the repository root, build the Maven reactor: `mvn clean verify` (once implementation modules are present).
3. Start PostgreSQL and the local S3-compatible service, then apply the backend's versioned database migrations.
4. Start the backend composition module: `mvn -pl splitsmart-boot -am spring-boot:run` (assuming Spring Boot is selected).
5. Start the browser client and configure its API base URL to the local `/api/v1` endpoint.
6. Start the Android/iOS client in its platform development environment and configure the same API base URL.
7. Launch the Swing application from `splitsmart-desktop`; configure the same API base URL. The desktop client must not need database or object-store credentials.

Exact module commands and framework plugin names should be finalized when the implementation framework and client toolchains are selected. The API contract in `contracts/expense-api.yaml` is the common client integration reference.

## Validation scenarios

### Account, group access, and client parity

Register and authenticate users, create a group, and exercise the same group-scoped views from web, mobile, and Swing clients. Confirm each client sees only authorized account and group data and that server authorization governs every request.

### Expense submission and approval

Create an equal-split expense and submit it. Confirm it is `SUBMITTED` and no balance changes yet. Approve it as the designated approver and confirm the decision/audit record and exactly one ledger posting appear. Try creator self-approval, a second approval, and an unauthorized approver; each must be rejected without duplicate ledger effects.

### Split validation

Submit equal and percentage expenses, including a valid 50/30/20 split. Confirm deterministic currency rounding and that shares add to the total. Try missing participants and a percentage total other than 100%; submission must fail without ledger effects.

### Rejection and notifications

Reject a submitted expense with a note. Confirm it remains out of the ledger and the decision is auditable. Confirm relevant members receive in-app notification intent; simulate delivery failure and verify the outbox remains retryable without reversing the financial transaction.

### Payment, balances, and reports

Record an allowed settlement payment between group members. Confirm the payment and corresponding ledger event are persisted and the displayed balance is derived correctly. Generate date-filtered group and user reports and verify that results remain within the caller's authorization scope.

### Receipts

Upload an allowed receipt through the authorized upload flow. Verify metadata is persisted, storage remains private, and an unauthorized user cannot obtain a receipt URL.

## Definition of done

- Web, mobile, and Swing use the same versioned API and have no direct persistence access.
- Pending/rejected expenses never affect balances; approval is authorized, auditable, transactional, and idempotent.
- Equal/percentage splits and payments reconcile to immutable financial source records.
- Balance projections can be rebuilt from approved expenses and payments.
- Group scoping applies consistently to screens, API queries, receipts, and reports.
- The PlantUML sources in `diagrams/` match the implemented domain and workflows.
