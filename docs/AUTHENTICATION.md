# Authentication and Authorization

The application has separate customer/member and staff authentication.

The staff system was originally based on a shared login. I later replaced that with individual database-backed accounts and role-based permissions.

That change affected much more than the login page. It required session handling, account lifecycle management, authorization rules, revocation, invitation flows, testing, and a staged production migration.

## Authentication Model

The application uses server-side sessions.

```text id="yn7q1c"
Browser
   |
   | Session cookie
   v
Express
   |
   v
MongoDB session store
   |
   v
Authenticated request
```

Sessions are stored in MongoDB rather than only in the application process.

This allows the session state to survive application deployments.

Cookies use `httpOnly` and `SameSite=Lax`, with secure cookies under HTTPS.

## Passwords

Passwords are hashed with bcrypt.

The staff implementation also accounts for bcrypt's input-length limitation.

Passwords are never stored as plaintext.

## Session Fixation

The session ID is regenerated when a user logs in.

This prevents an attacker from preparing or fixing a session identifier before authentication and then reusing it after the user logs in.

## Session Lifetime

Staff and member sessions use different rolling idle timeouts.

Session writes are also throttled when the session has not meaningfully changed.

Expired sessions are removed through a MongoDB TTL index.

## Staff Roles

The staff system has three levels:

```text id="a7f6w0"
Root
  |
  +-- Developer
  |
  +-- Admin
```

Root and developer accounts are unique roles.

The database itself enforces this with partial unique indexes, so the application is not relying only on application-level checks to prevent multiple root or developer accounts.

## Centralized Authorization

A central permission function determines whether an actor can perform an action against a target.

Conceptually:

```text id="j4y9c1"
canActOn(actor, target, action)
             |
             +--> Does the actor have permission?
             |
             +--> Is the target in a valid state?
             |
             +--> Are there role restrictions?
             |
             v
          Allow / Deny
```

The same authorization rules are used by the interface and server-side routes.

The server remains authoritative.

The browser cannot grant itself permission simply by displaying a button or constructing a request manually.

## Account Lifecycle

Staff accounts can move through an account lifecycle that includes:

- Invitation
- Invitation acceptance
- Activation
- Pause
- Resume
- Removal
- Password reset
- Re-invitation

Removed accounts can be re-invited while retaining their history.

## Invitation Tokens

Invitation and password-reset tokens are generated as random values.

Only their SHA-256 hashes are stored in the database.

Tokens are:

- Single-purpose
- Expiring
- Revoked when newer tokens are issued
- Consumed atomically

Atomic consumption matters when two requests try to use the same token at the same time.

Only one request should be able to successfully consume it.

## Account Lockout

Repeated failed login attempts increment an account-specific failure counter and can result in a timed lockout.

The application also performs a dummy bcrypt comparison for unknown accounts.

This keeps the work performed for an unknown email closer to the work performed for a real account and helps reduce account-enumeration signals.

## Password Reset

The staff password-reset flow intentionally avoids telling the requester whether an account exists.

The response is kept consistent, and the email work happens after the initial response.

A successful reset also revokes existing sessions and sockets.

## Session Revocation

A session can remain valid until its normal expiration unless there is a mechanism to explicitly revoke it.

I added a per-account `tokenVersion` for this purpose.

```text id="y1d2q3"
Account
  |
  +--> tokenVersion = 8
         |
         +--> Session A = 8
         +--> Session B = 8
         +--> Socket = 8

Security event
  |
  v
tokenVersion = 9

Session A -> invalid
Session B -> invalid
Socket    -> disconnected
```

This is checked during HTTP authentication and WebSocket authentication.

The "sign out other devices" flow can preserve the current session while invalidating older sessions.

## Replacing the Shared Login

The original staff system used a shared login.

I replaced it in stages rather than switching the production system all at once.

The migration included:

1. Adding database-backed staff accounts.
2. Introducing the new authentication path.
3. Adding invitations and password setup.
4. Adding account management and recovery.
5. Testing the personal-login workflow.
6. Removing the shared login.

This reduced the risk of disrupting staff access during the migration.

## Member Authentication

Members use a separate identity from staff.

The member system includes:

- Account registration
- Email verification
- Password authentication
- Password recovery
- Member-specific sessions
- Ownership checks on member resources

Member routes filter data by the authenticated member identity rather than accepting an arbitrary member ID from the browser.