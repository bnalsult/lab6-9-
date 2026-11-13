# Data Model: SplitSmart Expense Sharing

## Core Entities

### User Account

Represents a person who can authenticate, join groups, and track personal balances and notifications.

Fields:
- id: UUID
- email: unique string
- passwordHash: string
- firstName: string
- lastName: string
- createdAt: datetime
- updatedAt: datetime
- status: active | suspended

Relationships:
- one-to-many with GroupMembership
- one-to-many with ExpenseParticipant
- one-to-many with PaymentRecord
- one-to-many with Notification
- one-to-many with ApprovalRecord

Validation rules:
- email must be unique and valid
- passwordHash must be present
- status must be known and auditable

### Group

Represents a shared financial context such as roommates, a trip, or a team.

Fields:
- id: UUID
- name: string
- description: optional string
- createdByUserId: UUID
- createdAt: datetime
- updatedAt: datetime
- status: active | archived

Relationships:
- one-to-many with GroupMembership
- one-to-many with Expense
- one-to-many with PaymentRecord

Validation rules:
- name must not be empty
- at least one member is required for an active group

### GroupMembership

Links users with groups and defines access roles.

Fields:
- id: UUID
- groupId: UUID
- userId: UUID
- role: owner | admin | member
- joinedAt: datetime
- leftAt: nullable datetime

Relationships:
- many-to-one with Group
- many-to-one with UserAccount

Validation rules:
- user cannot be duplicated in the same group
- role must be valid and permission-bound

### Expense

Represents a shared cost event that may be approved before impacting balances.

Fields:
- id: UUID
- groupId: UUID
- createdByUserId: UUID
- amount: decimal(18,2)
- currency: string
- expenseDate: date
- description: string
- splitMethod: equal | percentage
- receiptUrl: nullable string
- status: draft | submitted | approved | rejected | archived
- submittedAt: nullable datetime
- approvedAt: nullable datetime
- rejectedAt: nullable datetime
- createdAt: datetime
- updatedAt: datetime

Relationships:
- many-to-one with Group
- many-to-one with UserAccount as creator
- one-to-many with ExpenseParticipant
- one-to-one with ApprovalRecord

Validation rules:
- amount must be greater than zero
- expenseDate must be valid
- description must not be empty
- splitMethod must be supported
- percentage totals must equal 100% when splitMethod is percentage
- receiptUrl must be present if a file was uploaded

State transitions:
- draft -> submitted -> approved or rejected
- approved -> archived after settlement close if required by policy

### ExpenseParticipant

Represents a user’s share of a specific expense.

Fields:
- id: UUID
- expenseId: UUID
- userId: UUID
- sharePercent: nullable decimal(5,2)
- shareAmount: decimal(18,2)
- isPayer: boolean
- createdAt: datetime

Relationships:
- many-to-one with Expense
- many-to-one with UserAccount

Validation rules:
- shareAmount must equal computed share for the method in use
- when splitMethod is equal, sharePercent may be null or derived later
- if the participant list is empty, the expense must be rejected

### ApprovalRecord

Captures the review decision for an expense before it affects balances.

Fields:
- id: UUID
- expenseId: UUID
- approverUserId: UUID
- decision: approved | rejected
- notes: nullable string
- createdAt: datetime

Relationships:
- one-to-one with Expense
- many-to-one with UserAccount as approver

Validation rules:
- approver cannot approve their own expense in a single-approver workflow
- decision must be recorded before balance updates are applied

### BalanceLedger

Represents the net amount owed between users within a group context.

Fields:
- id: UUID
- groupId: UUID
- fromUserId: UUID
- toUserId: UUID
- balanceAmount: decimal(18,2)
- updatedAt: datetime

Relationships:
- many-to-one with Group
- many-to-one with UserAccount as debtor and creditor

Validation rules:
- balanceAmount may be positive or negative depending on direction
- ledger entries are derived from expense and payment events; no direct manual balance mutations are allowed

### PaymentRecord

Records money that was paid to settle a balance.

Fields:
- id: UUID
- groupId: UUID
- payerUserId: UUID
- receiverUserId: UUID
- amount: decimal(18,2)
- currency: string
- paymentDate: date
- notes: optional string
- createdAt: datetime

Relationships:
- many-to-one with Group
- many-to-one with UserAccount as payer
- many-to-one with UserAccount as receiver

Validation rules:
- amount must be greater than zero
- payer and receiver cannot be the same user
- payment is not allowed to exceed the open balance without a settlement exception policy

### Notification

Represents a user-facing event about relevant group activity.

Fields:
- id: UUID
- userId: UUID
- groupId: UUID
- type: expense_created | expense_approved | payment_recorded | reminder
- message: string
- isRead: boolean
- createdAt: datetime

Relationships:
- many-to-one with UserAccount
- many-to-one with Group

Validation rules:
- type must be recognized
- message must not be empty

### Receipt

Stores optional image metadata associated with an expense.

Fields:
- id: UUID
- expenseId: UUID
- storageKey: string
- contentType: string
- fileSizeBytes: integer
- uploadedAt: datetime

Relationships:
- many-to-one with Expense

Validation rules:
- storageKey must be present
- contentType must be supported

## Derived Data and Rules

The system should generate reports and balance summaries from persisted events rather than by storing a secondary mutable copy that can drift from the source records. The key invariant is:

approved expense + payment record + approval record + group membership = current balance state

This preserves auditability and ensures any user-visible balance can be reconstructed from raw financial records.
