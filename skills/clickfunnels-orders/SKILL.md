---
name: clickfunnels-orders
description: >
  Manage an existing ClickFunnels order programmatically - move a subscription's trial end
  or Payments AI renewal date, cancel a Payments AI or Stripe subscription, refund a charge
  in full, or abandon an unpaid invoice to stop its automatic payment retries. Use this skill
  when you need to extend or shorten a trial, shift a billing date, cancel a subscription,
  stop a rebill, issue a refund, halt retry attempts, or read orders, invoices, and
  transactions through the API.
version: "1.3"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/subscription-changes/skill.md
    description: Change the variant/price on a subscription line item instead of cancelling and re-buying.
---

# ClickFunnels Orders Skill

Drive the schedule and lifecycle of an existing order: move a subscription's trial end or
Payments AI renewal date, cancel a Payments AI or Stripe subscription, refund a charge, or
abandon an unpaid invoice to stop its payment retries. These actions use ClickFunnels' own
processor-backed paths rather than treating local status fields as processor commands. Unlike
these action paths, a plain `PATCH` of `service_status` is silently ignored on a processor-managed
order.

Each action has its own subject, and picking the right one matters:

| You want to | Act on | Why |
|-------------|--------|-----|
| Stop future renewals | the **order** | The subscription is the order. |
| Reverse money already taken | a **transaction** | A charge is a transaction; an order accumulates one per renewal. |
| Stop retry attempts on a failed payment | an **invoice** | Retries are scheduled per invoice. |

## When to use this skill

- Extend or shorten a subscription trial, or move the next Payments AI billing date.
- Cancel a Payments AI or Stripe subscription (stop future renewals), either immediately or at
  the end of the paid period.
- Refund a specific charge on an order, in full.
- Abandon an unpaid or past-due invoice so the processor stops retrying the payment (for
  example an insufficient-funds decline where you want to grant a few days' grace instead of
  canceling or refunding).
- Read an order and its invoices/transactions to decide which action applies.

