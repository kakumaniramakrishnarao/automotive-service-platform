# Real-Time Updates

The staff dashboard supports live updates using Socket.IO.

The main goal was not simply to make the dashboard "real time." It was to let multiple staff members work at the same time without sending unnecessary data through WebSockets or overwriting someone's work while they were editing a record.

## Basic Flow

```text id="lq4m5h"
Staff A changes a record
        |
        v
Express route validates + saves
        |
        +--> Audit entry
        |
        +--> Alert if needed
        |
        +--> data_changed event
                    |
                    v
              Admin room
                    |
          +---------+---------+
          |                   |
          v                   v
      Staff B             Staff C
          |                   |
          +------ refetch ----+
                  |
                  v
           Updated data
```

The WebSocket message does not contain the changed customer record.

It tells the client which entity type changed. The page then fetches the current data through the normal authenticated HTTP route.

## Why I Used Invalidation Events

One option would have been to send complete database records through Socket.IO whenever something changed.

I chose not to do that.

The socket payloads are intentionally small and contain only the information needed to decide whether a page should refresh.

For example:

```text id="wq0d0m"
data_changed
    |
    +--> entity: appointment
    +--> timestamp
```

The client already knows how to retrieve the current appointment data through the authenticated API.

This has two useful properties:

- Customer information does not need to be broadcast through the socket.
- The HTTP API remains the source of truth for the current state.

## Socket Authentication

Only authenticated staff accounts can establish an administrative socket connection.

The Socket.IO handshake uses the same Express session and checks the account state and session token version.

Failed connections are disconnected immediately.

Authenticated sockets join a single admin room, so broadcasts are scoped to staff connections rather than being sent to every connected socket.

## Events

The application currently uses several server-to-client events.

### `new_alert`

Used for notification bell items such as:

- New appointments
- Customer messages
- Orders
- Appointment changes
- Payment anomalies

### `data_changed`

Used as a lightweight invalidation signal.

The application maintains a whitelist of entity types that can be broadcast.

The payload contains entity information and a timestamp rather than customer data.

### `activity`

Used for live activity notifications.

The payload is intentionally limited to non-personal information.

### `stock_updated`

Used for live inventory changes on product administration pages.

### `ping`

An application-level keepalive is used to prevent idle connections from being dropped by the hosting proxy.

## Reconnection

The client uses one shared socket per page.

The same connection is used by the notification bell and live-sync system instead of creating separate connections for each feature.

When the connection is lost, the client reconnects using exponential backoff.

After reconnecting, registered pages refetch their data because changes may have occurred while the connection was unavailable.

## Hidden Tabs

A browser tab does not need to continuously refresh data when the user cannot see it.

When an admin page is hidden, the live-sync layer marks the page as needing an update rather than making unnecessary network requests.

When the page becomes visible again, it refreshes if needed.

This also helps account for changes made by the background scheduler, which cannot send Socket.IO events itself.

## Debouncing and Single-Flight Refreshes

Several changes can happen close together.

For example, creating an appointment might result in multiple related notifications.

The client therefore debounces refresh requests rather than immediately starting a new request for every event.

The sync layer also allows only one refresh to run at a time.

If another update arrives while the refresh is running, the page can perform another refresh afterward instead of creating multiple concurrent requests.

## Protecting Unsaved Work

This was one of the more important parts of the implementation.

A live refresh can be harmful if someone is currently editing a record.

For example:

```text id="g0x2tu"
Staff member opens record
        |
        v
Starts editing
        |
        v
Another staff member changes record
        |
        v
Socket event arrives
        |
        v
Should the page refresh immediately?
        |
       NO
        |
        v
Show "New updates"
        |
        v
Refresh when the user is finished
```

Each page can provide an `isBusy()` condition.

Depending on the page, this can mean:

- An edit dialog is open
- A drawer is open
- The user has unsaved changes
- The user is interacting with the table
- A stock value is being edited
- The user has moved beyond the first page of results

While the page is busy, the refresh is held.

The user can then apply the update through the "New updates" control, or the refresh can happen once the page becomes idle.

## Reconnect Catch-Up

A WebSocket connection is not a durable event queue.

If the connection is down for several seconds, the client may miss events.

Instead of trying to reconstruct every missed event, the application simply refetches the affected views after reconnecting.

This makes the system easier to reason about:

```text id="c5q4e7"
Connection lost
      |
      v
Changes happen
      |
      v
Connection restored
      |
      v
Refetch current state
      |
      v
Current data is restored
```

## Session Revocation

The real-time layer also participates in account session revocation.

Each staff account has a token version.

When a security-sensitive account action occurs, the version is changed.

Both normal HTTP requests and new WebSocket handshakes compare the session's version against the current account version.

Existing sockets can also be disconnected directly.

This means actions such as password changes, account removal, account pausing, and signing out other devices can take effect across both HTTP and WebSocket connections.

## Scale

The shared live-sync system currently supports 11 staff dashboard pages.

The implementation includes:

- One socket per page
- Admin-room broadcasts
- Reconnect handling
- Exponential backoff
- Reconnect catch-up
- Hidden-tab deferral
- Debouncing
- Single-flight refreshes
- Edit protection
- Changed-row highlighting

The system deliberately favors a simple invalidation-and-refetch model over maintaining a second real-time representation of the database in the browser.