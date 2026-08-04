---
name: Shop and book lodging with the Engine Omni API
description: >-
  Take a traveler from a point of interest to a confirmed hotel booking using Engine's Omni
  Partner API — find properties, shop rates, lock a price, and purchase.
api: openapi/hotel-engine-omni-partner-api-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/_original/hotel-engine-omni-partner-api-2.4.0-swagger-original.json + https://engine-public.github.io/engine-partner-api/
operations:
  - ContentService_ListProperties
  - LodgingShoppingService_FindBestOffers
  - LodgingShoppingService_FindAvailability
  - LodgingBookingService_ConfirmOffer
  - LodgingBookingService_Book
---

# Shop and book lodging (Omni Halo)

Engine's Omni API is a four-stage pipeline, not a CRUD resource set: **Find Properties → Shop
Rates → Transact → Manage**. Each stage produces an object that the next stage consumes. You
cannot skip forward, and you cannot re-address an intermediate object by id — the handoff is an
opaque, single-use `continuation_token`.

## Before you start

- **Auth is mutual TLS.** There is no API key and no bearer token. You must present the client
  certificate and private key Engine issued you on every call, on both the gRPC and HTTP/JSON
  surface. Host: `partner-api.engine.com` (`:443` for gRPC).
- **Record `com-engine-request-id`** from every response. Engine support will not act on a
  report without it.
- **There is no idempotency key.** Read the retry rules at the bottom before you write any
  retry logic around `LodgingBookingService_Book`.

## Step 1 — Find properties

Call `ContentService_ListProperties` (`GET /content/v1/property`) with a point of interest —
geo coordinates, an address, or free text. Returns `ResponsiveProperty` records: static content
(name, address, coordinates, hero image, description, amenities, star rating, check-in/check-out
times) plus `distance` from the search point. Paginate with `page_size` / `page_token`, reading
`next_page_token` from the response.

Use this when you want to render a property list **without** prices. If you need prices to
decide which property to show, skip to step 2 instead — both paths converge at step 3.

If you already hold property ids, `ContentService_GetProperties`
(`POST /content/v1/properties`) fetches their content directly.

## Step 2 — Shop rates across properties

Call `LodgingShoppingService_FindBestOffers` (`POST /shop/v1/lodging/best-offers`) with
search criteria (radius around coordinates, or an explicit property-id list), the stay dates,
and traveler composition.

Returns, per property, the single lowest-priced `OfferSummary` plus availability flags —
`isRefundableAvailable`, `isLoyaltyAvailable`, `isFreeBreakfastAvailable` — and a
`continuation_token`.

**Best Offers are not exhaustive.** They are one offer per property, chosen for price. Never
present them as the full rate set.

If `sort_mode` is not valid for the criteria you supplied, this returns `INVALID_ARGUMENT` with
`FindBestOffersError.invalidSort`. Pick a compatible sort mode; do not retry unchanged.

## Step 3 — Get the real room and rate set

Call `LodgingShoppingService_FindAvailability` (`POST /shop/v1/lodging/availability`) with
either the `continuation_token` from step 2 or a bare `property_id`. **Prefer the token** — it
carries your shop criteria forward and yields more accurate results.

Returns `RoomGroup`s. One RoomGroup is one room type: a `RoomDescription` (bedding, amenities)
plus the `Offer`s available for it, which differ in amenities and cancellation conditions.

This is the screen where the traveler actually chooses. Surface each offer's `conditions`
(cancellation windows) and `price` breakdown — `Price` carries `base`, `taxesAndFees`,
`total`, `totalDueNow`, and `totalDueLater` separately, and due-now vs due-later matters to the
traveler.

## Step 4 — Lock the price

Call `LodgingBookingService_ConfirmOffer` (`POST /book/v1/lodging/confirm-offer`) with the
chosen offer's `continuation_token`.

Returns a `Quote`: the offer summary, room description, and `availableActions`, with the price
**locked for a limited window**. Treat the quote as perishable — show the traveler the price
from the quote, not from step 3, and re-confirm if they linger.

Failure modes:
- `ConfirmOfferError.offerNoLongerAvailable` — the offer sold out. Return to step 3 and re-shop.
- `ConfirmOfferError.invalidState` — this booking is already booked or is in customer-service
  review. **Do not retry.** Call `LodgingBookingService_GetBookings` to find out what happened.

## Step 5 — Book

Call `LodgingBookingService_Book` (`PUT /book/v1/lodging/booking`) with the quote's
`continuation_token`, `PaymentInfo`, and per-room guests.

`PaymentInfo` is one of two things: `engineDirectBill` (Engine consolidated invoicing) or
`paymentCard` (card plus billing address). Which one you may use is set by your partnership
agreement.

Guests go in `RoomGuests`: one `primaryGuest` (a `Guest` plus optional
`loyaltyProgramIdentifier`) and any `additionalGuests`, per room.

Returns `BookingDetails` with a stable `booking_id` — the first and only durable handle in the
whole flow. **Persist it immediately, before you render anything to the user.**

## Retry rules — read before writing retry logic

The Omni API has **no idempotency-key mechanism.** A `Book` call that times out is ambiguous,
and the token is single-use, so a blind retry is unsafe.

| Error on `Book` | Retriable | What to do |
|---|---|---|
| `offerNoLongerAvailable` | yes, after re-shopping | Go back to step 3 |
| `invalidState` | no | Call `GetBookings` — it may already have booked |
| `invalidPayment` | yes | Correct the payment details and re-book |
| `paymentProcessing` | **no** | Documented as unsafe to retry. Contact Member Support |
| `needsReview` (`INTERNAL`) | **no** | Booking is in an unknown state and needs manual review. Escalate with the `com-engine-request-id` |

On any timeout or ambiguous failure, **reconcile with `LodgingBookingService_GetBookings`
before doing anything else.** Never re-book to "make sure".

## Rate limits

Per endpoint, with a burst (per minute) and a sustained (per hour) allowance. `Book` is the
tightest write path at 20/minute and 900/hour. Watch `ratelimit-limit`,
`ratelimit-remaining`, `ratelimit-reset`; a 429 means back off until reset. Note that once the
sustained count drops below the burst allotment, `ratelimit-reset` reflects the next refill,
which may be smaller than the burst count — see `rate-limits/hotel-engine-rate-limits.yml`.

## Streaming

If you are on gRPC, `FindBestOffersStreaming` emits best offers as they are discovered and may
emit more than one per property; **the last one received always supersedes earlier ones.** It
is not available over HTTP/JSON.
