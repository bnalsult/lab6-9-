# Implementation Plan: SplitSmart Expense Sharing

**Branch**: `001-expense-sharing` | **Date**: 2026-11-12 | **Spec**: `specs/001-expense-sharing/spec.md`

**Input**: Feature specification from `/specs/001-expense-sharing/spec.md`

## Summary

SplitSmart will provide a secure group-based expense management platform with account access, group membership, expense submission, single-approver validation, automatic balance recalculation, payments, notifications, and reporting. The implementation will use a shared backend API and separate web/mobile clients so the same financial rules apply to browser and mobile users.


## Technical Context

**Language/Version**: C# / .NET 8, TypeScript 5.x, React 18, React Native / Expo

**Primary Dependencies**: ASP.NET Core Web API, EF Core, PostgreSQL, SignalR, React + Vite, React Native, JWT auth, MinIO/S3-compatible object storage, xUnit, Playwright

**Storage**: PostgreSQL for transactional data; object storage (S3/MinIO) for receipt images; optional Redis for cache/notification fan-out

**Testing**: xUnit + FluentAssertions for backend business logic; Playwright for web flows; React Native Testing Library for mobile screens; API integration tests for approval/payment/reporting scenarios

**Target Platform**: Linux-hosted backend with web browser access and Android/iOS mobile app clients

**Project Type**: web-service + mobile-app

**Performance Goals**: dashboard loads in under 2 seconds for typical group sizes; expense submission and balance refresh within 5 seconds; report generation for 90-day periods within 2 seconds for standard group data volumes

**Constraints**: secure role-based access for group members and approvers; immutable financial records; no direct balance edits outside approved workflow; support for equal and percentage splits; notifications must be delivered to group members who are relevant to the expense

**Scale/Scope**: supports small to medium group sizes typical of households, trips, and teams; not a global multi-currency ledger in v1

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The design remains aligned with the SplitSmart Constitution:

- User Ownership and Access: each account has isolated balances, transactions, and notifications; access is scoped by group membership and role.
- Shared Accountability for Groups: all expenses and payments are anchored to explicit group memberships and participant lists.
- Financial Integrity and Auditability: every balance change is generated from validated expense/payment flows and stored with approval metadata.
- Security, Privacy, and Consent: receipt uploads and personal financial data use least-privilege access and secure storage.
- Verification Before Release: core business rules will be covered by automated tests for equal split, percentage split, settlement reconciliation, and approval enforcement.

No constitution violations identified; architecture decisions do not require exceptions.

## Project Structure

### Documentation (this feature)

```text
specs/001-expense-sharing/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Validation guide
├── contracts/           # API contract definitions
└── tasks.md             # Future implementation task breakdown
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── API/
│   ├── Application/
│   ├── Domain/
│   ├── Infrastructure/
│   └── Identity/
└── tests/
    ├── Integration/
    └── Unit/

web/
├── src/
│   ├── app/
│   ├── components/
│   ├── features/
│   └── services/
├── public/
└── tests/

mobile/
├── src/
│   ├── app/
│   ├── features/
│   ├── navigation/
│   └── services/
└── tests/
```

**Structure Decision**: split the solution into a shared backend API plus two client surfaces (web and mobile). This keeps the financial and approval logic centralized while allowing platform-specific UX for browsers, Android, and iOS without drifting rules across clients.

## Complexity Tracking

No constitution violations requiring exception tracking.
