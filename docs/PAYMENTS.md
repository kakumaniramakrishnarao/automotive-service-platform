# Payments

The shop uses Stripe Checkout for customer payments.

The important design decision is that the browser is **not trusted to create a paid order**.

A successful redirect back to the application does not create an order. The order is created only after the server receives and verifies the corresponding Stripe webhook.

## Checkout Flow

```text
Customer
   |
   v
Shopping Cart
   |
   v
Server validates cart
   |
   +--> Recalculate prices
   +--> Check product availability
   +--> Validate shipping information
   |
   v
Create pending checkout state
   |
   v
Create Stripe Checkout Session
   |
   v
Customer pays on Stripe
   |
   v
Stripe webhook
   |
   +--> Verify signature
   +--> Verify payment status
   |
   v
MongoDB transaction
   |
   +--> Create order
   +--> Decrement inventory
   +--> Record payment state
   |
   v
Commit
   |
   v
Post-commit notifications
```

## Why the Success Page Does Not Create the Order

A browser redirect is not proof of payment.

A customer can close the browser, retry the request, manipulate client-side values, or reach the success URL without the payment actually being completed.

For that reason, the success page only waits for the server-side order state to appear.

The verified Stripe webhook is the source of truth for order creation.

## Server-Side Pricing

The browser sends the cart information, but the server does not simply accept the prices from the browser.

Before creating checkout, the server validates the cart against the current catalog.

This includes checking:

- Product identity
- Current pricing
- Available inventory
- Variants
- Shipping information

Stripe Tax is used for tax calculation.

This prevents the client from changing a product price or otherwise modifying the values that determine the final order.

## Webhook Idempotency

Stripe can deliver webhook events more than once.

The order creation path therefore needs to handle duplicate delivery safely.

The relevant database work happens inside a transaction:

```text
Webhook
   |
   v
Verify signature
   |
   v
Check payment state
   |
   v
Start transaction
   |
   +--> Check whether order already exists
   |
   +--> Decrement stock conditionally
   |
   +--> Insert order
   |
   v
Commit
```

A unique database constraint provides another layer of protection against duplicate order creation.

If two webhook deliveries race with each other, only one can successfully create the order. The losing transaction rolls back its inventory changes as well.

This is important because preventing duplicate orders is not enough by itself. A failed duplicate attempt must not leave the inventory incorrectly decremented.

## Checkout Idempotency

The checkout endpoint also needs protection against repeated requests.

A customer can double-click a button or retry after a network failure.

The application uses a server-derived idempotency key based on the request content and a short time window.

A staged pending order can then be reused when the same logical request is retried.

The time window was intentionally kept short. An earlier implementation used a window that was too long and prevented legitimate repeat purchases. That behavior was identified and corrected.

## Order State

After payment, the order moves through a controlled fulfillment workflow.

```text
placed
   |
   v
preparing
   |
   v
shipped
   |
   v
delivered
```

Cancellation is allowed from appropriate early states.

Carrier and tracking information can also be associated with the order.

Refunds and payment disputes are surfaced to staff so they can be reconciled with the order state.

## Handling Payment Exceptions

Not every payment problem can be resolved automatically.

For example, if payment succeeds but inventory becomes unavailable during the transaction, the application records the anomaly for staff rather than silently creating an inconsistent order.

This is an intentional tradeoff: the system preserves database consistency and makes the exception visible for manual resolution.

## Main Design Principles

The payment system follows a few simple rules:

1. **Never trust client-side pricing.**
2. **Never treat a success-page redirect as proof of payment.**
3. **Verify webhook signatures.**
4. **Make order creation idempotent.**
5. **Use database constraints in addition to application checks.**
6. **Keep inventory changes and order creation consistent.**
7. **Handle exceptions explicitly instead of hiding them.**