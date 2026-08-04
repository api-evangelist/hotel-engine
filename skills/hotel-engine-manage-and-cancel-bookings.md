---
name: Manage and cancel Engine lodging bookings
description: >-
  Retrieve bookings, produce a folio, preview a cancellation's refund, and cancel — the
  post-booking half of Engine's Omni Partner API.
api: openapi/hotel-engine-omni-partner-api-openapi.yml
generated: '2026-08-04'
method: generated
source: openapi/_original/hotel-engine-omni-partner-api-2.4.0-swagger-original.json + https://engine-public.github.io/engine-partner-api/
operations:
  - LodgingBookingService_GetBookings
  - LodgingBookingService_GenerateFolio
  - LodgingBookingService_PreviewCancellation
  - LodgingBookingService_SubmitCancellation
---

# Manage and cancel bookings (Omni Halo)

Everything after `Book` keys off one durable identifier: `booking_id`. Authenticate with your
mTLS client certificate against `partner-api.engine.com` and capture `com-engine-request-id`
from every response.

## Read bookings

`LodgingBookingService_GetBookings` (`POST /book/v1/lodging/booking`) takes a list of
`booking_ids` and returns `BookingDetails` records — status, reservation details (the realized
quote, the property, and the payment actually applied), cancellation details, metadata, and
per-room guests.

**This is a partial-failure operation.** On `INVALID_ARGUMENT` the error carries
`GetBookingsError.errors[]`, one `GetBookingError` per booking that could not be retrieved,
each with its `bookingId` and — when available — a nested `rpcStatus` explaining why. Some
bookings in the same call can succeed while others fail. Handle per booking; retry only the
failed ids.

This is also the **reconciliation endpoint**. Any time a `Book` call times out or fails
ambiguously, come here before taking any other action. Never re-book to find out.

At 20/minute and 900/hour, this is a tight budget — batch your ids rather than polling one at a
time. On gRPC, `GetBookingsStreaming` covers large reconciliation sweeps; it is not available
over HTTP/JSON.

## Generate a folio

`LodgingBookingService_GenerateFolio` (`POST /book/v1/lodging/booking/generate-folio`)
returns an Engine-branded folio as `google.api.HttpBody` — a **binary PDF**, not a JSON
envelope. Handle the response as bytes and set the content type from the body, not from your
JSON client's defaults.

Only a confirmed booking has a folio. If the booking has not been booked, this returns
`FAILED_PRECONDITION` with `GenerateFolioError.invalidState`.

## Always preview before you cancel

`LodgingBookingService_PreviewCancellation`
(`POST /book/v1/lodging/booking/preview-cancellation`) returns the availability, the means, and
the **expected refund** for cancelling a booking.

Run this first, every time, and show the traveler the refund number before asking them to
confirm. Cancellation conditions vary per offer — a rate that was non-refundable at shop time
is still non-refundable now, and the preview is the only place that tells you the money outcome
without committing.

Failure: `PreviewCancellationError.invalidState` — the booking was never booked, or is already
cancelled. Check with `GetBookings` first.

## Cancel

`LodgingBookingService_SubmitCancellation`
(`POST /book/v1/lodging/booking/submit-cancellation`) attempts the cancellation.

Failure modes, all `INVALID_ARGUMENT`:
- `actionNotAvailable` — carries `ActionAvailability`. Inspect it; the action is not offered
  for this booking in its current state.
- `cannotCancel` — the booking is not cancellable at all, typically by rate rule. Preview would
  have told you.
- `invalidState` — not booked, or already cancelled. Reconcile with `GetBookings`.

This is the tightest endpoint in the whole API: **10/minute, 450/hour.** It is also
irreversible and refund-bearing. If an agent is driving this flow, require human confirmation
before submitting — there is no undo and no idempotency key.

## Error handling in general

Every error is a `google.rpc.Status` (`code`, `message`, `details[]`). The actionable content
is the typed message inside `details[]`; discriminate by which field of that message is
populated, since the sub-errors are empty marker messages. See
`errors/hotel-engine-problem-types.yml`.

Note the contract declares no 401/403 responses despite mTLS being mandatory, and no 429
despite documented rate limits — handle both defensively at the transport layer rather than
expecting them in the spec.
