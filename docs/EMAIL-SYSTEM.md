# Email System

Email is a core part of the application rather than a simple notification layer.

The system handles transactional messages, marketing campaigns, delivery events, and two-way customer conversations.

The application originally used Mailjet and later migrated to Brevo.

## Email Architecture

```text id="p8s5h1"
                         +------------------+
                         |     Express      |
                         +--------+---------+
                                  |
                +-----------------+----------------+
                |                                  |
                v                                  v
       Transactional Email                  Campaign Sender
                |                                  |
                +----------------+-----------------+
                                 |
                                 v
                              Brevo
                                 |
              +------------------+------------------+
              |                  |                  |
              v                  v                  v
          Delivery            Inbound            Customer
           Events             Email              Inbox
              |                  |
              v                  v
        Campaign Stats       Ticket Thread
```

## Transactional Email

The application sends email for events such as:

- Appointment confirmation
- Appointment reschedule
- Appointment cancellation
- Appointment completion
- Member verification
- Password recovery
- Rewards
- Referrals
- Order confirmation
- Refunds
- Staff notifications
- Account invitations
- Password changes

The scheduler also sends appointment reminders and reward-related notifications.

## Development and Testing

A real-mail switch separates development/testing behavior from production email sending.

Tests use a dedicated database and do not send real emails.

This is important because integration tests can create large numbers of accounts and workflows without accidentally emailing real customers.

## Customer Messaging

Customer messaging is organized around tickets.

A customer starts a conversation through the contact system.

The application creates a ticket and generates a signed Reply-To address associated with that ticket.

```text id="t2x4c8"
Customer
   |
   | Contact form
   v
Ticket
   |
   +--> Confirmation email
   |
   +--> Signed Reply-To address
             |
             v
        Customer replies
             |
             v
       Brevo inbound parser
             |
             v
       Verify signature
             |
             v
       Find ticket
             |
             v
       Append message
```

This allows a customer to continue the conversation directly from their normal email client.

## Signed Reply Addresses

The Reply-To address contains signed information identifying the intended ticket.

The signature is verified using HMAC and compared in constant time.

The application does not simply trust an identifier supplied by an incoming email.

This is important because inbound email is an external input and could otherwise be manipulated to target another customer's conversation.

## Inbound Email

Inbound messages pass through several checks before they are attached to a ticket.

The system checks things such as:

- Provider authentication
- Reply-address signature
- Ticket validity
- Duplicate message identifiers
- Auto-reply indicators
- Bounce indicators
- Spam signals
- Sender mismatch
- Attachment limits

Inbound processing records an outcome so failures can be investigated.

Possible outcomes include attached, duplicate, unmatched, invalid signature, missing ticket, spam, auto-reply, and error.

## Idempotent Inbound Processing

Email providers can retry webhook delivery.

The system therefore creates a stable deduplication key using available message identifiers and fallback information.

The ticket update is conditional on that key not already existing.

Conceptually:

```text id="k5j7m2"
Inbound email
     |
     v
Create dedup key
     |
     v
Already processed?
   /       \
 yes        no
 |           |
skip       append
             |
             v
          record key
```

If the same provider event arrives again, it becomes a no-op rather than creating another message in the conversation.

## Email Threading

The system uses standard email threading headers so that customer replies stay grouped into the same conversation in the customer's email client.

Subjects are also normalized and cleaned before being placed into email headers.

User-controlled newlines are removed to prevent email header injection.

## Attachments

Customer attachments are not blindly served as executable content.

Safe raster image types can be displayed inline.

Other file types are forced to download.

The download path also uses response headers and cleaned filenames to reduce the risk of browser content execution.

## Marketing Campaigns

Staff can create rich-text campaigns and select an audience.

Before sending, the server builds the recipient list.

The list is:

- Deduplicated
- Filtered for marketing opt-outs
- Prepared from the appropriate customer groups

Campaign messages are sent in throttled batches rather than sending the entire audience at once.

## Delivery Analytics

Brevo event webhooks update campaign recipient state.

The system tracks events including:

- Delivered
- Opened
- Hard bounce
- Soft bounce
- Spam complaint
- Unsubscribe

Provider message IDs are used to associate events with the correct campaign recipient.

Hard bounces and spam complaints can also result in automatic suppression.

## Unsubscribe

Marketing unsubscribe links use signed tokens.

The application stores opt-out state centrally and checks it before sending marketing campaigns.

Marketing opt-out does not prevent transactional messages such as appointment or order notifications.

## Email Security

User-controlled values are escaped before being inserted into HTML email templates.

Subjects are cleaned before being used in email headers.

Inbound webhook authentication happens before the request body is processed.

Provider secrets and signing values are stored through environment configuration rather than being embedded in the application.

## Design Takeaway

The email system ended up becoming a small communication platform inside the application.

The interesting part was not sending an email. It was making email behave reliably when providers retry requests, customers reply from outside the application, attachments are involved, users unsubscribe, and multiple messages need to remain part of the same conversation.