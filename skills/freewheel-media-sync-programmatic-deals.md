---
name: Sync FreeWheel programmatic deals
description: >-
  Pull the programmatic deals a buyer seat has on FreeWheel, page through them
  safely, and set the buyer status on a deal so the buy-side system and
  FreeWheel agree.
api: openapi/freewheel-media-demand-deal-sync-openapi-original.json
base_url: https://api.freewheel.tv/demand/v1
operations:
  - RetrieveDeals
  - RetrieveSingleDeal
  - UpdateBuyerStatusForDeals
generated: '2026-08-12'
method: generated
source: >-
  openapi/freewheel-media-demand-deal-sync-openapi-original.json +
  https://api-docs.freewheel.tv/demand/docs
---

# Sync FreeWheel programmatic deals

## Before you start

- You need credentials **issued by FreeWheel**. There is no self-service signup.
- Get a token: `POST https://api.freewheel.tv/auth/token` with
  `grant_type=password&username=<user>&password=<pass>`. The response carries
  `access_token`, `token_type: Bearer` and `expires_in: 604800` (7 days).
- Send it as `authorization: Bearer <access_token>`. **Bearer tokens are case
  sensitive.**
- Check a token you already hold with `GET https://api.freewheel.tv/auth/token/info`
  rather than re-issuing on every run.
- Stay under **20 requests/second**. Above that, calls fail. There are no
  rate-limit response headers to read, so pace yourself on the clock.

## Steps

1. **List the deals for your seat** — `RetrieveDeals` (`GET /deals`).
   Filter with `seatid`, `sellerstatus`, `startdate`, `enddate`,
   `updatedat` and `preingestpermissions`. For an incremental sync, pass
   `updatedat` set to your last successful run and take only what changed.
2. **Page with `offset` and `count`.** `count` maxes at **50** and defaults to
   10; `offset` is zero-based. Keep requesting until you get a short page.
   An empty deal list comes back as **HTTP 200 with an empty list**, not a 404 —
   do not treat it as an error.
3. **Fetch detail where you need it** — `RetrieveSingleDeal`
   (`GET /deals/{id}`). Use the FreeWheel Deal ID, not an External Deal ID.
4. **Write the buyer status back** — `UpdateBuyerStatusForDeals`
   (`PATCH /deals/{id}`) with the `model.PatchDeal` body.

## Error handling

There is **no idempotency key** on this API. A `PATCH` that times out must be
re-driven by re-reading the deal with `RetrieveSingleDeal` and comparing, not by
blind retry.

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing/invalid credentials | Re-issue the token; check the header casing |
| 404 | `FreeWheel SSP (SFX) deals are not supported.` | The deal is an SFX deal — it cannot be updated here |
| 404 | Deal ID not in the seller network, or Shared External Deal ID | You sent the External Deal ID; ask the seller for the FreeWheel Deal ID |
| 405 | Method not allowed | Read-only resource, or wrong verb |
| 408 | Request timeout from concurrent exclusive access | **Retry** |
| 422 | `Deal has been archived and is no longer relevant.` | Seller status is ARCHIVE; stop trying to set buyer status |
| 429 | Throttled | You exceeded 20 req/s — back off |
| 500 | API system error | Contact your FreeWheel account team |

## Notes

- Deals live in a different object graph from Advertiser (Buzz) campaigns and
  from Publisher (MRM) placements. Nothing in the published contracts joins them.
- Sunset/Deprecation headers are not supported, and there is no working status
  page, so schedule a periodic re-read of the reference rather than relying on a
  runtime signal.
