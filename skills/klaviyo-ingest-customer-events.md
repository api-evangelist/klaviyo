---
name: Ingest customer profiles and events into Klaviyo
description: Upsert a customer profile and record the events that describe what they did, safely and idempotently, so segments, flows and reports have something to act on.
api: openapi/klaviyo-profiles-api-openapi.yml, openapi/klaviyo-events-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/*.yml (revision 2026-04-15) + conventions/klaviyo-conventions.yml
operations:
  - create_or_update_profile
  - create_event
  - bulk_create_events
  - get_profiles
  - get_profile
  - merge_profiles
scopes:
  - profiles:write
  - profiles:read
  - events:write
---

# Ingest customer profiles and events

The profile/event pair is Klaviyo's core. Everything downstream — segments, flows,
campaigns, reports — reads from it. Get this right and the rest of the platform works.

## Before you call anything

- Base URL is `https://a.klaviyo.com`. Server-side calls live under `/api/`.
- Send `Authorization: Klaviyo-API-Key <private-key>` on every request.
- Send a `revision` header with an ISO date, e.g. `revision: 2026-07-15`. It is
  **required**. Omitting it does not fail cleanly — Klaviyo falls forward to a supported
  revision, which means your behaviour can change under you. Set
  `X-Klaviyo-Revision-Fall-Forward-Opt-Out: 1` if you would rather fail loudly.
- Write bodies use `Content-Type: application/vnd.api+json` and the JSON:API envelope
  (`{"data": {"type": ..., "attributes": {...}}}`).

## Step 1 — Upsert the profile

Use `create_or_update_profile` (`POST /api/profile-import`), **not** `create_profile`.

`create_profile` (`POST /api/profiles`) is a strict create and will conflict on a
profile that already exists. `create_or_update_profile` is the upsert: it matches on
`email`, `phone_number` or `external_id` and updates in place if found. For an
ingestion pipeline that may replay, the upsert is the only correct choice.

Identity rules from Klaviyo's docs: you must supply at least one of `email`,
`phone_number` or `external_id`. `phone_number` must be in E.164 form.

If you discover you created two profiles for one person, `merge_profiles`
(`POST /api/profile-merge`) reconciles them. It is destructive — the source profile is
consumed. Read both with `get_profile` first.

## Step 2 — Record the event

Use `create_event` (`POST /api/events`). The body carries the metric name, the profile
identifiers, the event properties, and — this is the important part — a `unique_id`.

**Always set `unique_id` yourself.** It is Klaviyo's deduplication key and the only
thing that makes this call safe to retry. Klaviyo's own schema says: *"A unique
identifier for an event. If the unique_id is repeated for the same profile and metric,
only the first processed event will be recorded."* Use your own system's event ID —
an order ID, a message ID, a UUID you generated.

If you omit it, Klaviyo defaults `unique_id` to the timestamp truncated to the second.
That caps you at **one event per profile per metric per second**, and a retry inside
that same second silently disappears while a retry a second later silently duplicates.
Both failure modes are invisible. Set the field.

Note there is no `Idempotency-Key` header anywhere in Klaviyo's API. `unique_id` is the
whole idempotency story, and it only covers event creation.

## Step 3 — Backfill history separately

When you are migrating and want to load historical events **without** setting off every
flow those events would trigger, use the `backfill` flag introduced in the 2026-07-15
revision. This is the difference between a clean migration and mailing your entire list
about orders they placed two years ago.

## Step 4 — Bulk, when volume demands it

`bulk_create_events` (`POST /api/event-bulk-create-jobs`) accepts a batch and returns
`202 Accepted` with a job resource. Poll the job until it completes.

Watch the semantics change: on the bulk endpoint a repeated `unique_id` for the same
profile and metric causes **the whole request to fail** rather than deduplicating
silently. Deduplicate within your batch before you send it.

## Errors and retries

- `429` — honour `Retry-After` (integer seconds). On a 429 the `RateLimit-Limit` /
  `RateLimit-Remaining` / `RateLimit-Reset` triplet is replaced by `Retry-After`, so
  handle both header shapes.
- `400` — read `errors[0].source`. It gives you either a JSON Pointer into your body or
  the name of the offending query parameter.
- `409` — a genuine conflict. Re-read before retrying; do not blind-retry.
- `410` — your `revision` header names a retired revision. Klaviyo supports a revision
  for two years (one stable, one deprecated).
- `500` / `503` — retry with exponential backoff. Quote `errors[0].id` to support.

Full catalogue: `errors/klaviyo-problem-types.yml`.

## Rate limits

Limits are per-account and tiered per endpoint (XS 1/s–15/m through XL 350/s–3500/m).
Requests using `include` or `additional-fields` are subject to a stricter, globally
enforced limit — do not add `include` to a high-volume ingestion loop just for
convenience. See `rate-limits/klaviyo-rate-limits.yml`.
