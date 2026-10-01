# Security

Security was an ongoing part of building and maintaining the application.

The system handles customer accounts, staff accounts, appointments, payments, email conversations, uploads, and personal information, so I had to consider both normal application behavior and what could happen when requests were manipulated or repeated.

This document focuses on the security controls that are actually implemented. It does not claim that the application has no remaining security work.

## Authentication

The application uses server-side sessions with MongoDB as the session store.

Security measures include:

- Password hashing with bcrypt
- Session ID regeneration during login
- Rolling idle timeouts
- Session expiration through TTL
- Per-account session revocation
- Account lockout
- Rate limiting
- Password-reset protection
- Single-use invitation and reset tokens

Unknown account lookups also perform a dummy bcrypt comparison so the application does not give away whether an email address belongs to an account through response timing.

## Authorization

Authorization is enforced on the server.

Staff accounts have different permission levels, and management actions go through a centralized authorization rule.

The application does not treat frontend visibility as permission.

A user who manually constructs a request still has to pass the same server-side authorization checks.

The database also enforces singleton rules for the root and developer roles.

Member routes use the authenticated member identity rather than trusting a member ID supplied by the browser.

## Session Revocation

Normal session expiration is not enough for some security events.

The application therefore maintains a per-account token version.

When the version changes, existing sessions become invalid.

The same version is checked for WebSocket connections.

This allows actions such as:

- Signing out other devices
- Password changes
- Password resets
- Pausing an account
- Removing an account

to invalidate active access.

## Token Security

Invitation and password-reset tokens are generated using cryptographically secure random values.

Only SHA-256 hashes are stored in the database.

Tokens are:

- Single purpose
- Expiring
- Revoked when appropriate
- Consumed atomically

This means the database does not contain a directly usable invitation or password-reset link.

Appointment management tokens are also compared in constant time.

## Query and Input Validation

Administrative search fields are checked against a whitelist instead of being allowed to choose arbitrary database fields.

Inputs are coerced and validated before being used.

Regular-expression input is escaped where user-controlled text is used in regex queries.

Length limits are also applied to areas such as:

- Names
- Emails
- Passwords
- Addresses
- Inbound email text
- Audit details
- Request bodies

## XSS Protection

The application displays customer names, messages, product information, and other user-controlled content in staff pages.

These values are escaped before being inserted into HTML.

Inbound email is stored as plain text and escaped when displayed.

The application also uses shared escaping helpers for user-controlled values placed into HTML email templates.

Several stored-XSS paths in the dashboard were identified and fixed during development.

## Email Header Protection

Customer-controlled subjects can eventually become email headers.

The application cleans subject values by removing control characters such as CR, LF, and tabs and applying a length limit.

This prevents a customer from injecting additional email headers through a ticket subject.

## File Upload Security

Some uploads go directly to the media provider.

The browser reports the resulting asset ID, but the application does not automatically trust that ID for destructive operations.

Before an asset can be deleted, the server verifies information from the provider, including:

- Correct cloud account
- Asset metadata
- Recent creation time
- Provider-controlled values

If verification cannot be completed, the asset ID is not trusted for deletion.

This was particularly important because an attacker could otherwise submit another asset's identifier and potentially cause the cleanup process to delete it.

## Attachment Handling

Customer email attachments are not all served as inline browser content.

Only supported raster images are displayed inline.

Other file types are forced to download.

The response also uses:

- `nosniff`
- A sandbox content-security policy
- Cleaned filenames

The goal is to prevent an uploaded HTML or SVG file from executing in the application's origin.

## Rate Limiting

Rate limits are applied to sensitive endpoints including:

- Authentication
- Account operations
- Token endpoints
- Appointment lookup
- Checkout
- Cart operations
- Order tracking

The application is configured to trust the hosting proxy so the limiter can use the actual client IP.

## CSRF

The application uses cookie-based sessions with `SameSite=Lax` and restricts state-changing actions to methods such as POST, PATCH, and DELETE.

This provides basic cross-site request protection.

The application does **not** currently use explicit CSRF tokens, so this remains an area for future hardening.

## Webhook Security

External providers are not allowed to change application state simply by reaching a webhook URL.

Stripe webhooks are verified using the Stripe signature against the raw request body.

Brevo webhook and inbound-email authentication uses configured secrets and constant-time comparison.

Webhook authentication fails closed when the required secret is missing or too short.

Secrets are checked before the body is processed and are never logged.

Inbound customer replies also require a valid HMAC-signed reply address.

## Payment Security

The server recalculates product prices instead of trusting client-side totals.

Customers enter card information on Stripe's hosted checkout page, so card data does not pass through the application.

Orders are created only after a verified payment event indicates the payment is paid.

The amount used for the order comes from the verified payment event.

## Database Consistency

Security is not only about authentication.

The application also uses database-level protections against race conditions and duplicate operations.

These include:

- Transactions
- Unique indexes
- Partial unique indexes
- Conditional atomic updates
- Atomic token consumption
- Atomic stock updates
- Idempotent webhook handling

This is particularly important for payment, inventory, authentication tokens, and administrative actions.

## Data Minimization

The real-time system sends invalidation information rather than customer records.

Activity events also avoid unnecessary personal information.

Audit records store identifiers and short labels instead of copying complete customer records.

Password hashes are excluded from normal queries and scrubbed from serialized output.

Webhook secrets and authentication values are never written to logs.

## Secrets

Production secrets are supplied through environment variables.

`.env` files are excluded from version control.

Production maintenance scripts receive the production connection string through a shell-scoped environment variable rather than storing it in source code.

### Important repository history note

The original production Git history is **not safe to publish**.

Earlier commits contained hard-coded database credentials and session secrets.

For that reason, the public portfolio should be built from a fresh sanitized repository rather than publishing the original Git history.

Any credentials that may still be valid should also be rotated before the public repository is created.

## Known Security Gaps

The application still has security work that should not be hidden from the portfolio.

The current report identifies areas including:

- Some public token lookup paths still needing the same query protections
- Mass-assignment risk around booking data
- No server-side double-booking guard
- Member OTP weaknesses
- Missing rate limits in some areas
- Potential regex denial-of-service concerns
- An unsubscribe fallback that needs stronger secret handling and constant-time comparison
- Some provider/customer strings in staff emails that need additional escaping
- No dedicated security-header middleware
- No explicit CSRF token implementation
- Some timezone-related edge cases
- Stock administration using read-modify-write behavior

These are documented here intentionally.

The goal of this portfolio is to show the engineering work accurately, including where the system can still be improved.