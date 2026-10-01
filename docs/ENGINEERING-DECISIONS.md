# Engineering Decisions

This document focuses on problems that required more than simply adding another feature.

The decisions below came from issues that appeared while building and maintaining the production system.

---

## 1. Exactly-Once Order Creation from Retried Webhooks

### Problem

Payment providers can deliver the same webhook more than once.

Two copies can also arrive at nearly the same time.

For an order system, processing both could:

- Create duplicate orders
- Decrement inventory twice
- Produce inconsistent payment state

### Decision

Order creation is performed inside a MongoDB transaction.

The transaction includes:

1. Checking whether the Stripe session has already been processed
2. Conditionally decrementing inventory
3. Creating the order

A partial unique index on the Stripe session ID provides another database-level protection against duplicates.

If two requests race, only one can successfully create the order.

The losing transaction is rolled back, including any inventory changes made inside that transaction.

Side effects happen only after the transaction commits.

### Tradeoff

MongoDB transactions require a replica-set deployment.

An out-of-stock condition after a successful payment is treated as an operational exception and handled through a recorded manual refund rather than attempting an automatic refund.

---

## 2. Checkout Idempotency Without Trusting the Browser

### Problem

Customers can double-click checkout buttons.

Network retries can also cause the same request to arrive more than once.

Stripe also requires an idempotency key to correspond to an identical request body when reused.

### Decision

The application creates the idempotency key on the server.

The key is derived from the request content and a short time bucket.

A staged pending order can then be reused when the same logical checkout request is retried.

This allows the application to control idempotency instead of trusting a value supplied by the browser.

### What Changed

The first version used a longer idempotency window.

That eventually blocked a legitimate repeat purchase.

The window was deliberately shortened after the issue was identified.

This is a good example of a production design changing based on actual behavior rather than assuming the first implementation was correct.

---

## 3. Live Dashboard Without Overwriting Staff Edits

### Problem

Multiple staff members can work in the dashboard at the same time.

A naive real-time implementation could refresh a page while someone is editing a form.

That can:

- Move focus
- Replace form values
- Make an edit disappear
- Send unnecessary customer information over WebSockets

### Decision

Socket.IO is used only to tell the browser that something changed.

The socket message does not contain the changed customer record.

Instead:

```text
Database changes
       |
       v
Socket.IO
       |
       |  "appointment changed"
       v
Authenticated browser
       |
       v
HTTP refetch
       |
       v
Updated page
```

A shared client-side synchronization layer handles:

- Debouncing
- Single-flight refreshes
- Reconnects
- Reconnect catch-up
- Hidden-tab deferral
- Stale-tab refresh
- Edit protection

When a user is actively editing, the page can hold the refresh and show that new updates are available.

### Tradeoff

The browser performs another authenticated HTTP request instead of receiving a complete data delta through the socket.

That creates some additional requests, but it keeps the real-time layer simpler and avoids putting customer data into socket payloads.

---

## 4. Global Session Revocation

### Problem

A cookie session can remain valid even after a security-sensitive account action.

Open WebSocket connections create another problem because they can outlive the HTTP request that originally authenticated them.

### Decision

Each account has a token version.

The version is checked during normal requests and WebSocket authentication.

Security-sensitive operations can increment the version.

Existing sessions then fail the version check.

Open sockets are also disconnected directly.

### Result

Actions such as password resets, account pauses, and signing out other devices can take effect without waiting for the normal session expiration period.

---

## 5. Two-Way Email Conversations

### Problem

Customers need to reply to support messages from normal email clients.

The system therefore needs to determine which ticket an incoming email belongs to.

A plain public ticket ID would be too easy to guess or manipulate.

The system also has to handle provider retries, duplicate messages, attachments, and auto-responders.

### Decision

Each ticket receives an HMAC-signed reply address.

When an inbound message arrives, the signature is verified before the ticket is accepted.

The system also uses:

- Deduplication keys
- Conditional thread updates
- Email threading headers
- Safe subject handling
- Attachment safety checks
- Outcome logging

### Result

Customers can reply through their normal email client while the application keeps the conversation attached to the correct ticket.

Repeated provider deliveries can be handled without creating duplicate thread entries.

---

## 6. Append-Only Audit Trail

### Problem

The system needs to answer a simple operational question:

> Who changed this?

The audit history also needs to remain useful when changes are made automatically by the scheduler.

### Decision

The application writes audit entries asynchronously.

The audit model prevents normal update and delete operations.

Retention is handled through a TTL policy rather than application-level deletion.

The scheduler records itself as the system actor.

Audit details are also size-limited and scrubbed so the audit trail does not become a second copy of customer data.

### Result

The system records more than 35 distinct audited action types across areas such as:

- Accounts
- Appointments
- Membership
- Scheduling
- Products
- Orders
- Messages
- Campaigns

---

## 7. Verifying Upload IDs Before Destructive Operations

### Problem

Some images are uploaded directly to the media provider.

The browser receives the resulting asset ID.

A cleanup process later uses that ID when deleting old media.

Trusting the browser-provided ID directly creates a destructive path: a manipulated request could provide an ID belonging to another asset.

### Decision

Before allowing deletion, the server verifies the asset with the provider.

The verification checks that the asset belongs to the expected cloud account and was created recently.

The application stores provider-controlled values after verification.

If verification cannot be completed, the asset is not trusted for deletion.

### Result

The application can keep direct-to-cloud uploads while avoiding blind trust in a client-provided identifier during destructive operations.

---

## 8. Replacing a Shared Staff Login

### Problem

The original system used a shared staff login.

That made it difficult to answer:

- Which employee performed an action?
- Which account should be disabled?
- Which sessions should be revoked?
- Which permissions should apply?

### Decision

The system moved to individual staff accounts with role-based authorization.

The migration also had to avoid interrupting existing staff access.

The application supports the newer account model while explicitly rejecting the legacy shared login.

### Result

Staff actions can be associated with individual accounts and controlled through the application's authorization rules.

---

## 9. Why These Decisions Matter

The important part of these decisions is not the specific technology.

MongoDB, Socket.IO, Stripe, and Brevo are tools.

The engineering work was deciding where trust belongs, how state changes under concurrency, what information crosses system boundaries, and what should happen when an external system retries a request or fails halfway through.

Those decisions shaped the application more than the framework choices did.