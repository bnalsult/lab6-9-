# Feature Specification: SplitSmart Expense Sharing

**Feature Branch**: `001-expense-sharing`

**Created**: 2026-11-12

**Status**: Architecture consolidated; implementation pending

**Input**: User description: "Customer asked you to develop a software “SplitSmart” for sharing expenses between individuals or groups of people, such as friends, roommates, family members or coworkers. It should come with the following functions/features.

• User accounts: Each person using the SplitSmart should have their own user account, which they can use to log in and access their own expense and balance information.

• Group management: The software should allow users to create and manage groups of people, such as roommates, trip or party participants to more easily track shared expenses and balances within those groups.

• Expense creation: The software should allow users to create new expenses
    o The new expenses should contain information such as amount, date, description, shared manner (equally split or split by %) and an optional receipt image.

    o It should also allow users to specify which other users were involved in the expense, so that the appropriate balance adjustments can be made.

• Notification system: The software should have a way to notify users when other group members create a new expense.

• Approval workflow: The software should have a system for reviewing and approving expenses before they are added to the system and used to adjust balances. This could involve multiple levels of approval or a single review process, depending on the needs of the group.

• Balance tracking: The software should track the balance owed by each user to every other user, taking into account all expenses that have been entered.

• Payment tracking: The software should allow users to record when payments have been made to settle balances owed, so that the balances are accurately reflected in the system.

• Reporting: The software should provide users with reports and summaries of their expenses, balances, and payments, possibly including the ability to generate custom reports with specified date ranges and other parameters.

• OS supported: primary web-based and also comes with app that can be run on Android and IOS."

## Clarifications

### Session 2026-11-12

- Q: Which approval model should the v1 SplitSmart expense workflow use for expenses before they affect balances? → A: Single approver

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create an account and access personal balances (Priority: P1)

A user signs up for SplitSmart, logs in, and views only the balances and expenses that belong to their own account and the groups they joined. This gives each person a private place to track expenses while still participating in shared group ledgers.

**Why this priority**: Without secure account access and personal balance visibility, all other group and payment features would be unreliable and unsafe.

**Independent Test**: A new user can create an account, log in, and successfully view their personal summary and group membership without seeing another user’s private financial data.

**Acceptance Scenarios**:

1. **Given** a new person with no existing account, **When** they register with valid credentials and confirm the account, **Then** they receive a personal account and can log in successfully.
2. **Given** a logged-in user who belongs to multiple groups, **When** they open their dashboard, **Then** they see only their own balances, payments, and relevant group summaries and not other users’ private account data.

---

### User Story 2 - Manage group expenses and split costs fairly (Priority: P1)

A group organizer or member creates an expense for a trip, household, or team activity, selects who participated, and chooses an equal or percentage split. The system calculates each user’s share and updates balances only after the expense is approved.

**Why this priority**: Shared-expense tracking is the core value of the product; accurate split logic is essential to trust and adoption.

**Independent Test**: A group member can submit a shared expense with a date, description, amount, and participants; after approval, the system calculates the correct balances for each user.

**Acceptance Scenarios**:

1. **Given** a group with three users, **When** one user submits a $150 expense split equally and the approver approves it, **Then** each participant’s share is $50 and the payer is credited or debited correctly according to the settlement logic.
2. **Given** a group with three users and a 50/30/20 expense split, **When** the expense is submitted and approved, **Then** each participant’s balance is calculated from the specified percentages and shares total the original amount.
3. **Given** a user uploads a receipt image with an expense, **When** the expense is created, **Then** the receipt is attached to the expense record and is visible to authorized group members.

---

### User Story 3 - Review and approve expense entries before settlement changes are finalized (Priority: P1)

A designated single approver reviews a new expense before it affects balances. This allows the group to catch incorrect amounts, duplicates, or missing participants before the ledger changes.

**Why this priority**: Approval gates reduce disputes and protect the integrity of the financial records in shared groups while keeping the first release simple and predictable.

**Independent Test**: The designated approver can review a flagged expense, approve or reject it, and the system only changes balances after the approval is granted.

**Acceptance Scenarios**:

1. **Given** a user submits an expense awaiting review, **When** the designated approver rejects the entry, **Then** no balance is updated and the expense remains in a rejected state with review notes.
2. **Given** a submitted expense that meets policy, **When** the designated approver approves it, **Then** the group ledger updates and the expense becomes active in the settlement history.

---

### User Story 4 - Track payment settlements and review financial reports (Priority: P2)

A user records a payment they made to settle a balance, and later reviews custom reports showing expenses, balances, and payment history over selected date ranges. This keeps the ledger current and supports reconciliation.

**Why this priority**: Accurate payment tracking and reporting are central to trust, especially for roommates, travelers, and teams that need regular reconciliation.

**Independent Test**: A user can record a payment, see balances update, and generate a summary report for a chosen date range.

