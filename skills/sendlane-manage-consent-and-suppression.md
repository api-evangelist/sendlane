---
name: sendlane-manage-consent-and-suppression
description: >-
  Record and revoke email and SMS consent, unsubscribe contacts, and manage email
  and domain suppression lists in Sendlane — the compliance surface of the v2 API.
api: sendlane:sendlane-api
base_url: https://api.sendlane.com/v2
operations:
  - post-contacts-email-consent
  - get-contacts-contactId-sms-consent
  - post-contacts-sms-consent
  - delete-contacts-contactId-sms-consent
  - post-contacts-search-sms-consent
  - post-contacts-unsubscribe
  - get-contacts-unsubscribed
  - get-lists-unsubscribed
  - post-suppression
  - get-suppression
  - post-domain-suppression
  - get-domain-suppression-suppressionId
  - delete-domain-suppression-suppressionId
generated: '2026-08-13'
method: generated
source: openapi/sendlane-openapi.yml
---

# Manage consent and suppression in Sendlane

Consent is a first-class resource in the Sendlane v2 API, which makes this the part of
the surface with real legal weight. Get it wrong and you send marketing SMS to someone
who never opted in.

## Before you start

- Auth: `Authorization: Bearer <SENDLANE_API_TOKEN>`, `Accept: application/json`.
- **Gate:** the Contact SMS Consent endpoint "must first be enabled in your account by
  contacting support" — the spec states this in its own operation descriptions. Plan
  for a support conversation before you can write SMS consent at all.
- No sandbox. Everything here mutates production consent state.

## Email consent

```
POST /contacts/{contactId}/email-consent
{ "email_consent": true, "list_ids": [ ... ] }
```
`post-contacts-email-consent`. `email_consent` is required. `list_ids` scopes it.

## SMS consent

```
GET    /contacts/{contactId}/sms-consent    get-contacts-contactId-sms-consent
POST   /contacts/{contactId}/sms-consent    post-contacts-sms-consent
DELETE /contacts/{contactId}/sms-consent    delete-contacts-contactId-sms-consent
POST   /contacts/sms-consent                post-contacts-search-sms-consent
```

`ContactSmsConsentRequest.v2` requires **six** fields, and the list tells you what
Sendlane is actually recording — an auditable opt-in:

- `phone`
- `sms_consent`
- `optin_date`
- `optin_url` — the page the consent was collected on
- `ip_address` — the consenting user's IP
- `user_agent` — their browser

Optional: `consent_language` (the exact wording shown at opt-in) and `cookie_uuid`.

**Capture these at the moment of opt-in.** They cannot be reconstructed later, and
they are the evidence a TCPA complaint turns on. If your form does not record IP,
user agent and the consent language you displayed, fix the form before you write the
integration.

`post-contacts-search-sms-consent` (`POST /contacts/sms-consent`) does the same thing
addressed by search criteria rather than by contact id — useful when you have a phone
number but no contact id.

## Unsubscribe

```
POST /contacts/{contactId}/unsubscribe    post-contacts-unsubscribe
GET  /contacts/unsubscribed               get-contacts-unsubscribed
GET  /lists/{listId}/unsubscribed         get-lists-unsubscribed
```
The unsubscribe POST takes no body. The two GETs are paginated (`limit`, `page`) and
support `from`/`to` datetime filters — `from` is required whenever `to` is present.

**These GETs are how you learn about unsubscribes.** Sendlane sends no outbound
webhooks, so a downstream system that must honour opt-outs has to poll these on a
schedule. Poll `get-contacts-unsubscribed` with a rolling `from`/`to` window rather
than re-reading the whole list.

## Suppression

```
GET    /suppression/emails                     get-suppression
POST   /suppression/emails                     post-suppression
GET    /suppression/domains                    get-domain-suppression-suppressionId
POST   /suppression/domains                    post-domain-suppression
DELETE /suppression/domains/{suppressionId}    delete-domain-suppression-suppressionId
```

`StoreEmailSuppressionRequest.v2` requires `email`; `list_id` scopes the suppression to
one list rather than the whole account. Returns **201**.

Domain suppression blocks an entire sending domain and is deletable; email suppression
has no delete operation in the published spec — treat it as one-way.

**Watch the operationId.** `get-domain-suppression-suppressionId` is the operationId
Sendlane assigned to `GET /suppression/domains` — the *collection* index, which takes
no path parameter. The name is misleading and there is no
`GET /suppression/domains/{suppressionId}` at all; that path supports DELETE only.
Bind to the path, not to the name.

## Errors and safety

Envelope: `{ "message": string, "errors": object|array }`. On 422 the `errors` object
is keyed by the failing attribute.

There is **no idempotency key**. Do not blind-retry a consent write on a timeout —
read the current state with `get-contacts-contactId-sms-consent` first, then decide.

See `errors/sendlane-problem-types.yml` and `conformance/sendlane-conformance.yml`.
