---
name: Build a FreeWheel audience from third-party segments
description: >-
  Walk from an advertiser industry category to the data providers allowed for it,
  to their segments, to a saved FreeWheel audience definition you can target.
api: openapi/freewheel-media-demand-audience-management-openapi-original.json
base_url: https://api.freewheel.tv/demand
operations:
  - list-industries-1
  - retrieve-data-providers-1
  - retrieve-segments-by-data-provider
  - filter-segment-searches
  - create-temporary-audience-definition
  - retrieve-audience-definitions
  - modify-audience-definitions
generated: '2026-08-12'
method: generated
source: openapi/freewheel-media-demand-audience-management-openapi-original.json
---

# Build a FreeWheel audience from third-party segments

## Before you start

- Authenticate as for the other Demand APIs (OAuth 2.0 password grant at
  `https://api.freewheel.tv/auth/token`). This spec also declares HTTP basic.
- **Every request carries `X-Freewheel-Ad-Industry`.** Audience and segment
  permissions are scoped by advertiser industry; without the right value you will
  see the wrong provider set.
- Stay under 20 requests/second.

## Steps

1. **Get the industry categories you are permitted to use** —
   `list-industries-1`
   (`GET /v1/services/v4/ds/v1/permissions/ad-industries`). Pick the category
   that matches the advertiser; that value is what goes in
   `X-Freewheel-Ad-Industry` on every later call.
2. **List the data providers for that industry** — `retrieve-data-providers-1`
   (`GET /v1/providers`). Provider availability is industry-dependent, which is
   why step 1 comes first.
3. **List a provider's segments** — `retrieve-segments-by-data-provider`
   (`GET /v1/segments`) with `providers`. Page with `page` and `per_page`.
4. **Narrow by keyword** — `filter-segment-searches` (`GET /v1/segments?`)
   with `keywords` plus the comparison operators the API exposes:
   `eq`, `neq`, `match`, `gt`, `lt`, `gte`, `lte`.
5. **Create the audience definition** —
   `create-temporary-audience-definition` (`POST /v2/audiences`) from the segment
   set you selected. Note the `temporary` flag: a temporary definition is for
   sizing and evaluation, a non-temporary one is for targeting. Decide which you
   want before you post — **there is no idempotency key**, so a retried create
   makes a second audience.
6. **Read your definitions back** — `retrieve-audience-definitions`
   (`GET /v2/audiences`). Do this after every create instead of trusting the
   POST response.
7. **Amend rather than recreate** — `modify-audience-definitions`
   (`PATCH /v2/audiences/{audience_id}`).

## Error handling

`401` means the token or basic credentials are wrong. A surprising empty result
is more often a wrong `X-Freewheel-Ad-Industry` than an actual absence — re-check
step 1 before concluding a provider has no segments. `429` means you crossed
20 req/s. See `errors/freewheel-media-problem-types.yml` for the full catalog.

## Notes

- This is the **Demand** segment graph. The Advertiser (Buzz) API has its own
  `segment` / `segment_category` objects keyed by `segment_key`; the two are not
  joined by anything published.
