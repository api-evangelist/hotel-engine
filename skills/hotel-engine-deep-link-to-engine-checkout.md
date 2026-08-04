---
name: Deep link travelers into Engine checkout (Omni Swift)
description: >-
  Build the discovery experience yourself and hand the traveler off to Engine's hosted checkout
  with a deep link — the turnkey Omni Swift integration path.
api: openapi/hotel-engine-omni-partner-api-openapi.yml
generated: '2026-08-04'
method: generated
source: https://engine-public.github.io/engine-partner-api/deep-linking.html
operations:
  - ContentService_ListProperties
---

# Deep link into the Engine web experience (Omni Swift)

Omni Swift is the low-effort integration path. You own discovery; Engine owns room selection,
checkout, payments, confirmation, cancellations, and post-booking support. The only API surface
you need is content, plus a URL you construct.

Choose Swift over Halo when you are not travel-native, want fast time to market, and do not want
to handle payments or PCI scope. Choose Halo when you need control over every screen — see
`skills/hotel-engine-shop-and-book-lodging.md`.

## Step 1 — Power discovery from the content API

Call `ContentService_ListProperties` (`GET /content/v1/property`) over mTLS against
`partner-api.engine.com` to render your own property list: names, addresses, coordinates, hero
images, descriptions, amenities, star ratings, distance from the search point.

You need the `property_id` from each record — it is the key to the deep link.

Sandbox mTLS certificates are issued during onboarding; the Swift path needs them for exactly
this one call.

## Step 2 — Deep link to room and rate selection

Send the traveler to Engine's room-and-rate page for a property:

```
https://members.engine.com/properties/:propertyId
```

| Parameter | In | Required | Notes |
|---|---|---|---|
| `propertyId` | path | yes | The Engine property id, e.g. `P0000000000000102095` |
| `checkIn` | query | yes | ISO-8601 calendar date in the property's local timezone; must be `> now` |
| `checkOut` | query | yes | ISO-8601 calendar date; must be `>= checkIn` |
| `roomCount` | query | no | Default 1; `1 <= roomCount <= 8` |
| `guestCount` | query | no | Default 2; `1 <= guestCount <= 16` |

Dates are in the **property's** local timezone, not the traveler's and not UTC. Getting this
wrong is the classic off-by-one-night bug.

## Step 3 — Land new customers on your own branded page

If the traveler does not yet have an Engine account, route them through your custom landing
experience instead of dropping them cold into checkout:

```
https://members.engine.com/join/:slug?redirect_url=<url-encoded deep link>
```

`slug` is the slug Engine provisions for your landing experience. `redirect_url` is a
URL-encoded property deep link from step 2 — after sign-in or account creation the traveler
lands there. Omit it and they land on their dashboard instead, which loses the booking intent.
Always set it.

## Step 4 — Group travel and RFPs

For stays needing 9 or more rooms, hand off to the group RFP flow instead of the standard
checkout:

```
https://groups.engine.com/new-trip
```

Prefillable query parameters: `checkin` / `checkout` (either `MM/DD/YYYY` or ISO-8601, in the
destination timezone), `lat` + `lng` (each required if the other is present), `city` (free text
such as `Austin, TX` or `NYC`), and `sc` for your own search-attribution string.

## Attribution

`sc` on the RFP link is the documented hook for carrying your own attribution data through.
There is no documented equivalent query parameter on the property deep link — if you need
attribution on standard bookings, raise it with Engine during onboarding rather than inventing
a parameter.

## What you do not build

Payment capture, PCI scope, cancellation handling, refund logic, and traveler support all stay
with Engine on this path. You also do not get programmatic access to the resulting booking —
if you need `GetBookings`, folios, or API-driven cancellation, you are on the Halo path, not
Swift.
