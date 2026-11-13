# Research: SplitSmart Expense Sharing

## Research Questions

### 1) Which architecture best matches the product and the constitutional constraints?

Decision: Use a shared backend API with separate web and mobile clients, backed by PostgreSQL and object storage for receipts.

Rationale:
- The spec requires secure account isolation, role-based group membership, approval gates, and balance reconciliation.
- Centralizing the business logic in one API prevents drift between web and mobile experiences.
- PostgreSQL provides reliable transactional integrity and reporting queries for balances and payment histories.
- Receipt images are large and less likely to belong in the primary relational schema; object storage keeps the system efficient and scalable.

Alternatives considered:
- Single monolithic app with direct DB access in the UI: rejected because it weakens security boundaries and makes cross-platform consistency harder to enforce.
- Serverless functions only: rejected because the domain logic and approval workflow are too stateful and audit-sensitive for a fragmented architecture.

### 2) What approval model should be implemented in v1?

Decision: Use a single designated approver per group, with a review decision captured in immutable records.

Rationale:
- The clarified requirement explicitly chooses a single approver model.
- This is simpler to govern, easier to test, and consistent with the constitutional requirement for auditability.

Alternatives considered:
- Multi-level approval: rejected because it adds complexity and delay beyond the v1 requirement.
- No approval gate: rejected because it violates the finance integrity principle and introduces disputes.

### 3) How should the system calculate and reconcile balances?

Decision: Treat balances as derived values from approved expenses, payment records, and user participation, not as directly editable ledger fields.

Rationale:
- Balance changes are a consequence of validated events and should be reproducible from underlying data.
- This preserves transparency and prevents hidden or manual edits that would break auditing.

Alternatives considered:
- Direct user-editable balance fields: rejected because they violate the constitution and make settlement history untrustworthy.
- Recalculation only at report time: rejected because it would delay user-visible updates and complicate payment reconciliation.

### 4) What notification strategy should be used?

Decision: Use in-app feeds plus push notifications for relevant group activity, emitted from backend event handlers.

Rationale:
- The requirement requires notification when new expenses or updates occur in a group.
- Event-driven notifications keep group members informed without making the client poll aggressively.

Alternatives considered:
- Email-only delivery: rejected because it is not immediate enough and does not match the mobile-first workflow.
- Client-side polling only: rejected because it is less efficient and less reliable for timely updates.

### 5) How should receipt images be handled?

Decision: Store receipt files in object storage and record metadata in the expense record.

Rationale:
- Receipt uploads are large and may involve binary data.
- Keeping them out of the core transactional database helps maintain performance and simplifies retention rules.

Alternatives considered:
- Store images directly in the database: rejected because it increases database load and complicates backup/retention.
- Ignore receipts: rejected because the requirements explicitly call for optional receipt images.

## Final Architecture Decision

SplitSmart will be implemented as a secure backend service with a Postgres-owned truth model, a React-based web client, and a React Native mobile client. Financial flows will be validated by approval rules before they affect balances, and all reporting will be generated from the same underlying ledger and payment records.
