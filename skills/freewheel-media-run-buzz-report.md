---
name: Run and collect a FreeWheel Buzz report
description: >-
  Save a report definition, queue it asynchronously, poll the queue, and collect
  the result — the Buzz reporting pattern, which is request/poll rather than
  request/response.
api: openapi/freewheel-media-advertiser-buzz-openapi-original.json
base_url: https://buzz-key.api.beeswax.com/rest/
operations:
  - authenticate-post
  - report_save-post
  - report_save-get
  - report_queue-post
  - report_queue-get
generated: '2026-08-12'
method: generated
source: >-
  openapi/freewheel-media-advertiser-buzz-openapi-original.json +
  https://api-docs.freewheel.tv/advertiser/docs/reporting
---

# Run and collect a FreeWheel Buzz report

Buzz reports are **asynchronous**: you queue a run and then poll for it. There is
no webhook, no callback and no event stream anywhere in FreeWheel's published API
surface, so polling is the only mechanism available.

## Before you start

- Authenticate once with `authenticate-post` and reuse the
  `<buzz_key>_buzz_cookie` session — `/authenticate` is rate limited.
- Base host is your tenant: `https://<buzz_key>.api.beeswax.com/rest/`.
- Global ceiling is 20 requests/second. Because collection is a poll loop, pick a
  poll interval in seconds, not milliseconds.

## Steps

1. **Save the report definition** — `report_save-post` (`POST /report_save`).
   Reuse the returned `report_save_id` on later runs instead of re-saving; the
   definition is the durable object and the run is the disposable one.
   You can read your saved definitions back with `report_save-get`.
2. **Queue a run** — `report_queue-post` (`POST /report_queue`) referencing the
   `report_id`. The response envelope's `id` is your `report_queue_id`.
   **No idempotency key exists** — record the returned id before doing anything
   else, because a retried queue call produces a second run.
3. **Poll for completion** — `report_queue-get` (`GET /report_queue`) filtered
   by `report_queue_id`. Back off between polls; a 408 here means concurrent
   exclusive access and is explicitly safe to retry.
4. **Collect the output.** Buzz can return CSV and Excel as well as JSON for
   reporting responses, so set your accept/format according to what you intend
   to load downstream.

## Error handling

- Check the envelope `success` flag, not just the HTTP status.
- `408` — retry. `429` — you are over 20 req/s, widen the poll interval.
- `500` — contact your FreeWheel account team; there is no working status page to
  check first (the Statuspage tenant at freewheel.statuspage.io is deactivated).

## Notes

- The 0.5 reporting resources were deprecated in favour of the 2.0 reporting API;
  FreeWheel announced this in the Advertiser changelog and published no migration
  deadline, notice period or Sunset header.
- Multi-account reporting was added in the same 2.0 series — check the changelog
  entry before assuming a report is single-account.
