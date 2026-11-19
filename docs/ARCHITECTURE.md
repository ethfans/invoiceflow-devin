# Architecture Notes

## Status and scope

This document describes the proposed MVP architecture. The application runtime, external integrations, and automated agent workflows are not implemented yet.

## System boundaries

- **Web application:** authenticated dashboard and invoice/customer management.
- **Relational database:** PostgreSQL accessed through Prisma.
- **Payment adapter:** Stripe Checkout/payment intents and signed webhook intake.
- **Email adapter:** outbound invoice/reminder email and inbound support messages.
- **Background processing:** retryable webhook, reminder, and email jobs.
- **Engineering automation:** Devin investigates alerts and proposes code changes through pull requests.

## Stripe webhook lifecycle

1. Read the raw request body and verify the Stripe signature before parsing.
2. Insert the Stripe event ID into `StripeWebhookEvent`. Its unique constraint is the idempotency boundary.
3. If the event already exists and is processed, acknowledge it without repeating side effects.
4. Mark the event as processing and apply payment/invoice updates in a database transaction.
5. Mark the event processed. On failure, persist a sanitized error and increment the attempt count for retry.
6. Keep external side effects (email, refunds) out of the database transaction; enqueue them after a successful state transition.

## Invoice and payment invariants

- Amounts use fixed-precision decimal columns; never use floating-point arithmetic for money.
- Invoice totals are calculated server-side from validated line items.
- Payment success is derived from verified provider events, not browser redirects.
- A payment can be recorded only against an invoice belonging to the authenticated user.
- Reconcile invoice status from successful payment totals; do not trust client-supplied status values.

## Devin maintenance workflow

**Signals:** Stripe delivery failures, Sentry issues, failed GitHub Actions, and operational alerts.

**Agent loop:** collect sanitized evidence → inspect relevant code → reproduce in a test → implement a narrow patch → add regression coverage → open a pull request with diagnosis and test output.

**Human approval required:** merging, deploying, database migrations, refunds, financial adjustments, and customer-facing email replies. The agent must not receive production secrets or unrestricted access to customer data.

## Security and privacy

- Verify Stripe signatures using server-side secrets.
- Keep credentials in environment/secret storage; never commit them.
- Redact email addresses, payment identifiers, and message content from logs where possible.
- Minimize retention of inbound email content; store sanitized excerpts only when necessary.
- Enforce tenant ownership checks on every customer, invoice, payment, and email query.
- Apply rate limits to authentication, invoice creation, and webhook endpoints.