For moving a subscription onto a different plan without cancelling, use the
[subscription-changes](https://accounts.myclickfunnels.com/.well-known/subscription-changes/skill.md) skill instead.

## Authentication

All requests use the shared OAuth bearer token and act on the workspace the token belongs to:

```
Authorization: Bearer YOUR_TOKEN
```

## Endpoint reference

| Action | Method | Path |
|--------|--------|------|
| List orders | `GET` | `/api/v2/workspaces/:workspace_id/orders` |
| Show an order | `GET` | `/api/v2/orders/:id` |
| List invoices | `GET` | `/api/v2/orders/:order_id/invoices` |
| List transactions (charges) | `GET` | `/api/v2/orders/:order_id/transactions` |
| Move a trial end or renewal date | `PATCH` | `/api/v2/orders/:id` |
| Cancel a Payments AI/Stripe subscription | `POST` | `/api/v2/orders/:id/cancel` |
| Refund a charge in full | `POST` | `/api/v2/orders/transactions/:id/refund` |
| Abandon an unpaid invoice (stop retries) | `POST` | `/api/v2/orders/invoices/:id/abandon` |

`:id` accepts either the integer database id or the obfuscated `public_id` string.

## Move a subscription's dates (extend a trial, shift a billing date)

`PATCH /api/v2/orders/:id` accepts `trial_end_at` on a Payments AI or Stripe subscription, and
`renewal_at` on a Payments AI subscription.

| Field | What it moves | Where it works |
|-------|---------------|----------------|
| `trial_end_at` | Trial end and first charge | Payments AI and Stripe when `in_trial` is true; Payments AI also accepts it as the equal companion to a legal renewal move while the stored trial end is still in the future |
| `renewal_at` | Next billing-cycle start | Payments AI in `pending`, `active`, or `paused` state |

Both take an absolute RFC 3339 timestamp with a date, time, and UTC offset, within the next two
years. Read the current value, calculate the new timestamp, and send it. A later `trial_end_at`
extends the trial; an earlier future value shortens it. Send a timestamp a few seconds ahead to
end a trial now.

### What happens to the invoice

Moving `trial_end_at` also moves the first charge. Moving `renewal_at` changes the next billing
attempt without changing the amount or prorating it. An invoice that has already been issued is
never re-dated, unpaid and past-due ones included; use the invoice endpoints for those. So do not
verify a move by reading an invoice - the last one keeps its original period. In the array returned
by `GET /api/v2/orders/:order_id/invoices`, read each invoice's
`line_items[].period_start_at` / `period_end_at`; the invoice itself has no period field. Read the
order's `trial_end_at`, `renewal_at` and `next_charge_at` instead.

Two consequences worth planning for, neither of them visible on an invoice:

- Pushing `renewal_at` later does not extend the paid period and nothing is prorated or credited.
  The contact simply keeps service until the new date, so the gap between the last billed period
  and the new renewal is unbilled service.
- A renewal that lands after the trial ends means the trial finishes with no charge due:
  the processor ends the trial on that date, `in_trial` turns false after its sync lands, service
  continues, and the first charge falls on the renewal date. Useful as a grace period, but track
  the intent on your side.

While `trial_end_at` is still in the future, `renewal_at` may not be set earlier than it: the
contact would be billed while still flagged as in a trial. Send both fields to move them together.
When both fields are sent, they must use the same timestamp. To renew after the trial ends, update
`trial_end_at` first, re-read the order until it shows the effective new trial end, then update
`renewal_at` in a second request.

```bash
# Extend a trial by two weeks after calculating the new timestamp.
curl -X PATCH \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"order": {"trial_end_at": "<the order trial_end_at plus 14 days, ISO8601>"}}' \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/6396323"
```

| Status | Meaning |
|--------|---------|
| `200` | The payment processor confirmed the new date. |
| `202` | The change is accepted but ClickFunnels could not confirm the exact value requested; the processor may have returned no value or a different effective date. |
| `422` | The change was rejected or not applied. Correct the request and retry. |

For an actual schedule change, on `200` as well as `202`, the order in the response will show the
PREVIOUS date. These dates are applied at the payment processor rather than stored, and the local
values settle when the processor's sync lands. Keep the pre-PATCH value and poll
`GET /api/v2/orders/:id` with backoff. After `200`, wait until the field equals the timestamp you
sent, comparing parsed instants at whole-second precision rather than timestamp strings. After
`202`, wait until it differs from the pre-PATCH value, then use the returned value as the effective
schedule. Choose your own timeout because there is no fixed settlement deadline. If it expires,
the outcome is unknown, not failed; do not retry the `PATCH` blindly.

These two fields need the same permission as the in-app "Change trial end date" and "Update
renewal" screens. An unauthorized change returns `404`. A `null` or unchanged value requests no
schedule change, so a GET/PATCH round trip remains safe.

## Cancel a subscription

`POST /api/v2/orders/:id/cancel` cancels a Payments AI or Stripe SUBSCRIPTION order. One-time
orders and external subscriptions return `422`. The request body is optional - a bare POST
cancels with defaults. All fields nest under `order`:

| Field | Default | Notes |
|-------|---------|-------|
| `effective_date` | `next_renewal` | `next_renewal` cancels at the end of the paid period (access retained until `paid_until`). Any other value cancels immediately. |
| `cancel_reason` | `other` | Processor-specific set (see below). An unaccepted value returns `422`. |
| `cancel_description` | `""` | Free-text note stored on the order. |
| `cancelation_proration` | `false` | Whether to prorate on an immediate cancel. |
| `canceled_by` | `API` | Attribution persisted on the order and recorded on its timeline. |
| `voidable_invoices` | none | Array of invoice ids to void as part of the cancel. |

`cancel_reason` and `cancel_description` mirror the `cancel_reason` / `cancel_description`
fields you read back on the order, so you can round-trip them.

Accepted `cancel_reason` values depend on the order's processor:

- Stripe orders: `customer_service`, `low_quality`, `missing_features`, `other`,
  `switched_service`, `too_complex`, `too_expensive`, `unused`.
- Payments AI orders: `did-not-use`, `did-not-want`, `missing-features`, `too-expensive`,
  `bugs-or-problems`, `other`.

Cancel at end of the paid period:

```http
POST /api/v2/orders/jxXYbj/cancel
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json

{ "order": { "effective_date": "next_renewal", "cancel_reason": "too_expensive" } }
```

Response `200 OK` means the processor accepted the cancellation and returns the latest local
order snapshot. Processor-populated fields can settle in a later webhook. For example, an
immediate Payments AI cancel can first return `service_status: "canceled"` with a null
`canceled_at`, then return `service_status: "churned"` with its final cancellation fields on a
later `GET`. Poll `GET /api/v2/orders/:id` when those fields are not yet populated. The supplied
`canceled_by` value is persisted before that sync and remains stable.

```json
{
  "id": 33478,
  "public_id": "jxXYbj",
  "order_number": "#1305",
  "order_type": "subscription-order",
  "service_status": "canceled",
  "billing_status": "paid",
  "canceled_at": "2026-07-20T09:44:02.000Z",
  "canceled_by": "API",
  "cancel_reason": "too_expensive",
  "cancel_description": "",
  "churned_at": "2026-08-20T09:31:15.000Z",
  "paid_until": "2026-08-20T09:31:15.000Z",
  "payment_processor": "stripe"
}
```

## Refund a charge

`POST /api/v2/orders/transactions/:id/refund` refunds one charge IN FULL. Partial refunds are
not supported through this endpoint, so there is no amount to send.

Refunds address a transaction, not an order or an invoice, because a transaction IS the charge:
a subscription order accumulates one charge per renewal, and an invoice can carry more than one.
So first list the order's charges and pick the one you mean:

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/JBYzEj/transactions"
```

A refundable charge has `external_type` of `sale` or `capture`, `result: "approved"`, and
`status` of `completed` or `partially-refunded`. Then refund it - body fields nest under
`orders_transaction`, and the body is optional:

| Field | Default | Notes |
|-------|---------|-------|
| `reason` | `requested_by_customer` | Stripe orders require `duplicate`, `fraudulent`, or `requested_by_customer` (an unaccepted value returns `422`). Payments AI treats it as free text. |
| `partial` | `false` | Must be false/omitted; `true` returns `422` (partial refunds are unsupported). |

```http
POST /api/v2/orders/transactions/NpboZw/refund
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json

{ "orders_transaction": { "reason": "requested_by_customer" } }
```

Two success shapes, both returning the serialized transaction:

- `200 OK` - the refund is reflected immediately (Payments AI records it in-request). The charge
  shows `status: "refunded"`.
- `202 Accepted` - the refund was accepted but settles asynchronously (Stripe syncs local state
  via a later webhook). The returned charge may still show the pre-refund `status`; poll
  `GET /api/v2/orders/transactions/:id` for the settled state.

```json
{
  "id": 41277,
  "public_id": "NpboZw",
  "order_id": 33477,
  "status": "refunded",
  "external_type": "sale",
  "amount": "100.00",
  "currency": "USD",
  "reason": "requested_by_customer",
  "result": "approved"
}
```

A charge that cannot be refunded (already fully refunded, or not a charge at all) returns
`404` rather than an error about its state, the same way this API hides any record you cannot
act on.

## Abandon an unpaid invoice (stop payment retries)

`POST /api/v2/orders/invoices/:id/abandon` abandons an unpaid or past-due invoice and STOPS the
processor's automatic payment retries for it. This is the way to halt retries: writing
`status: "abandoned"` through `PUT /api/v2/orders/invoices/:id` is only available for external
orders and returns `404` for Payments AI and Stripe orders.

Reach for it when a payment fails for a reason that is not a bad card - an insufficient-funds
decline, say - and you want to stop the retry cycle for a few days rather than cancel the
subscription or refund a charge. Only invoices in `unpaid`, `draft`, or `past-due` status can be
abandoned; anything else returns `404`. No request body is needed.

```http
POST /api/v2/orders/invoices/gjDMvQ/abandon
Authorization: Bearer YOUR_TOKEN
```

`200 OK` returns the serialized invoice, with retries stopped:

```json
{
  "id": 88213,
  "public_id": "gjDMvQ",
  "order_id": 33477,
  "status": "abandoned",
  "invoice_type": "renewal",
  "total_amount": "100.00",
  "currency": "USD"
}
```

## Worked example - cancel then confirm

```bash
# 1. Cancel at period end
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"order":{"effective_date":"next_renewal"}}' \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/jxXYbj/cancel"

