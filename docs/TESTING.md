# Testing

The project uses HTTP-level integration tests rather than relying only on isolated unit tests.

The tests run against a locally running application and a dedicated test database.

Real email is disabled during test runs.

## How the Tests Work

A typical test does something close to this:

```text id="t8q3m6"
Create test data
      |
      v
Make real HTTP request
      |
      v
Use cookies / forms / JSON
      |
      v
Check HTTP response
      |
      v
Check MongoDB state
      |
      v
Check side effects
```

The tests exercise the real application routes rather than calling internal functions directly.

Test groups also use separate spoofed client IP addresses so rate-limit tests do not interfere with one another.

## Test Coverage

There are 16 automated suites.

They cover areas including:

### Authentication

- Forgot-password flow
- Password reset
- Token status
- Enumeration resistance
- Reset timing
- Cooldowns
- Token reuse
- Concurrent reset attempts

### Staff Invitations

- Valid invitations
- Invalid invitations
- Concurrent invitation acceptance
- Link invalidation

### Staff Management

- Role-based access
- State transitions
- Root/developer rules
- Concurrent pause operations
- Rate limiting

### Account Management

- Name changes
- Password changes
- Sign-out-other-devices
- Session revocation

### Legacy Login Removal

Tests verify that the old shared login is rejected while personal staff accounts continue to work.

### Protected Pages

Protected pages are tested for authentication gating and redirects.

### Audit and Activity

Tests cover:

- Audit creation
- Read-only audit behavior
- Activity filtering
- Pagination
- Account events
- Appointment events
- Member events
- Message events
- Schedule events
- Shop and order events
- Campaign events
- Scheduler events

The appointment tests also include parallel creation scenarios.

### Real-Time Activity

Socket.IO client tests verify live activity events and check that the payloads do not expose unnecessary personal information.

## Concurrency Testing

Some of the more useful tests are not simple happy-path tests.

The project includes concurrent tests for operations where two requests can arrive at nearly the same time.

Examples include:

- Password reset token consumption
- Invitation acceptance
- Staff account changes
- Appointment creation
- Session-related operations

The goal is to test what happens when the application receives requests simultaneously rather than only one after another.

## Test Size

The current test suite contains approximately:

- 16 automated suites
- ~450 assertion call sites
- ~3,300 lines of test code

These numbers are approximate because some assertions occur inside loops.

## What Is Not Automated

The project does not claim complete test coverage.

The current report specifically identifies missing automated tests for:

- Stripe checkout
- Stripe webhook processing
- Member authentication
- Booking validation

There is also:

- No unit-test framework
- No CI test pipeline

These are known areas for future work.

## Why Integration Tests

For this application, many important behaviors depend on several pieces working together.

For example, a staff authorization test can involve:

```text id="e6y1w4"
Session
  +
Role
  +
Express middleware
  +
Route
  +
MongoDB
  +
Response
```

Testing only an individual function would not verify the complete behavior.

The integration tests therefore focus on the behavior that users and external systems actually exercise.