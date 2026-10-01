# Background Jobs

The application has a separate scheduler service for work that should happen without a customer or staff member waiting for an HTTP request.

The scheduler runs on Railway and connects to the same MongoDB database as the main application.

The main application also has a small number of asynchronous tasks that run after a request has already returned.

## Scheduler

The scheduler has two cron entry points.

The main recurring work includes appointment reminders, no-show handling, reward processing, and cleanup.

```text id="u6r2k9"
Railway Cron
     |
     +----------------------+
     |                      |
     v                      v
Morning Job             Evening Job
     |                      |
     |                      +--> Tomorrow reminders
     |                      +--> No-show processing
     |                      +--> Staff summary
     |                      +--> Reward reconciliation
     |                      +--> Reward expiry warnings
     |                      +--> Media cleanup
     |
     v
Today's appointment reminders
```

## Morning Reminder

The morning job processes the day's upcoming appointments.

For each appointment, it sends the customer reminder and builds the staff daily schedule summary.

Each email send is isolated in its own error handler so one failed message does not stop the remaining reminders.

The job records counts and exits non-zero for fatal failures.

There is no automatic retry of an individual failed email. The next scheduled run occurs the following day.

## Evening Processing

The evening job handles several independent operations.

### Tomorrow reminders

Customers with appointments scheduled for the following day receive reminder emails.

### No-show processing

Appointments that are still in the `upcoming` state after the relevant time are moved into the missing/no-show flow.

Customers receive the appropriate notification and staff receive a summary.

### Reward reconciliation

No-show appointments can affect rewards.

The scheduler performs the required reward return or credit removal depending on the appointment state.

### Reward expiry

The scheduler identifies rewards approaching expiration and sends warning messages.

### Media cleanup

Old message images are removed from Cloudinary after the retention period.

The cleanup process does not blindly trust an image identifier stored from the client.

Unverified identifiers are hidden rather than deleted.

## Failure Isolation

A scheduler run contains several unrelated operations.

A failure in one record should not stop processing the rest of the records.

For example:

```text id="z4h8p2"
Appointment A
    |
    +--> success

Appointment B
    |
    +--> email failed
    |
    +--> log failure
    |
    +--> continue

Appointment C
    |
    +--> success
```

This is especially important for reminder jobs where one invalid email address should not prevent other customers from receiving their reminders.

## System Audit Events

Automated changes are not treated differently from staff changes when they need to be auditable.

The scheduler records system-generated audit entries for relevant actions.

The audit actor is represented as the system rather than as a human staff account.

## Campaign Sender

Marketing campaigns are sent by an in-process campaign sender.

The campaign API can respond without waiting for the entire recipient list to finish.

Recipients are processed in batches of 10 with a one-second delay between batches.

Each recipient has a stored status and provider message ID.

If an individual recipient fails, the failure is recorded rather than stopping the complete campaign.

Failed recipients are not automatically resent.

## Post-Response Webhook Work

Some webhook requests need to update the database immediately but also trigger secondary notifications.

The application performs the critical webhook work first and sends the provider a successful response.

Secondary work such as staff alerts and email notifications happens afterward.

This keeps the webhook response path short while still allowing the application to perform the additional work.

## Audit Writer

Audit entries are written asynchronously.

The audit writer is designed not to slow down the main request.

If an audit insert fails, the failure is logged rather than thrown back through the user's request.

The audit writer also emits the relevant real-time activity events.

## Database TTL Cleanup

MongoDB handles several cleanup tasks using TTL indexes.

These include:

- Sessions
- Pending orders
- Administrative tokens
- Alerts
- Audit records
- Inbound email logs

This removes the need for a separate application job for each expiration rule.

## Scheduler Design

The scheduler is intentionally separate from the HTTP application.

The main web process handles requests and real-time connections.

The scheduler handles recurring work.

This keeps long-running or recurring operational tasks from being tied to a user's request and gives the background processing its own deployment and runtime configuration.