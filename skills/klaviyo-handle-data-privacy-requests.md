---
name: Handle a Klaviyo data privacy deletion request
description: Honour a GDPR/CCPA erasure request against a Klaviyo profile, and understand what deletion does and does not remove.
api: openapi/klaviyo-data-privacy-api-openapi.yml, openapi/klaviyo-profiles-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/klaviyo-data-privacy-api-openapi.yml (revision 2026-04-15)
operations:
  - request_profile_deletion
  - get_profiles
  - get_profile
  - bulk_suppress_profiles
scopes:
  - data-privacy:write
  - profiles:read
---

# Handle a data privacy deletion request

## The operation

`request_profile_deletion` (`POST /api/data-privacy-deletion-jobs`) is the whole
API surface for erasure. One operation, in its own API, behind its own scope
(`data-privacy:write`).

It takes a profile identifier — email address, phone number, or Klaviyo profile id — and
queues an asynchronous deletion job.

## Confirm the subject before you delete

Deletion is irreversible and there is no undo operation. Before calling it:

1. Resolve the identifier with `get_profiles`
   (`GET /api/profiles`, filtered on the identifier) or `get_profile`.
2. Confirm you have exactly one match, and that it is the right person. An email that
   matches two profiles means you have a duplicate-identity problem to resolve first,
   not a deletion to run twice.
3. Record what you are about to delete for your own compliance audit trail — once the
   job runs, you cannot go back and describe what was there.

## Scope it correctly — separate the private key

`data-privacy:write` is a high-consequence scope. Do not attach it to the same key an
ingestion pipeline or a reporting job uses. Issue a dedicated key for the privacy
workflow so a bug in an unrelated integration cannot delete customer records.

The same reasoning applies to agents: an agent that only needs to read profiles should
never hold a key carrying `data-privacy:write`.

## Deletion is not suppression — do not confuse them

- **Deletion** (`request_profile_deletion`) removes the profile and its data. It is what
  a GDPR Article 17 / CCPA erasure request asks for.
- **Suppression** (`bulk_suppress_profiles`) keeps the profile but stops all sending to
  it. It is what an opt-out or a hard bounce asks for.

Deleting a profile that merely opted out is over-compliance that destroys your own
suppression record — and if that address is later re-imported, nothing remembers it
should not be mailed. Reach for suppression unless the subject actually requested
erasure.

## Deletion is asynchronous

The call queues a job. The profile does not disappear on the response. Do not assert
completion to a data subject on the basis of the API accepting the request; verify the
profile is gone before you close the ticket.

## Cross-system obligation

Deleting from Klaviyo does not delete from the systems that feed Klaviyo. If your
ecommerce platform, CDP or warehouse re-syncs that customer, the profile reappears.
An erasure workflow has to suppress the source as well, or the deletion silently
reverses itself on the next sync.

## Errors

- `403` — the key lacks `data-privacy:write`.
- `404` — no profile matched the identifier. Treat as "already gone", but log it.
- `429` — honour `Retry-After`. Deletion sits in a low rate-limit tier; batch privacy
  work rather than looping.

See `errors/klaviyo-problem-types.yml`.
