# Architecture

## Overview

The application is a Node.js and Express application backed by MongoDB.

It serves the customer website, member portal, shop, and staff dashboard. A separate scheduler service runs recurring background jobs against the same database.

The system uses a traditional server-rendered architecture rather than a frontend framework or SPA. This kept the application relatively simple while still allowing features such as real-time staff updates, online payments, customer messaging, and background processing.

## Main Components

```text
                         Customers / Staff
                                |
                                | HTTPS
                                v
                    +------------------------+
                    |     Node.js / Express   |
                    |                         |
                    |  Routes + Middleware    |
                    |  Auth + Validation      |
                    |  Business Logic         |
                    |  Webhooks               |
                    +-----------+-------------+
                                |
             +------------------+------------------+
             |                  |                  |
             v                  v                  v
        +---------+       +-----------+       +-----------+
        | MongoDB |       | Socket.IO |       | External  |
        |         |       |           |       | Services  |
        +---------+       +-----------+       +-----------+
             ^                                  |
             |                         +--------+--------+
             |                         |        |        |
             |                       Stripe   Brevo   Cloudinary
             |
             |
      +------+----------------+
      | Separate Scheduler    |
      |                       |
      | Reminders             |
      | No-shows              |
      | Rewards               |
      | Cleanup               |
      +-----------------------+
```

## Application Structure

The main application is organized around Express routers, Mongoose models, middleware, shared utilities, and server-rendered pages.

The main codebase contains:

- 23 Express routers
- 16 Mongoose models
- 175 route handlers
- 49 server-rendered pages
- Shared browser JavaScript modules
- A separate scheduler service

The application also uses MongoDB-backed sessions so authentication state survives application deployments.

## Request Flow

A normal request follows roughly this path:

```text
Browser
   |
   v
Express
   |
   +--> Authentication / session checks
   |
   +--> Rate limiting
   |
   +--> Route validation
   |
   +--> Business logic
   |
   +--> MongoDB
   |
   +--> Audit / notification work
   |
   +--> Real-time invalidation
   |
   v
Response
```

For most administrative changes, the server does not send the entire updated record through Socket.IO.

Instead, the server:

1. Validates the request.
2. Saves the change.
3. Records the relevant audit event.
4. Sends a small real-time invalidation event.
5. Lets the affected admin page fetch the current data through the normal authenticated HTTP API.

This keeps the WebSocket payloads small and avoids sending customer information through the real-time channel.

## Two Public Domains

One Node.js process serves two public domains.

Host-based routing determines which page set should be served while the underlying application, database, authentication system, and business logic remain shared.

This allowed the service site and shop experience to remain separate from the user's perspective without maintaining two independent applications.

## External Services

The application integrates with several external services.

### Stripe

Used for:

- Hosted checkout
- Automatic tax
- Payment webhooks
- Refund and dispute information

Stripe webhooks are handled before the global JSON parser so the application can verify the original request body.

### Brevo

Used for:

- Transactional email
- Customer replies through inbound email
- Delivery events
- Marketing campaigns

### Cloudinary

Used for image storage and media operations.

Customer uploads are verified server-side before destructive operations are allowed.

### Railway

Used to run the main Node.js service and scheduled jobs.

## Database

MongoDB is used through Mongoose.

The application uses:

- Transactions
- Unique indexes
- Partial unique indexes
- TTL indexes
- Compound indexes
- Atomic updates
- Pagination
- Cursor-based paging
- MongoDB-backed sessions

Transactions are used when several related database changes need to succeed together.

For example, order creation and inventory changes are part of the same transaction.

## Background Processing

The scheduler is intentionally separate from the web application.

This means recurring jobs do not need to stay attached to a long-running HTTP request.

The scheduler connects to the same database and performs operations such as:

- Appointment reminders
- No-show processing
- Reward processing
- Reward expiry notifications
- Cleanup

Automated changes are also written to the audit trail as system actions.

## Why This Architecture

I did not start with a large collection of separate services.

The system is primarily a modular Node.js application because most of its functionality shares the same database, authentication model, and business domain.

Where work had different runtime requirements, I separated it out. The clearest example is the scheduler, which runs short-lived jobs independently from the web application.

The result is one main application with clear internal boundaries rather than a distributed system that would add operational complexity without a clear benefit for this project.