# Quickstart: SplitSmart Validation Guide

## Prerequisites

- .NET 8 SDK
- Node.js 20+
- Docker Desktop or equivalent for Postgres and MinIO
- Mobile dev environment for iOS/Android if validating native flows

## Suggested Local Setup

1. Start the required infrastructure:
   - PostgreSQL
   - MinIO or S3-compatible storage for receipt uploads
   - Optional Redis for caching or live notifications

2. Restore and build the backend:
   - `dotnet restore`
   - `dotnet build`

3. Apply database migrations:
   - `dotnet ef database update --project backend/src/Infrastructure`

4. Start the API:
   - `dotnet run --project backend/src/API`

5. Start the web app:
   - `npm install`
   - `npm run dev`

6. Start the mobile app if needed:
   - `npm install`
   - `npx expo start`

## Validation Scenarios

### Scenario 1: Account creation and personal dashboard

Goal: confirm a user can register, log in, and see only their own relevant data.

Steps:
1. Register a new account.
2. Log in with valid credentials.
3. Open the dashboard.
4. Verify the user sees their balances, activity, and group membership without another user’s private data.

Expected outcome:
- Account creation succeeds.
- Dashboard only includes the authenticated user’s scope of data.

### Scenario 2: Group creation and member add/remove

Goal: confirm shared financial contexts are explicit and permissioned.

Steps:
1. Create a group named “Trip 2026”.
2. Add at least two members.
3. Remove one member and confirm the group ledger and membership state update correctly.

Expected outcome:
- Allowed group members see the shared ledger.
- Removed members lose access to future group data according to policy.

### Scenario 3: Equal split expense flow

Goal: confirm a shared expense calculates and updates balances correctly.

Steps:
1. Create a group with three members.
2. Create a $150 expense with an equal split.
3. Submit the expense for approval.
4. Approve the expense as the designated approver.

Expected outcome:
- Each participant’s share is $50.
- Balance updates are applied only after approval.
- The ledger remains consistent with the original transaction.

### Scenario 4: Percentage split validation

Goal: verify percentage-based splits and validation errors.

Steps:
1. Create a group with four members.
2. Submit an expense using 50/30/20 split values.
3. Attempt to submit an invalid total such as 50/30/10.

Expected outcome:
- Valid percentage totals are accepted and balances match the original amount.
- Invalid percentage totals are rejected with a clear error.

### Scenario 5: Payment settlement and reporting

Goal: ensure payment records reduce outstanding balances and custom reports reflect the correct time range.

Steps:
1. Create an open payable balance.
2. Record a payment against it.
3. Generate a report for a date range that includes the payment and expense.

Expected outcome:
- Outstanding balance is reduced correctly.
- Reports show only the selected group and time range.

### Scenario 6: Notification behavior

Goal: confirm group members are notified when relevant activity occurs.

Steps:
1. Create an expense in a group.
2. Verify the relevant members receive the notification.
3. Open the notification center and mark the item as read.

Expected outcome:
- Only relevant users are notified.
- The activity appears in the app feed and/or push payload.

## Definition of Done for the Feature

The implementation is ready to move to development tasks when:
- account access rules pass role and membership checks,
- approval is enforced before any balance update,
- equal and percentage splits are covered by automated regression tests,
- payment adjustments reconcile to the underlying expense record,
- reporting output matches the same date range and group filters used in the input data.
