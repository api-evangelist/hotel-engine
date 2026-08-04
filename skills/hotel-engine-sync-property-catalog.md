---
name: Sync the Engine lodging property catalog
description: >-
  Pull and keep current a local copy of Engine's lodging property catalog, including per-entry
  catalog state, using the Omni Partner API content surface.
api: openapi/hotel-engine-omni-partner-api-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/_original/hotel-engine-omni-partner-api-2.4.0-swagger-original.json
operations:
  - CatalogService_ListPropertyCatalog
  - ContentService_GetProperties
  - ContentService_ListProperties
---

# Sync the property catalog

Engine's inventory spans a large lodging catalog. If you are building search, ranking, or
mapping over it, you want a local mirror of the static content and to call the shopping
endpoints only for live prices.

Authenticate with your mTLS client certificate against `partner-api.engine.com`.

## Full catalog pull

`CatalogService_ListPropertyCatalog` (`POST /content/v1/catalog/property`) returns
`PropertyCatalogEntry` records — each a `Property` plus a `PropertyCatalogState`.

The state field is the point of this endpoint: it tells you whether an entry is current, so
your mirror can retire properties that have left the catalog instead of silently keeping stale
rows. Diff against your local copy on every run and apply removals, not just upserts.

Paginate with `page_size` / `page_token`, following `next_page_token` until it is empty.

## Targeted refresh

`ContentService_GetProperties` (`POST /content/v1/properties`) fetches full content for a
specific set of property ids. Use it to re-hydrate rows your diff flagged as changed, rather
than re-pulling the whole catalog.

This is the one operation in the contract with an **untyped 400** — its `400` response declares
no schema, so on invalid parameters you get only the `rpcStatus` message string with no typed
detail to branch on. Validate ids before sending.

## Geographic lookup

`ContentService_ListProperties` (`GET /content/v1/property`) finds properties near a point of
interest by geo, address, or text, and returns each with its `distance` from that point. This
is a query surface, not a sync surface — use it to serve user searches, not to build the mirror.

At 400/minute and 18,000/hour it is by far the most generous endpoint in the API, which is a
deliberate signal: discovery traffic is expected to be heavy, transaction traffic is not.

## What a Property record contains

`id` (format `P` + 19 digits, e.g. `P0000000000000102095`), `name`, `physicalAddress`
(`PostalAddress`), `coordinates` (`GeoPoint`), `heroImageUri`, `description`, `mediaItems`,
`amenities` (each a `PropertyAmenity` carrying a `LodgingAmenityCode`), `starRating`,
`checkInTime` / `checkOutTime`, `emails`, `catalog` (external catalog identifiers), and
`loyaltyRewardsProgram`.

`catalog` / `ExternalCatalogIdentifiers` is the field to key on if you are reconciling Engine
properties against another supplier's inventory — it is the cross-walk to external catalogs.

## Rate limit budgeting

`ListPropertyCatalog` shares no documented limit line of its own on Engine's rate-limits page;
the published table covers `ListProperties`, the shopping RPCs, and the booking RPCs. Treat
catalog pulls conservatively, run them off-peak, and watch `ratelimit-remaining` on every
response rather than assuming a ceiling.

## Do not cache prices

Property content is stable and safe to mirror. Offers, quotes, and prices are not — they are
volatile, carried by single-use `continuation_token`s, and price-locked only inside a
`ConfirmOffer` window. Never persist an offer and serve it later as if it were live.
