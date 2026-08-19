---
name: sendlane-onboard-contacts
description: >-
  Add or update contacts on a Sendlane list, record their email consent, and tag
  them — the standard subscriber-onboarding flow for the Sendlane v2 API.
api: sendlane:sendlane-api
base_url: https://api.sendlane.com/v2
operations:
  - get-lists
  - post-lists-listId-contacts
  - get-contacts-search
  - post-contacts-email-consent
  - get-tags
  - post-contacts-contactId-tags
generated: '2026-08-13'
method: generated
source: openapi/sendlane-openapi.yml
---

# Onboard contacts into Sendlane

Bring subscribers into a Sendlane list with consent recorded and tags applied.

## Before you start

- Auth: every request needs `Authorization: Bearer <SENDLANE_API_TOKEN>` and
  `Accept: application/json`. The token is account-scoped with no scopes — it can do
  anything the account can do, including delete.
- Rate limit: 240 requests/minute per ACCOUNT, shared with every other integration on
  that account. No `RateLimit-*` headers come back, so count your own calls.
- **There is no idempotency key and no sandbox.** A retried POST can duplicate work,
  and every call hits production — real sends to real inboxes.

## Steps

### 1. Find the target list

```
GET /lists?limit=100&page=1
```
`get-lists`. Paginated: read `links.next` and keep going until it is null. Each
`List.v2` carries `id`, `list_name` and the list's default from-name, from-email and
reply-to address.

### 2. Add the contacts

```
POST /lists/{listId}/contacts
{ "contacts": [ { ...ListContact.v2 } ] }
```
`post-lists-listId-contacts`. The body takes a `contacts` array (unique items,
minimum 1), so batch rather than looping one contact per request — this is the single
biggest thing you can do to stay inside 240/min.

Returns **202 Accepted**, not 200. The write is queued, so a contact will not
necessarily be readable on the next request. Do not treat 202 as "the contact exists
now".

Each `ListContact.v2` can carry list-scoped `custom_fields` and `sms_consent`. Note
the gate: SMS consent in this payload only works if Sendlane support has enabled the
Contact SMS Consent endpoint on your account — the spec says so in the operation
description itself.

### 3. Resolve a contact id when you only have an email

```
GET /contacts/search
```
`get-contacts-search`. Sends a `ContactSearchRequest.v2` body (`email` and/or `phone`)
— unusually, on a GET. You need the numeric `contactId` for every per-contact
operation below, and ids are bare integers with no type prefix.

### 4. Record email consent

```
POST /contacts/{contactId}/email-consent
{ "email_consent": true, "list_ids": [ ... ] }
```
`post-contacts-email-consent`. `email_consent` is required; `list_ids` scopes the
consent to specific lists.

### 5. Tag the contact

```
GET /tags
POST /contacts/{contactId}/tags
{ "tag_ids": [ 1, 2 ] }
```
`get-tags` then `post-contacts-contactId-tags`. The body requires `tag_ids` — numeric
ids, not tag names, so resolve names to ids from `get-tags` first and cache the map.

## Errors

The envelope is `{ "message": string, "errors": object|array }`. On **422** the
`errors` object is keyed by the request attribute that failed, so read the keys to
find the bad field. On other errors `errors` is a flat array of strings.

Only 400 and 422 are declared in the spec. **401 comes back as `text/html`, not JSON**
— do not assume every error body parses. 403 can mean "blocked after too many errors
in a short window", so back off rather than retrying immediately.

See `errors/sendlane-problem-types.yml` and `conventions/sendlane-conventions.yml`.
