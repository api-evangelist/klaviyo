---
name: Report on Klaviyo campaign, flow and metric performance
description: Pull campaign and flow performance values, time series, and raw metric aggregates so an agent can answer "how did this send do" from the API rather than the UI.
api: openapi/klaviyo-reporting-api-openapi.yml, openapi/klaviyo-metrics-api-openapi.yml
generated: '2026-08-13'
method: generated
source: Grounded in openapi/*.yml (revision 2026-04-15)
operations:
  - query_campaign_values
  - query_flow_values
  - query_flow_series
  - query_segment_values
  - query_segment_series
  - query_form_values
  - query_form_series
  - query_metric_aggregates
  - get_metrics
  - get_metric
scopes:
  - campaigns:read
  - flows:read
  - segments:read
  - metrics:read
---

# Report on performance

## The reporting endpoints are POSTs, not GETs

This trips people up. Every reporting operation is a `POST` that carries a query object
in the body:

- `query_campaign_values` → `POST /api/campaign-values-reports`
- `query_flow_values` → `POST /api/flow-values-reports`
- `query_flow_series` → `POST /api/flow-series-reports`
- `query_segment_values` → `POST /api/segment-values-reports`
- `query_segment_series` → `POST /api/segment-series-reports`
- `query_form_values` → `POST /api/form-values-reports`
- `query_form_series` → `POST /api/form-series-reports`

They are reads despite the verb — POST is used because the query (statistics, timeframe,
filter, conversion metric) does not fit sanely in a query string.

## Values vs series — pick deliberately

- **`*_values`** returns aggregate totals for the timeframe. One number per statistic per
  entity. This is what you want for "how did this campaign do".
- **`*_series`** returns the same statistics bucketed over time. This is what you want
  for a chart or a trend.

Asking for a series when you only need a total is the most common way to make a
reporting call slower and larger than it needs to be.

## Every report needs a conversion metric

Klaviyo's revenue-oriented statistics are computed relative to a **conversion metric** —
usually "Placed Order". You must supply its metric id.

Resolve it first with `get_metrics` (`GET /api/metrics`) and find the metric by name.
Do not hard-code a metric id: metric ids are per-account, so an id that works in one
account is meaningless in another.

## Raw aggregates when the canned reports do not fit

`query_metric_aggregates` (`POST /api/metric-aggregates`) is the general-purpose
analytical endpoint. Give it a metric id, a measurement (count, sum of a property,
unique profiles), an interval, a timeframe and optional grouping, and it returns the
aggregate directly.

Use it when you need a cut the campaign/flow reports do not offer — revenue by product
property, event counts by a custom dimension, unique actors per day.

## Timeframes

Timeframes are supplied either as a named key or as explicit RFC 3339 start/end
datetimes. All datetimes in Klaviyo are ISO 8601 / RFC 3339
(`2026-01-16T23:20:50.52Z`). Be explicit about timezone — a report that silently uses a
different timezone than the account is the classic off-by-one-day reporting bug.

## Rate limits are tighter here than you expect

Reporting endpoints are computationally expensive and sit in the lower rate-limit tiers.
Do not poll them in a loop to build a dashboard. Fetch, cache, and refresh on a
schedule. Honour `Retry-After` on a `429`.

See `rate-limits/klaviyo-rate-limits.yml` for the tier table.

## Note on MCP

Klaviyo's MCP server exposes `get_campaign_report` and `get_flow_report`, which are
report-shaped wrappers over `query_campaign_values` and
`query_flow_values`/`query_flow_series`. If you are working through MCP you get the
convenience shape; if you are calling REST you assemble the query yourself. See
`mcp/klaviyo-tool-crosswalk.yml`.
