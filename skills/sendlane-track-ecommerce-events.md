---
name: sendlane-track-ecommerce-events
description: >-
  Send e-commerce behaviour and transaction events into Sendlane so automations
  can trigger on browse abandonment, cart abandonment and purchases, and revenue
  attribution works.
api: sendlane:sendlane-api
base_url: https://api.sendlane.com/v2
operations:
  - list-custom-integrations
  - create-custom-integration
  - create-custom-event
  - index-custom-events
  - post-tracking-product-viewed
  - post-tracking-added-to-cart
  - post-tracking-checkout-started
  - post-tracking-order-placed
  - post-tracking-back-in-stock
  - post-tracking-event
generated: '2026-08-13'
method: generated
source: openapi/sendlane-openapi.yml
---

# Track e-commerce events into Sendlane

Sendlane's automations fire on customer behaviour. The behaviour has to be pushed in:
**the direction is inbound.** Sendlane calls these endpoints "Custom Integration
Webhooks", but Sendlane never calls you — there are no outbound webhooks anywhere in
the v2 API. If you need to know something that happened inside Sendlane, you poll.

## Before you start

- Auth: `Authorization: Bearer <SENDLANE_API_TOKEN>`, `Accept: application/json`.
- Every `/tracking/*` payload also requires a `token` field — the **integration**
  token, which is a different credential from the API bearer token. Get it from the
  dashboard: Manage next to your Sendlane Integration, then Copy code.
- Rate limit 240/min per account. High-traffic storefronts will hit this; batch and
  queue on your side.
- No idempotency key. A retried `order-placed` can double-count revenue.

## Steps

### 1. Set up the integration

```
GET  /integrations/custom          list-custom-integrations
POST /integrations/custom          create-custom-integration
```

### 2. Register any custom event types you need

```
GET  /integrations/custom/{integrationToken}/events   index-custom-events
POST /integrations/custom/{integrationToken}/events   create-custom-event
```
Only needed for events outside the eight typed ones below.

### 3. Emit the typed events

| Trigger | Operation | Path | Required fields |
|---|---|---|---|
| Product page view | `post-tracking-product-viewed` | `/tracking/product-viewed` | `token` |
| Add to cart | `post-tracking-added-to-cart` | `/tracking/added-to-cart` | `token` |
| Checkout started | `post-tracking-checkout-started` | `/tracking/checkout-started` | `token`, `checkout_id` |
| Order placed | `post-tracking-order-placed` | `/tracking/order-placed` | `token`, `order_id` |
| Back in stock request | `post-tracking-back-in-stock` | `/tracking/back-in-stock` | `token` |
| Category browsed | `post-tracking-category-added` | `/tracking/category-added` | `token` |
| New customer | `post-tracking-customer-added` | `/tracking/customer-added` | `token` |
| Product catalog add | `post-tracking-product-added` | `/tracking/product-added` | `token` |
| Anything else | `post-tracking-event` | `/tracking/event` | `token` |

`OrderPlacedRequest.v2` is the payload that drives revenue attribution — send
`subtotal`, `total`, `currency`, `line_items`, `discounts` and `tax_lines`, plus
`email` or `phone` so the order binds to a contact. Without an identifier the event
cannot be attributed.

`CheckoutStartedRequest.v2` carries `checkout_url`, which is what an abandoned-cart
automation links the customer back to. Omit it and the recovery email has nowhere to
send them.

### 4. Backfilling history

Include `time` or `date_created` on the payload, otherwise the event is stamped at
receipt and your historical import lands as if it all happened today. Set
`initial_sync` on `order-placed` for a bulk import.

## Browser alternative

The same stream can be produced client-side instead of server-side:

```html
<script src="https://sendlane.com/scripts/pusher.js" async data-token="INTEGRATION_TOKEN"></script>
```

Exactly one instance per site — multiple instances break tracking. The script is
served unpinned, so you cannot control which build you get.

## Errors

These are the only endpoints that declare error responses: **400** (invalid
parameters, fields or filters) and **422** (validation failed, with `errors` keyed on
the offending attribute). A 200 means accepted, not processed.

See `asyncapi/sendlane-webhooks.yml`.