# 2. Confirm the scheduled churn
curl -X GET \
  -H "Authorization: Bearer YOUR_TOKEN" \
  "https://YOUR_WORKSPACE.myclickfunnels.com/api/v2/orders/jxXYbj"
# -> service_status "canceled", churned_at == paid_until
```

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `422 Only subscription orders can be canceled` | Cancelled a one-time order. | Only subscription orders can be cancelled. If you meant to reverse a charge, refund the transaction instead. |
| `422 trial_end_at can only be adjusted while the order is in a trial` | The order already converted or never had a trial. | Check `in_trial` and `trial_end_at`. A pending Payments AI order with a future trial end can send equal `trial_end_at` and `renewal_at` values together. |
| `422 renewal_at cannot be adjusted on a Stripe order` | Stripe has no renewal date to move. | Move `trial_end_at` while the order is still in a trial, or cancel and re-subscribe. |
| `422 renewal_at cannot be adjusted while the order is <status>` | The subscription is canceled, churned, or completed. | Only `pending`, `active`, and `paused` orders have a renewal date to move. |
| `422 ... must be in the future` / `must be a valid ISO8601 timestamp carrying a date, a time of day and a UTC offset (RFC 3339)` | The value is not a future RFC 3339 timestamp within the next two years. | Send a value such as `2026-09-11T12:00:00Z`. |
| `422 renewal_at ... is before the trial ends` | Would bill a contact who is still flagged as in a trial. | Send `trial_end_at` in the same request to end the trial at that date too. |
| `422 renewal_at and trial_end_at must be the same timestamp when sent together` | Combined changes must share a timestamp. | Send the same timestamp for both, or move `trial_end_at` first, re-read until the effective new trial end appears, then move `renewal_at`. |
| `422 External orders have no processor-managed schedule` | Sent a schedule field on an external order. | External orders have no processor to command. Record the next billing date with `next_charge_at`. |
| `422 The payment processor rejected the schedule change: ...` | The processor refused the write; its own message follows the colon. | The processor, not your request, refused it. A subscription the processor has not created yet is the usual cause - check the order's `service_status`. |
| `422 The payment processor did not apply the change` | The requested date was not applied. | Re-read `GET /api/v2/orders/:id`, then correct and retry. |
| `404 Not found` on a schedule field | The token may update the order but not change its schedule. | These two fields need the same permission as the in-app trial/renewal screens. Use a token whose user can manage orders in that workspace. |
| `422 External subscription orders cannot be canceled ...` | Tried to cancel an externally managed subscription. | Cancel it in the external system, then update the external order through the supported external-order API. |
| `422 Invalid reason '...'. Valid values: ...` | `cancel_reason`/`reason` not in the processor's accepted set. | Send one of the listed values (they differ by processor). |
| `422 Partial refunds are not supported ...` | Sent `partial: true` or an `amount`. | This endpoint only does a full refund of one charge. An `amount` is rejected rather than silently widened to the full charge. |
| `404 Not found` on a refund | The transaction id is unknown, outside the token's workspace, or not refundable (already refunded, or not a charge). | List `GET /api/v2/orders/:order_id/transactions` and pick a charge with `external_type` `sale`/`capture`, `result: "approved"`, `status: "completed"`. |
| `404 Not found` on an abandon | The invoice id is unknown, outside the token's workspace, or not in `unpaid`/`draft`/`past-due` status. | Check the invoice `status` with `GET /api/v2/orders/invoices/:id`; only those three can be abandoned. |
| `404 No such external order` | Tried to `PUT` an invoice `status` on a Payments AI/Stripe order. | Use the action endpoint for what you meant: abandon to stop retries, or refund the charge. |
| `404 Not found` | Order id is unknown or not in the token's workspace. | Verify the id and that the token owns the order's workspace. |
| `401 API key missing or invalid` | Missing/invalid bearer token. | Send a valid `Authorization: Bearer` token. |