**Acceptance Scenarios**:

1. **Given** a user has a balance owed to another group member, **When** they record a payment against that balance, **Then** the system updates the outstanding balance to reflect the payment.
2. **Given** a user requests a report for a date range and specific participants, **When** the report is generated, **Then** it shows the relevant expenses, payments, and remaining balances without exposing unrelated data.

---

### Edge Cases

- What happens when a group member is removed or leaves the group after an expense is created?
- How does the system handle an expense that is split by percentages that do not total 100%?
- What happens when a payment exceeds the outstanding balance for a user or group member?
- How does the system handle duplicate receipt uploads or invalid dates?
- What happens if a user attempts to approve their own expense in a single-approver workflow?
- What happens when a designated approver is unavailable or does not act on a pending expense within a defined review window?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow each user to create a unique account with secure login and personal access to their own expense and balance records.
- **FR-002**: The system MUST allow users to create groups and add or remove members as needed for shared activities such as roommates, trips, or events.
- **FR-003**: The system MUST allow a user to create an expense with an amount, date, description, participant list, split method, and optional receipt image.
- **FR-004**: The system MUST support at least two expense-splitting methods: equal split and percentage split.
- **FR-005**: The system MUST calculate each participant’s share based on the selected split method and update balances for all affected users only after the designated approver approves the expense.
- **FR-006**: The system MUST notify group members when a new expense is created or updated in a group they belong to.
- **FR-007**: The system MUST require a single designated approver to review and approve each expense before it becomes active in the ledger for balance adjustments.
- **FR-008**: The system MUST retain sufficient metadata for each expense and approval decision to support auditability and dispute resolution.
- **FR-009**: The system MUST track each user’s outstanding balances with every other participant in a group and keep those balances current as expenses and payments change.
- **FR-010**: The system MUST allow users to record payments made to settle balances owed and update the remaining balance after payment entry.
- **FR-011**: The system MUST provide summaries and reports covering expenses, balances, and payments for a user or group over selected date ranges and criteria.
- **FR-012**: The system MUST provide a primary web experience and mobile app experiences for Android and iOS that support the core expense, group, notification, and reporting flows through the shared versioned API.
- **FR-013**: The system MUST prevent unauthorized users from viewing or modifying another user’s private account information or unrelated group balances.
- **FR-014**: The system MUST reject or flag invalid expense entries such as a malformed date, missing amount, invalid percentage totals, or missing participants.
- **FR-015**: The system MUST provide a Java Swing desktop client for core account, group, expense, approval, payment, notification, balance, and reporting workflows. The desktop client MUST use the shared versioned API and MUST NOT connect directly to persistence or implement authoritative financial rules.

### Key Entities

- **User Account**: Represents a person using the system, including their login credentials, group memberships, personal balances, and payment history.
- **Group**: Represents a shared context such as roommates, a trip, or a coworking group, containing members and shared expenses.
- **Expense**: Represents a recorded cost event with amount, date, description, participants, split method, receipt, and approval status.
- **Expense Participant**: Represents a user’s involvement in a specific expense and their share of the total cost.
- **Ledger Event and Balance Projection**: Immutable financial events record approved expense/payment postings; read-only balance projections summarize amounts owed and can be rebuilt from those events.
- **Payment Record**: Represents a settlement action that reduces or clears an outstanding balance between users.
- **Notification**: Represents a message sent to group members when relevant expense or payment activity occurs.
- **Approval Record**: Represents a review decision for an expense, including approver identity, outcome, and timestamps.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: New users can create an account, join a group, and view their expense summary in under 2 minutes during a standard onboarding flow.
- **SC-002**: Group members can create a shared expense and see the correct balance updates for all affected users within 5 seconds of approval.
- **SC-003**: At least 95% of users can complete the primary shared-expense workflow without support intervention during a test session.
- **SC-004**: Approved expenses and payment records remain consistent across all views, with no unexplained balance mismatch in end-to-end reconciliation checks.
- **SC-005**: Users can generate custom expense and balance summaries for chosen date ranges and receive reports that match the underlying records for the same period.
- **SC-006**: The web, mobile, and Swing experiences support the same core flows for account access, group management, expense entry, notifications, and reporting without blocking critical user tasks.
- **SC-007**: The Swing desktop client uses the shared API and supports the same core financial rules and authorization outcomes as the web and mobile clients.

## Assumptions

- Users have a stable internet connection and can access the product through a browser or mobile app.
- For the initial version, the system is expected to manage group-based settlement logic without complex multi-currency conversion.
- The organization uses a single designated approver per group for expense review in v1; the approver is responsible for approving or rejecting new expenses before they affect balances.
- Group membership is managed directly within the application and requires explicit user enrollment for shared expense visibility.
- Payment records are treated as settlement events within the platform and are not integrated with external banking or payment processing in the first release.
