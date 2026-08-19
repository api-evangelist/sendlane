---
name: sendlane-pull-campaign-performance
description: >-
  Pull email campaign and SMS performance and revenue metrics out of Sendlane into
  a warehouse or dashboard, paginating correctly and staying inside the account
  rate limit.
api: sendlane:sendlane-api
base_url: https://api.sendlane.com/v2
operations:
  - get-campaigns
  - get-campaigns-campaignId
  - get-campaigns-report
  - get-sms-messages
  - get-sms-message
  - get-sms-report
  - get-contacts-history
generated: '2026-08-13'
method: generated
source: openapi/sendlane-openapi.yml
---

# Pull campaign and SMS performance from Sendlane

Sendlane's pitch is revenue attribution, and the reporting endpoints are where that
number lives. This is a pull-only flow: there is no reporting webhook and no export
API, so a warehouse sync is a scheduled poll.

## Before you start

- Auth: `Authorization: Bearer <SENDLANE_API_TOKEN>`, `Accept: application/json`.
- **Rate limit 240 requests/minute per account, shared across every integration.** A
  naive "list campaigns, then fetch a report per campaign" loop is N+1 and will burn
  the whole account's budget. Batch by time window and cache.
- No `RateLimit-*` headers come back — you cannot read remaining quota, so throttle
  client-side and back off on 429.

## Steps

### 1. List campaigns

```
GET /campaigns?limit=100&page=1
```
`get-campaigns`. Paginated. Read `meta.total` and `meta.last_page` to size the job,
then walk `links.next` until it is null.

### 2. Pull the report per campaign

```
GET /campaigns/{campaignId}/report
```
`get-campaigns-report`. `CampaignReport.v2` returns:

`sent`, `opens`, `unique_opens`, `clicks`, `unique_clicks`, `bounced`, `suppressed`,
`unsubscribed`, `spam`, **`revenue`**, and `automation`.

Both raw and unique open/click counts are returned — decide which one your dashboard
means and be consistent. `revenue` only populates if e-commerce events are being
tracked; see `skills/sendlane-track-ecommerce-events.md`.

### 3. Do the same for SMS

```
GET /sms-messages                        get-sms-messages
GET /sms-messages/{messageId}            get-sms-message
GET /sms-messages/{messageId}/report     get-sms-report
```
`SmsMessage.v2` carries the parent `sms_campaign` and `automation`, so you can group
one-off sends and automation-driven sends separately.

### 4. Per-contact detail when you need it

```
GET /contacts/{contactId}/history
```
`get-contacts-history`. Returns a `ContactHistory.v2` timeline referencing
`campaign_id`, `list_id` and `event_id`. Expensive at scale — one call per contact —
so use it for investigation, not for bulk export.

## Incremental syncs

Use the `from` and `to` query parameters where the operation accepts them. RFC 3339
datetimes, and **`from` is required whenever `to` is present**. Percent-encode `+` as
`%2b`:

```
?from=2026-08-01T00:00:00Z&to=2026-08-13T00:00:00Z
```

Store the high-water mark yourself. Nothing in the API gives you a change feed.

## Pagination contract

Every list response is `{ data, links, meta }`:

- `links.first` / `last` / `prev` / `next`
- `meta.current_page` / `from` / `last_page` / `path` / `per_page` / `to` / `total`

The provider's stated rule: keep calling until `links.next` is null. Do not compute
page counts from `meta.total` and fire them in parallel — that is the fastest way to
a 429.

## Errors

`{ "message": string, "errors": object|array }`. 429 on rate-limit exhaustion; back
off exponentially. 401 returns `text/html`, not JSON.

See `conventions/sendlane-conventions.yml` and `rate-limits/sendlane-rate-limits.yml`.
