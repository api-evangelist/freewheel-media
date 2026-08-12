---
name: Onboard a creative and assign it to a FreeWheel deal
description: >-
  Create a creative on a FreeWheel demand account, confirm it, and pre-ingest it
  against the programmatic deals it will serve on — the flow the FreeWheel
  Partner APIs exist for.
api: openapi/freewheel-media-demand-creative-management-openapi-original.json
base_url: https://api.freewheel.tv/demand/v1
operations:
  - CreateACreative
  - RetrieveAllCreatives
  - RetrieveAllCreativesWithoutAccountId
  - UpdateACreative
  - RetrieveUnderlyingCreatives
  - AssignACreativeToADeal
  - RetrieveADealAssignmentForACreative
  - DeleteACreativeDealAssignments
generated: '2026-08-12'
method: generated
source: >-
  openapi/freewheel-media-demand-creative-management-openapi-original.json +
  https://api-docs.freewheel.tv/demand/docs
---

# Onboard a creative and assign it to a FreeWheel deal

FreeWheel states the purpose of the Partner APIs plainly: *"Manage the
pre-ingestion and assignment of creatives to deals."* This is that flow.

## Before you start

- Token: `POST https://api.freewheel.tv/auth/token`, `grant_type=password`.
  Send `authorization: Bearer <access_token>` (case sensitive), valid 604800s.
- You need the `account_id` of the demand account the creative belongs to.
- Ceiling is **20 requests/second** across the whole estate.

## Steps

1. **Check what is already there** — `RetrieveAllCreatives`
   (`GET /accounts/{account_id}/ads`). Page with `offset` and `count`
   (max 50, default 10). Use `RetrieveAllCreativesWithoutAccountId`
   (`GET /ads`) when you want everything visible to the token instead of one
   account.
2. **Create the creative** — `CreateACreative`
   (`POST /accounts/{account_id}/ads`) with a `model.DemandAd` body. Video
   creatives carry a `model.DemandAdCreativeVideo`.
   **There is no idempotency key.** Before retrying a create that did not return
   cleanly, re-run step 1 and look for the creative — a blind retry makes a
   duplicate ad.
3. **Expand composites if needed** — `RetrieveUnderlyingCreatives`
   (`GET /accounts/{account_id}/ads/{ad_id}/underlying_creatives`) returns the
   `model.UnderlyingCreative` list behind a composite ad.
4. **Assign to the deals it will serve on** — `AssignACreativeToADeal`
   (`POST /accounts/{account_id}/deal_assignments`), passing the creative
   (`adid`) and the deal (`dealid`). This is the pre-ingestion step: it is what
   gets the creative approved against the deal before it is called on to serve.
5. **Verify the assignment** — `RetrieveADealAssignmentForACreative`
   (`GET /accounts/{account_id}/deal_assignments`) filtered by `adid` /
   `dealid`. Treat this as the confirmation, since the create response is not an
   idempotent receipt.
6. **Correct or withdraw** — `UpdateACreative`
   (`PUT /accounts/{account_id}/ads/{id}`) to change the creative;
   `DeleteACreativeDealAssignments`
   (`DELETE /accounts/{account_id}/deal_assignments`) to unassign it.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Bad or expired token | Re-issue at `/auth/token` |
| 404 | Resource not available | Check `account_id`, `adid`, `dealid` |
| 405 | Method not allowed | Wrong verb for the resource |
| 408 | Timeout from concurrent exclusive access | Retry |
| 422 | Invalid resource | The body or the created/updated object failed validation |
| 429 | Throttled | Drop below 20 req/s |
| 500 | API system error | Contact your FreeWheel account team |

Use the deal-status rules in `skills/freewheel-media-sync-programmatic-deals.md`
before assigning: an ARCHIVE deal will not accept changes.

## Notes

- Deal IDs here are FreeWheel Deal IDs. A Shared External Deal ID will 404.
- These creatives are **not** the same objects as Advertiser (Buzz) creatives;
  the two graphs are separate, on separate hosts, with separate credentials.
