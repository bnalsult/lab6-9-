# SplitSmart Constitution

<!-- Sync Impact Report
- Version change: 0.0.0 -> 1.0.0
- Modified principles: none (initial constitution)
- Added sections: User Ownership and Access, Shared Accountability for Groups,
  Financial Integrity and Auditability, Security, Privacy, and Consent,
  Verification Before Release, Security, Privacy, and Data Integrity,
  Development Workflow and Quality Gates, Governance
- Removed sections: none
- Follow-up TODOs: none
-->

## Core Principles

### I. User Ownership and Access
Every user account is the sole owner of their personal profile, balances,
payments, and notifications. SplitSmart MUST authenticate each user, scope
all data to that account, and prevent cross-user visibility without explicit
group membership and authorization. This protects personal financial data and
preserves trust in shared expense tracking.

### II. Shared Accountability for Groups
Every expense, group, and balance adjustment MUST be rooted in an explicit
group or participant list. Users can only create or view expenses within
groups they belong to, and group members must be able to see the same shared
ledger state. This keeps settlement calculations fair, transparent, and
reviewable.

### III. Financial Integrity and Auditability
Every balance change MUST come from a validated expense, payment, or approved
adjustment. Amounts, dates, split logic, and approvers must be retained as
immutable records. Manual edits to balances without a corresponding
action are prohibited. This prevents silent errors and ensures that reports
reflect the actual settlement history.

### IV. Security, Privacy, and Consent
Receipt images, account data, and payment history MUST be protected with
least-privilege access, encryption in transit and at rest, and clear consent
boundaries for data sharing. Group members must not be able to see more than
the minimum required to manage shared costs. This reduces privacy risk in a
financial system.

### V. Verification Before Release
All core calculations, approval flows, notifications, and payment
reconciliation logic MUST be covered by automated tests and verified before
release. New or changed expense-splitting logic requires regression tests for
equal splits, percentage splits, and balance reconciliation. This prevents
incorrect settlements and preserves trust in financial reporting.

## Security, Privacy, and Data Integrity
SplitSmart MUST enforce role-based access control for group admins, members,
and approvers. Receipt uploads, sensitive financial metadata, and cross-device
session tokens MUST be protected with encryption and secure storage. Payment
and balance records MUST be immutable once posted, with a tamper-evident
audit trail and a clear record of who approved each financial action.

## Development Workflow and Quality Gates
All features MUST be implemented through a reviewable workflow: requirements
are captured, changes are tested, and approval is required before release. New
expense logic MUST include tests for edge cases such as zero-value expenses,
partial settlements, percentage totals, currency rounding, and multi-user
reconciliation. Mobile and web experiences MUST remain functionally equivalent
for core flows, including group access, expense entry, notifications, and
reporting.

## Governance
This Constitution supersedes informal practices for SplitSmart development and
operations. Any change to the project rules, working standards, or release
expectations MUST be documented as a versioned amendment with a rationale,
affected principles, and migration or compliance notes. Reviewers MUST confirm
that code, tests, and operational practices remain aligned with these
principles before a release is approved.

Compliance is mandatory: every feature, architecture decision, and release
candidate MUST be checked against the Principles above; exceptions require
explicit written approval and a traceable remediation plan. The project MUST
maintain clear records of decisions, approvals, and financial-risk controls so
the system remains accountable to its users.

**Version**: 1.0.0 | **Ratified**: 2026-11-12 | **Last Amended**: 2026-11-12
