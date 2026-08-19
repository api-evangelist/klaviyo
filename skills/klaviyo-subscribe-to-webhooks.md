---
name: Subscribe to Klaviyo webhooks and verify deliveries
description: Discover available webhook topics, create a subscription, verify the HMAC signature on every delivery, and re-enable a webhook Klaviyo disabled after repeated failures.
api: openapi/klaviyo-webhooks-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/klaviyo-webhooks-api-openapi.yml (revision 2026-04-15) + https://developers.klaviyo.com/en/docs/working_with_system_webhooks
operations:
  - get_webhook_topics
  - get_webhook_topic
  - create_webhook
  - get_webhooks
  - get_webhook
  - update_webhook
  - delete_webhook
scopes:
  - webhooks:read
  - webhooks:write
  - events:read
---

# Subscribe to webhooks and verify deliveries

## Step 1 — Discover the topics for THIS account

Call `get_webhook_topics` (`GET /api/webhook-topics`) first. Do not hard-code a topic
list. The available topics vary per account because they include integration-specific
and API-defined metrics on top of the standard set.

The standard set covers email (bounced, clicked, delivered, opened, marked as spam,
unsubscribed), SMS (clicked, received, sent, failed to deliver), push (received,
bounced, opened), reviews (ready to review, submitted rating, submitted review) and
subscription changes across channels.

`get_webhook_topic` (`GET /api/webhook-topics/{id}`) reads one.

## Step 2 — Create the subscription

`create_webhook` (`POST /api/webhooks`) with your endpoint URL, the topics you want, and
a **secret of at least 16 characters**. Klaviyo signs every delivery with that secret;
generate it randomly and store it where your receiver can read it.

## Step 3 — Verify every delivery

Klaviyo signs the request body with **HMAC-SHA256** and sends the result in the
`Klaviyo-Signature` header.

Verify before you trust the payload. Compute HMAC-SHA256 over the raw request body using
your webhook secret and compare against the header using a constant-time comparison.
Two things to get right:

- Sign the **raw bytes** of the body, not a re-serialised version of the parsed JSON.
  Re-serialising changes whitespace and key order and the signature will never match.
- Reject on mismatch. An unverified webhook endpoint is an open write path into your
  system.

## Step 4 — Respond fast, then work

Klaviyo expects a **200, 201 or 202 within 5 seconds**. Do the signature check, persist
the payload, return. Do the actual processing asynchronously. Anything you do inline
before responding is on the 5-second budget.

## The payload is batched

A single delivery may contain **up to 1,000 events**. The envelope is:

- `data[]` — each element carries `external_id`, `payload` and `topic`
- `meta` — `klaviyo_account_id`, `klaviyo_webhook_id`, `timestamp`

Write your receiver as a loop over `data[]` from day one. A receiver that assumes one
event per request works fine at low volume and breaks silently under load.

Use `external_id` for your own deduplication — deliveries can repeat under retry.

## Step 5 — Handle disablement

Klaviyo retries failures with exponential backoff up to a **maximum delay of one hour**.
If a webhook stays in an error state for **more than 48 hours, Klaviyo disables it.**

That is the failure mode to actually plan for: a deploy breaks your endpoint over a
weekend, and by Monday your event stream is off with no further retries and no backfill
of what was dropped.

- Monitor with `get_webhooks` / `get_webhook` and alert on a disabled state.
- Re-enable with `update_webhook` (`PATCH /api/webhooks/{id}`).
- Reconcile the gap by reading `get_events` over the outage window — the webhook
  transport dropped, but the events themselves are still in Klaviyo.

## Related artifacts

- `asyncapi/klaviyo-webhooks-asyncapi.yaml` — the event surface described as AsyncAPI 2.6.
- `errors/klaviyo-problem-types.yml` — API-level errors on the management endpoints.
