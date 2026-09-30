---
name: clickfunnels-external-purchases
description: >
  Turn external provider purchases into ClickFunnels orders: create incoming endpoints,
  map external plan or SKU ids to ClickFunnels prices, and diagnose the retained events.
  Use this skill when an agent needs to configure, diagnose, or maintain purchases that
  originate in an external commerce provider.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index.
  - path: https://accounts.myclickfunnels.com/.well-known/products/skill.md
    description: Create the ClickFunnels products, variants, and prices used by mappings.
  - path: https://accounts.myclickfunnels.com/.well-known/orders/skill.md
    description: Manage orders created from accepted external purchase events.
---

# ClickFunnels External Purchases Skill

External purchase endpoints turn provider webhooks into ClickFunnels orders. Each endpoint
belongs to one workspace and one provider. Product mappings tell ClickFunnels which local
variant and price to invoice when a provider sends a plan id or SKU.

## When to use this skill

- Create or update an endpoint for a provider ClickFunnels accepts.
- Set the signing secret of an endpoint whose provider issues its own.
- Map an external plan id or SKU to a ClickFunnels variant and price.
- Inspect retained events and understand why an order was or was not created.

Not every provider is available to every workspace. Creating an endpoint for one a workspace
has not been granted returns 403, and the `provider` values a workspace can actually use are
the ones its own app settings offer.

## Authentication

Use a ClickFunnels V2 API OAuth token:

```http
Authorization: Bearer <access_token>
```

The token must have access to the target workspace. Member endpoints return 404 when the
record is outside that workspace, even if the id exists elsewhere.

ClickFunnels never exposes a workspace's Whop OAuth access or refresh tokens, and its API
does not proxy Whop catalog reads. To obtain a Whop `company_id` or plan id, call Whop
directly with a Whop credential supplied to you for that purpose. Send only those non-secret
ids to ClickFunnels. Never ask the user to extract ClickFunnels' stored OAuth token.

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| List endpoints | GET | `/api/v2/workspaces/:workspace_id/webhooks/incoming/external_purchase_endpoints` |
| Create endpoint | POST | `/api/v2/workspaces/:workspace_id/webhooks/incoming/external_purchase_endpoints` |
| Fetch / update / delete endpoint | GET / PATCH / DELETE | `/api/v2/webhooks/incoming/external_purchase_endpoints/:id` |
| List / create mappings | GET / POST | `/api/v2/webhooks/incoming/external_purchase_endpoints/:id/product_mappings` |
| Fetch / update / delete mapping | GET / PATCH / DELETE | `/api/v2/webhooks/incoming/external_purchase_product_mappings/:id` |
| List endpoint events | GET | `/api/v2/webhooks/incoming/external_purchase_endpoints/:id/events` |
| Fetch event | GET | `/api/v2/webhooks/incoming/external_purchase_events/:id` |

## Create an endpoint

Choose `provider` and `live_mode` when creating the endpoint. They cannot be changed through
the update API, because changing either would point an existing signing secret at a different
provider connection.

```http
POST /api/v2/workspaces/3/webhooks/incoming/external_purchase_endpoints
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "webhooks_incoming_external_purchases_endpoint": {
    "name": "Whop memberships",
    "provider": "whop",
    "live_mode": false,
    "suppress_order_system_emails": true
  }
}
```

```json
{
  "id": 291,
  "public_id": "RvpWnj",
  "workspace_id": 3,
  "name": "Whop memberships",
  "provider": "whop",
  "live_mode": false,
  "suppress_order_system_emails": true,
  "created_at": "2026-08-26T13:43:02.918Z",
  "updated_at": "2026-08-26T13:43:02.918Z",
  "url": "https://acme.myclickfunnels.com/webhooks/incoming/external_purchases_webhooks/RvpWnj",
  "webhook_secret_last_four": null,
  "provider_metadata": {
    "whop": {}
  }
}
```

`provider_metadata` is keyed by provider, so the single key both names the provider and
types its value. No provider has fields yet, so each returns an empty object under its key.

