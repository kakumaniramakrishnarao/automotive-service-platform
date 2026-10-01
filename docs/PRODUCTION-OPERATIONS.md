# Production Operations

The application is deployed on Railway and uses separate services for the web application and scheduled background work.

The production setup is intentionally simple: the web service handles customer and staff traffic, while the scheduler handles recurring operational tasks.

## Deployment

The main application runs as an always-on Railway web service.

The scheduler runs separately through Railway cron services.

```text
                         ┌─────────────────────┐
                         │       Railway       │
                         │                     │
Customers / Staff ──────>│   Web Application   │
                         │                     │
                         └──────────┬──────────┘
                                    │
                                    v
                              MongoDB
                                    ^
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         │     Scheduler       │
                         │    Cron Services    │
                         │                     │
                         └─────────────────────┘
```

The main application serves both public domains from the same Node process.

The scheduler is deployed from its own project and connects to the same database.

## Environment Configuration

Production configuration is supplied through environment variables.

This includes:

- Database connection information
- Provider credentials
- Webhook secrets
- Application URLs
- Email configuration
- Scheduler timezone
- Production/test behavior flags

Secrets are not intended to be stored in source code.

A real-mail flag is also used so local and test environments can exercise email-related code without accidentally sending messages to real customers.

## Local and Production Behavior

The application is designed to run locally over HTTP as well as behind Railway's HTTPS proxy.

Proxy-aware configuration allows the application to determine the correct request security context.

Cookies use automatic secure behavior so the same application can work in local development and production without hard-coding one environment's behavior.

Tests use a separate database.

## Webhooks

External webhooks are treated as security-sensitive entry points.

The application:

1. Receives the provider request.
2. Verifies the required authentication/signature.
3. Rejects requests when the required secret is missing.
4. Processes only verified events.
5. Uses idempotent handling where providers can retry events.

For payment webhooks, the response path is kept short and secondary work happens afterward where possible.

## Logging

The application uses structured console logging with status markers.

Logs are used primarily for:

- Request outcomes
- Background-job counts
- Email outcomes
- Webhook processing
- Payment exceptions
- Operational failures

Sensitive authentication values are not intentionally logged.

The application does not currently use a dedicated APM or external monitoring platform.

## Operational Alerts

The staff dashboard provides operational alerts for important payment-related situations.

Examples include:

- An unpaid checkout completion
- An out-of-stock condition after payment
- A refund associated with an unknown order
- A payment dispute

Owner email notifications are also used for selected operational events.

For inbound email, an outcome log records how the message was processed, which helps when debugging provider or threading issues.

## Database Maintenance

MongoDB indexes are created and maintained through dedicated maintenance scripts.

The scripts check the existing state before applying changes.

Other maintenance utilities are used for tasks such as:

- Administrative bootstrap
- Root-account recovery
- Index management

Production connection information is passed to these scripts through a shell-scoped environment variable rather than being embedded in the scripts.

## Production Incidents and Fixes

Several production issues led to changes in the system.

### Idempotency window

The checkout idempotency window was originally too long.

That caused a legitimate repeat purchase to be treated as the same checkout request.

The window was shortened and the reason was documented.

### Static source exposure

The project root had previously been exposed through static-file serving.

That was removed so project files were no longer directly exposed.

### Upload ID trust

A cleanup path originally trusted a client-reported media ID.

The application was changed to verify the asset with the media provider before allowing destructive operations.

### Appointment lookup

Appointment confirmation information could previously be accessed by ID alone.

The route was changed to require the appropriate management token and return a reduced response.

### Missing admin authorization

Some administrative appointment routes were found without the expected authentication guard.

Those routes were secured.

These fixes are useful examples of how the application changed as real edge cases and security concerns were identified.

## Deployment Model

The normal deployment path is straightforward:

```text
Git push
   |
   v
Railway build
   |
   v
Deploy
   |
   v
Production service
```

The application does not currently have a CI pipeline.

Automated integration tests are available locally, but they are not currently connected to CI.

That is an identified improvement area rather than something hidden from the portfolio.