# Klaviyo GraphQL Schema

## Overview

This document describes a conceptual GraphQL schema for the Klaviyo marketing automation and customer data platform. Klaviyo exposes a REST API (v1–v3 and the newer revision-dated 2024-xx-xx API at `https://a.klaviyo.com/api/`), covering profiles, events, lists, segments, campaigns, flows, catalogs, coupons, metrics, reviews, templates, webhooks, tags, data privacy, and reporting. The GraphQL schema below models those same capabilities as a typed, queryable graph.

## Source

- Developer Portal: https://developers.klaviyo.com/
- API Reference: https://developers.klaviyo.com/en/reference/api_overview
- OpenAPI spec: https://github.com/klaviyo/openapi
- Changelog: https://developers.klaviyo.com/en/docs/changelog_

## Design Notes

- IDs follow Klaviyo's JSON:API convention — string UUIDs.
- Revision headers (e.g. `2024-10-15`) are modeled as an enum so callers can select the API version.
- Relationship edges (e.g. `List -> [Profile]`) mirror Klaviyo's `/relationships/` endpoints.
- Metric aggregation inputs map to the `Query Metric Aggregates` endpoint.
- Data-privacy requests follow the `Data Privacy` endpoint group.

## Type Categories

| Category | Types |
|---|---|
| Identity & Access | Account, APIKey, ClientToken |
| Profiles | Profile, ProfileIdentifier, SuppressedProfile, Suppression, BounceRecord |
| Events & Metrics | Event, EventMetric, Metric, MetricAggregate, MetricAggregateQuery, MetricAggregateResult |
| Lists | List, ListProfile |
| Segments | Segment, SegmentProfile, SegmentDefinition, Condition, ConditionalGroup |
| Campaigns | Campaign, CampaignMessage, CampaignSendJob, CampaignSchedule |
| Flows | Flow, FlowAction, FlowMessage, FlowFilter, FlowTrigger |
| Templates | Template, TemplateBlock |
| Catalog | Catalog, CatalogItem, CatalogVariant, CatalogCategory |
| Coupons | Coupon, CouponCode |
| Reviews | ReviewRequest, Review |
| Assets | Image |
| Alerts | BackInStockAlert, PriceDropAlert |
| Tags | Tag |
| Webhooks | WebhookEvent, WebhookSubscription |
| Subscriptions | Subscription, SubscriptionOptIn |
| Integrations | Integration |
| Web | WebFeed |
| Data Privacy | DataPrivacyRequest |
| Graph edges | Relationship |

## Schema File

See `klaviyo-schema.graphql` in this directory for the full SDL definition.