Where ClickFunnels generates the signing secret, create returns it as `webhook_secret` in
that response only, so store it immediately. Later reads return only
`webhook_secret_last_four`. Where the provider issues its own signing secret, the endpoint is
created without one and you set it afterwards; see the next section.

Filter the list by `id`, `provider` or `live_mode`, for example
`?filter[provider]=whop&filter[live_mode]=false`.

## Set a Whop endpoint's signing secret

Whop reveals a webhook's signing secret only to whoever creates the webhook, and it grants
webhook management only to a company API key, so the account owner creates the webhook in
the Whop dashboard. Point it at the endpoint's `url`, copy the `ws_` signing secret Whop
shows once at creation, and set it on the endpoint:

Subscribe the webhook to all of `payment.succeeded`, `payment.failed`,
`membership.activated`, `membership.deactivated`,
`membership.cancel_at_period_end_changed`, `refund.created`, `refund.updated`,
`dispute.created`, and `dispute.updated`. A webhook subscribed to fewer of them still
produces orders, which makes the omission easy to miss: the cancellations, refunds and
disputes it left out simply never arrive. Any other Whop event type is dropped, and so is
a `payment.succeeded` whose plan is not mapped yet: neither leaves an event to read, so map
the plan before the first sale you want recorded.

```http
PATCH /api/v2/webhooks/incoming/external_purchase_endpoints/RvpWnj
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "webhooks_incoming_external_purchases_endpoint": {
    "secret": "ws_demo"
  }
}
```

The secret must start with `ws_` and a blank value is ignored, so an unrelated update can
never clear a working one. Reads return only `webhook_secret_last_four`, which is null until
a secret is set. Because
each webhook lives in its owner's Whop dashboard, a workspace can run any number of Whop
endpoints, each fed by a different Whop account or company. Deleting the endpoint leaves
the webhook untouched in Whop; remove it from the Whop dashboard as well.

## Create a product mapping

First create or select a ClickFunnels product, variant, and price. See the Products skill.
The variant and price must belong to the same product and the endpoint's workspace. Numeric
or public ClickFunnels ids are accepted.

For Whop, `external_product_sku` is the plan `id` returned by Whop's API when you query it
with your own Whop credential:

```http
POST /api/v2/webhooks/incoming/external_purchase_endpoints/RvpWnj/product_mappings
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "external_purchase_product_mapping": {
    "external_product_sku": "plan_creator_monthly",
    "products_variant_id": "PjvAAJ",
    "products_price_id": "PjvAAJ"
  }
}
```

```json
{
  "id": 35,
  "public_id": "WJNwwj",
  "external_purchase_endpoint_id": 291,
  "external_product_sku": "plan_creator_monthly",
  "created_at": "2026-08-26T13:43:03.582Z",
  "updated_at": "2026-08-26T13:43:03.582Z",
  "clickfunnels_product": {
    "id": 35,
    "public_id": "PjvAAJ",
    "name": "Creator Community"
  },
  "clickfunnels_variant": {
    "id": 35,
    "public_id": "PjvAAJ",
    "name": "Creator Community"
  },
  "clickfunnels_price": {
    "id": 35,
    "public_id": "PjvAAJ",
    "name": "Monthly",
    "currency": "usd",
    "payment_type": "one_time",
    "amount": "99.00"
  }
}
```

The ClickFunnels price controls what appears on the ClickFunnels invoice, so match its amount
and recurrence to the external plan. One external SKU can appear only once per endpoint.

## Read and diagnose events

Whop records the verified lifecycle events this listener acts on, including the ones that
change nothing. An event type it does not act on is logged and not recorded.
For verified `v1` deliveries, ClickFunnels creates purchases from `payment.succeeded`, marks
existing subscriptions past due after `payment.failed`, applies scheduled cancellations,
deactivation, and reactivation from membership events, applies only succeeded refunds to the
correlated invoice, and marks that invoice disputed only when the correlated Whop dispute has
a final `lost` status. Pending refunds, non-lost disputes, and events that cannot be correlated
to an existing order remain visible without changing financial state; they are never guessed
from another endpoint or company.

