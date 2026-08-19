---
name: Build a Klaviyo audience and manage marketing consent
description: Create lists and segments, add profiles, and subscribe or unsubscribe them from marketing across email and SMS with the consent record Klaviyo requires.
api: openapi/klaviyo-lists-api-openapi.yml, openapi/klaviyo-segments-api-openapi.yml, openapi/klaviyo-profiles-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/*.yml (revision 2026-04-15) + conventions/klaviyo-conventions.yml
operations:
  - create_list
  - get_lists
  - add_profiles_to_list
  - remove_profiles_from_list
  - get_profiles_for_list
  - create_segment
  - get_segment
  - get_profiles_for_segment
  - bulk_subscribe_profiles
  - bulk_unsubscribe_profiles
  - bulk_suppress_profiles
  - bulk_unsuppress_profiles
  - bulk_import_profiles
  - get_bulk_import_profiles_job
  - get_errors_for_bulk_import_profiles_job
scopes:
  - lists:read
  - lists:write
  - segments:read
  - profiles:write
  - subscriptions:write
---

# Build an audience and manage consent

## Lists vs segments — pick the right one

- **List** (`create_list`, `POST /api/lists`) is static membership. You put profiles in
  and take them out explicitly. Use it when *you* decide who belongs.
- **Segment** (`create_segment`, `POST /api/segments`) is a stored definition Klaviyo
  evaluates continuously. Membership changes on its own as profile and event data
  changes. Use it when *the data* decides who belongs.

You cannot add a profile to a segment. There is no such operation, by design. If you
find yourself wanting one, you want a list.

## Adding profiles to a list is not the same as consent

This is the mistake that gets people into deliverability trouble, so be explicit about
which one you are doing:

- `add_profiles_to_list` (`POST /api/lists/{id}/relationships/profiles`) adds membership.
  It does **not** grant marketing consent.
- `bulk_subscribe_profiles` (`POST /api/profile-subscription-bulk-create-jobs`) records
  **consent** for a channel (email, SMS) against a list, with the consent metadata
  Klaviyo needs.

If you want someone to receive marketing, you need the subscribe job. Adding them to a
list alone leaves them unsubscribed and unmailable.

`bulk_unsubscribe_profiles` (`POST /api/profile-subscription-bulk-delete-jobs`) is the
inverse. Both are job-shaped: they return `202 Accepted`, and you poll.

Note both endpoints are bulk-only. Klaviyo publishes no single-profile subscribe
operation — to subscribe one person you send an array of one.

## Suppression is a different axis

`bulk_suppress_profiles` / `bulk_unsuppress_profiles` control global suppression, not
per-list subscription. A suppressed profile receives nothing regardless of any list
subscription. Use suppression for hard bounces and complaints; use unsubscribe for a
person opting out of a specific list.

## Loading an audience at scale

For an initial load, use `bulk_import_profiles` (`POST /api/profile-bulk-import-jobs`).
It accepts a batch of profiles and optionally the list to place them on.

1. `bulk_import_profiles` → `202 Accepted`, returns a job id.
2. Poll `get_bulk_import_profiles_job` (`GET /api/profile-bulk-import-jobs/{job_id}`)
   until the status is complete.
3. **Always** then call `get_errors_for_bulk_import_profiles_job`
   (`GET /api/profile-bulk-import-jobs/{id}/import-errors`). A job can complete while
   individual records failed. Skipping this step is how silent data loss happens.

You can also read what actually landed with
`get_profiles_for_bulk_import_profiles_job` and `get_list_for_bulk_import_profiles_job`.

## Reading membership back

`get_profiles_for_list` (`GET /api/lists/{id}/profiles`) and
`get_profiles_for_segment` (`GET /api/segments/{id}/profiles`) both paginate with
`page[cursor]` and `page[size]`. Follow `links.next` until it is absent — do not compute
your own offsets, there are none.

Segment membership is evaluated continuously, so a segment read is a point-in-time
snapshot and the next page may reflect a slightly different population. For a stable
export, prefer a list or the Bulk Export APIs (Beta, 2026-07-15 revision).

## Errors

Consent operations are exactly where a `400` matters most — read `errors[0].source` for
the JSON Pointer into the offending record. See `errors/klaviyo-problem-types.yml`.
