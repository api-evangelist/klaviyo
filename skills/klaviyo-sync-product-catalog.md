---
name: Sync a product catalog into Klaviyo
description: Load items, variants and categories into Klaviyo's catalog with bulk jobs so back-in-stock, price-drop and product-recommendation features have inventory to work from.
api: openapi/klaviyo-catalogs-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/klaviyo-catalogs-api-openapi.yml (revision 2026-04-15)
operations:
  - bulk_create_catalog_items
  - get_bulk_create_catalog_items_job
  - bulk_update_catalog_items
  - get_bulk_update_catalog_items_job
  - bulk_delete_catalog_items
  - bulk_create_catalog_variants
  - bulk_create_catalog_categories
  - create_catalog_item
  - create_catalog_variant
  - create_catalog_category
  - add_categories_to_catalog_item
  - get_catalog_items
  - get_variants_for_catalog_item
scopes:
  - catalogs:read
  - catalogs:write
---

# Sync a product catalog

Klaviyo's catalog is a three-level structure: **items** (products), **variants**
(purchasable SKUs under an item), and **categories** (groupings of items). Load them in
that order — a variant needs its item to exist, and a category association needs both
ends.

## Use the bulk jobs, not the singular endpoints

`create_catalog_item` (`POST /api/catalog-items`) exists and works, but a catalog sync
is a batch operation and the singular endpoint will put you into rate limiting fast.
Use the job endpoints:

- `bulk_create_catalog_items` → `POST /api/catalog-item-bulk-create-jobs`
- `bulk_update_catalog_items` → `POST /api/catalog-item-bulk-update-jobs`
- `bulk_delete_catalog_items` → `POST /api/catalog-item-bulk-delete-jobs`
- `bulk_create_catalog_variants` → `POST /api/catalog-variant-bulk-create-jobs`
- `bulk_create_catalog_categories` → `POST /api/catalog-category-bulk-create-jobs`

Each returns `202 Accepted` with a job id.

## The job pattern

Every bulk endpoint follows the same shape, and it is the same shape used by profile
import and suppression:

1. `POST` the batch → `202 Accepted`, response carries the job id.
2. Poll `GET /api/catalog-item-bulk-create-jobs/{job_id}`
   (`get_bulk_create_catalog_items_job`) until status is complete.
3. Read the job's errors before you declare success.

Do not fire-and-forget. A `202` means Klaviyo accepted the batch, not that the records
landed.

## Ordering a full sync

1. `bulk_create_catalog_categories` — categories first, so items have somewhere to go.
2. `bulk_create_catalog_items` — the products. Poll to completion.
3. `bulk_create_catalog_variants` — SKUs, referencing their parent item.
4. `add_categories_to_catalog_item`
   (`POST /api/catalog-items/{id}/relationships/categories`) — wire items to categories,
   or set the association in the item payload during step 2.

## Create vs update

Klaviyo's catalog bulk endpoints are **not** upserts. There are separate create, update
and delete job families, and a create against an existing external id will fail.

For a recurring sync, that means you must know which records are new. Either track state
on your side, or read the current catalog with `get_catalog_items`
(`GET /api/catalog-items`, cursor-paginated) and diff before you write. Reading the
catalog is cheap relative to failing half a create job.

## What the catalog unlocks

The catalog is not decorative — it is the data source for back-in-stock alerts,
price-drop alerts, and product recommendations in email, SMS and RCS. Stale catalog data
shows up as wrong prices in customer-facing messages, so treat sync freshness as a
correctness concern rather than a nice-to-have.

## Errors and limits

- `400` — `errors[0].source` gives the JSON Pointer to the offending record in your batch.
- `429` — honour `Retry-After`. Catalog bulk jobs are a common way to hit the account
  burst limit; pace the job submissions, not just the retries.
- Avoid adding `include` to catalog reads during a sync — `include` and
  `additional-fields` carry a stricter, globally enforced rate limit.

See `errors/klaviyo-problem-types.yml` and `rate-limits/klaviyo-rate-limits.yml`.