Because every event is retained, both of those outcomes can be given back. A refund Whop
later reports as `failed` or `canceled` under the same refund id is released, and the invoice
returns to whatever the refunds that still stand say it is, along with the access that refund
revoked on its way in. A dispute Whop resolves as `won` after it was lost is released the same way,
restoring the invoice and lifting the churn that loss caused, unless the membership was itself
canceled or deactivated in the meantime. Whop does not guarantee delivery order, so a report
that a later report already contradicts is retained rather than applied. Every outcome above
is recorded on the order's notes, which are part of the order payload.

A refund or dispute event that correlates to an order but settles nothing, such as a pending
refund or a dispute still awaiting a response, is reported against that order with
`processing_status` `processing_skipped` and a `processing_errors` line saying why. An invoice
moved to a status this listener does not manage, `voided` for instance, is left where it is.
List events under the endpoint and fetch one for its full payload:

```http
GET /api/v2/webhooks/incoming/external_purchase_events/GePAAJ
Authorization: Bearer <access_token>
```

```json
{
  "id": 140,
  "public_id": "GePAAJ",
  "workspace_id": 3,
  "external_purchase_endpoint_id": 291,
  "order_id": null,
  "event_type": "membership.activated",
  "external_event_id": "evt_demo",
  "payload": {
    "id": "evt_demo",
    "api_version": "v1",
    "data": {
      "id": "mem_demo",
      "plan": {
        "id": "plan_creator_monthly"
      },
      "status": "active"
    },
    "type": "membership.activated"
  },
  "processed_at": null,
  "processing_status": "processing_skipped",
  "processing_errors": "No existing order found for Whop membership mem_demo",
  "created_at": "2026-08-26T13:43:03.602Z",
  "updated_at": "2026-08-26T13:43:03.602Z"
}
```

`order_id` is always present and may be null. Use `processing_status`, `processing_errors`
and the raw payload together. The list response omits `payload`, because a page of raw
payloads would be almost entirely payload; fetch a single event to read it. Narrow a busy
endpoint with `?filter[event_type]=payment.succeeded` or
`?filter[processing_status]=processing_skipped`.

## Worked example

1. Create a Whop endpoint in the required live or test mode.
2. Ask the account owner for the plan id they want to sell. Whop shows it on the plan in
   their dashboard, and the app lists it under the endpoint's connected businesses. Once a
   sale has arrived, its retained `payment.succeeded` event carries it at
   `payload.data.plan.id`.
3. Create the webhook in the Whop dashboard pointing at the endpoint's `url`, then set
   the `ws_` secret Whop reveals on the endpoint.
4. Create or select the ClickFunnels product, variant, and price.
5. Create a mapping whose `external_product_sku` is the Whop plan id.
6. After Whop sends an event, list endpoint events and confirm `processing_status` and
   `order_id`. If no mapping existed, create it from `payload.data.plan.id` and process a
   new provider event.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| 401 | Token missing or invalid. | Send a valid Bearer token. |
| 403 `<Provider>` is not enabled for this workspace | This workspace has not been granted that provider. | Use a provider the workspace has, or ask ClickFunnels for access. |
| 404 | Endpoint, event, mapping, variant, or price is outside the authorized workspace. | Re-list records in the intended workspace and use those ids. |
| 422 secret is not a Whop signing secret | The pasted value does not start with `ws_`. | Copy the signing secret Whop shows once when the webhook is created. |
| 422 live_mode cannot be changed | The endpoint already holds a `ws_` signing secret, which verifies one Whop account's deliveries. | Create a separate endpoint for the other mode. `PATCH` a new `secret` to rotate the one you have. |
| 422 duplicate external SKU | That plan id or SKU is already mapped on this endpoint. | Update the existing mapping instead of creating another. |
| 422 variant and price mismatch | The selected variant and price belong to different products. | Pick both ids from the same ClickFunnels product. |
| 422 deleting an endpoint | The endpoint has retained events. | Keep it for event history; delete only endpoints with no events. |
