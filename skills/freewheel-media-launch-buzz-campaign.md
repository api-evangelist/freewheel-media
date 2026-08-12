---
name: Launch a campaign on the FreeWheel Advertiser (Buzz) API
description: >-
  Authenticate against a Buzz tenant, then build the buy-side object chain —
  advertiser, campaign, targeting template, line item, creative, creative
  assignment — using the real Buzz resource operations.
api: openapi/freewheel-media-advertiser-buzz-openapi-original.json
base_url: https://buzz-key.api.beeswax.com/rest/
operations:
  - authenticate-post
  - advertiser-post
  - advertiser-get
  - campaign-post
  - targeting_template-post
  - line_item-post
  - creative-post
  - creativeasset-post
  - creativelineitem-post
  - line_item_flight-post
generated: '2026-08-12'
method: generated
source: >-
  openapi/freewheel-media-advertiser-buzz-openapi-original.json +
  https://api-docs.freewheel.tv/advertiser/docs/getting-started
---

# Launch a campaign on the FreeWheel Advertiser (Buzz) API

## Before you start

- The base host is **your tenant's**: `https://<buzz_key>.api.beeswax.com/rest/`.
  There is no shared host and no sandbox.
- Auth is a **session cookie**, not a bearer token. `authenticate-post`
  (`POST /authenticate`) with `{"email": "...", "password": "..."}` sets
  `<buzz_key>_buzz_cookie`. Send it on every later request.
  Default session life is an environment setting, typically 100 hours; add
  `"keep_logged_in": true` for a second cookie with 30-day expiry, and then read
  **and** write the cookie jar on every request because the value rotates.
- `/authenticate` is separately rate limited and answers **429** when you push it.
  Authenticate once and reuse the session.
- Every response is the Buzz envelope: `success`, `payload`, `message`, `error`,
  and `id` on create.

## Steps

1. **Authenticate** — `authenticate-post`.
2. **Find or create the advertiser** — `advertiser-get` (`GET /advertiser`,
   filter on your own `alternative_id` to check first), then `advertiser-post`
   if it is genuinely new. **There is no idempotency key on this API** — the
   read-before-write is the only duplicate protection you get.
3. **Create the campaign** — `campaign-post` (`POST /campaign`) with
   `advertiser_id`. Attach `bid_modifier_id` / `delivery_modifier_id` if you are
   using them.
4. **Create the targeting template** — `targeting_template-post`
   (`POST /targeting_template`). Targeting modules are fetched and validated
   separately at `/rest/targeting/{module_name}` — pull the available fields
   before you build the payload rather than guessing.
5. **Create the line item** — `line_item-post` (`POST /line_item`) with
   `campaign_id`, `advertiser_id`, `targeting_template_id` and
   `line_item_type_id`. Add flights with `line_item_flight-post`.
6. **Create the creative** — `creativeasset-post` (`POST /creative_asset`) to
   upload the asset, then `creative-post` (`POST /creative`) with
   `advertiser_id` and `creative_template_id`. Creative attribute modules are
   fetched at `/rest/attributes/{module_name}`.
7. **Assign the creative to the line item** — `creativelineitem-post`
   (`POST /creative_line_item`) with `creative_id` and `line_item_id`.

## Error handling

- **A 200 is not a success.** Read `success` in the body. Bulk `PUT`/`DELETE`
  returns 200 with a per-object array where each element has its own `success`,
  `error_code` (for example `AD_BAD_VALIDATION`) and `message`, plus a top-level
  `errors` array. Walk it.
- `406` is the only non-2xx response the contract declares at all, and only on
  `advertiser-put`, `account_alert-put`, `alert-post`, `alert-get` and
  `alert-put`. Everywhere else the spec promises 200 and the real failure modes
  are documented in prose only — see `errors/freewheel-media-problem-types.yml`.
- `429` on `/authenticate` means back off, not retry immediately.
- Audit what happened with `activitylog-get` (`GET /activity_log`) — it is the
  only correlation surface, as no request-id header is returned.

## Notes

- The harvested spec self-reports version `0.5`; a `2.0` resource series exists
  alongside it and 2.0 authentication works against both.
- Carry your own identifier in `alternative_id` on every object — it is the only
  metadata field the model offers.
