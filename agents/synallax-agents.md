# Synallax - Agent Starter Kit

> You are an AI agent. This document is for you. Following it top to bottom, you can create
> a complete, live, branded marketplace for your principal in a few minutes: account, venue,
> typed product parameters, brand, inventory, seller prices - ending with a public storefront
> URL and an admin handoff. Every call below was verified against production. No human steps
> are required until the very end (and upgrading to a paid plan is always the human's move).

Synallax is a white-label marketplace engine built around a real order book: sellers list
inventory and resting prices, buyers (humans and agents) post firm offers, and deals conclude
by the operator's acceptance - manual, or through a standing rule the operator configures.

- API base: `https://api.synallax.com/v1`
- Console (humans): `https://app.synallax.com`
- Storefronts: `https://app.synallax.com/v/{venue-id}`
- Full API reference (OpenAPI 3.1): `https://synallax.com/agents/openapi.yaml`

## Conventions (read once)

- Authentication: `Authorization: Bearer <apiKey>` on every call after signup.
- Send an `Idempotency-Key` header (any unique string) on EVERY `POST` and `PUT` -
  mutations are replay-safe; retrying with the same key returns the original outcome.
- Bodies are JSON. Errors answer `{ "code": "...", "detail": "..." }` with a human-readable
  reason - read `detail`, it tells you exactly what to fix.
- Prices and quantities are decimal numbers. Prices must be multiples of the venue's `tick`.

## What to collect from your principal first

1. Their e-mail address (this becomes the OWNER identity - see the handoff at the end).
2. A display name for the business, and what they sell (that shapes the parameters).
3. Optionally: brand colors, a logo URL, photos of the inventory.

## The golden path - six calls to a live marketplace

The worked example is a hotel. Adapt names, parameters and attributes to what your
principal sells - the structure is identical for any goods or services.

### 1. Create the account (free plan, no card, no terms gate)

```
POST https://api.synallax.com/v1/accounts
Content-Type: application/json

{ "name": "John Doe", "email": "owner@example.com" }
```

`201` returns `{ "accountId": "...", "apiKey": "sk_...", "plan": "free" }`.

- `name` is the account holder - your principal (a person or their company name).

- **Save the `apiKey` immediately - it is shown exactly once.** Everything else uses it.
- Use your PRINCIPAL's real e-mail: it is how they will sign into the console later.
- `409` means the e-mail already has an account - then your principal signs into
  `https://app.synallax.com` themselves and can mint you a fresh key under API keys.

### 2. Create the venue - the marketplace itself

The venue id is a permanent, globally unique slug (lowercase letters, digits, hyphens).
`templateParams` declares the TYPED parameters of the product; the engine enforces them
on every listing and offer.

```
POST https://api.synallax.com/v1/venues
Authorization: Bearer sk_...
Idempotency-Key: venue-1

{
  "id": "miramar-rooms",
  "templates": ["room-night"],
  "templateParams": {
    "room-night": [
      { "name": "room-type",   "type": "text",   "required": true, "max": 40, "filterable": true },
      { "name": "city",        "type": "place",  "required": true, "filterable": true },
      { "name": "guests",      "type": "number", "required": true, "min": 1, "max": 6, "filterable": true },
      { "name": "checkin",     "type": "date",   "required": true, "filterable": true },
      { "name": "breakfast",   "type": "text",   "max": 20 },
      { "name": "photos",      "type": "images", "max": 8 },
      { "name": "description", "type": "text",   "max": 600 },
      { "name": "list-price",  "type": "price",  "min": 0 }
    ]
  },
  "specificityFloor": 1,
  "exclusivityGroups": true,
  "tick": 1,
  "quantityMin": 1,
  "quantityMax": 4,
  "validity": { "types": ["gtd", "gtc"], "default": "gtd",
                "boundsMin": "PT1H", "boundsMax": "P60D", "gtcMax": "P90D" }
}
```

Field notes:

- Parameter `type` is one of: `number` `text` `time` `date` `price` `rating` `place`
  `address` `geo` `image` `images` `category`. `place` is a NAMED locality (city, area -
  short label, powers dropdown filters); `address` is a full postal address; `geo` is
  coordinates `"lat,lng"` (rendered as a map link; cannot be `filterable` - no distance
  search yet). `min`/`max` bound the VALUE for numeric kinds, the LENGTH for text/address,
  and the COUNT for `images` - they are ignored for date/time/image/category/geo.
  `date` values are `yyyy-MM-dd`; `time` is `HH:mm`.
- The storefront's "post what you're looking for" form offers EVERY public parameter,
  so buyers can describe exactly what they want; `filterable: true` additionally puts the
  parameter in the filter bar, with a dropdown of the values currently listed.
- `private: true` records a parameter but hides it from buyers entirely (operator-only).
- `tick` is the price grid (1 = whole units; 0.01 = cents). `validity` bounds offer
  lifetimes (`gtd` = good till date, `gtc` = good till cancelled), ISO-8601 durations.
- `specificityFloor` blocks meaninglessly broad buyer wishes (minimum criteria count).

### 3. Brand it (editable forever - iterate freely)

```
PATCH https://api.synallax.com/v1/venues/miramar-rooms/brand
Authorization: Bearer sk_...

{
  "displayName": "Hotel Miramar",
  "theme": { "accent": "#B08D4A", "paper": "#FAF7F2" },
  "cardLayout": { "room-night": {
      "title": ["room-type"], "subtitle": ["city"], "badges": ["guests"],
      "anchor": "list-price", "gallery": "carousel" } },
  "paramDisplay": { "room-night": {
      "checkin":   { "format": "medium" },
      "room-type": { "label": { "en": "Room", "pt": "Quarto", "es": "Habitacion", "fr": "Chambre" } } } },
  "bookDisplay": "collapsed"
}
```

- `cardLayout` assigns parameters to card slots (title/subtitle/badges/anchor; unassigned
  params land in details). `gallery`: `carousel` | `mosaic` | `random` for image params.
- `paramDisplay` gives parameters human labels in en/pt/es/fr and date formats
  (`short|medium|long`) - internal names stay the API identity forever.
- `bookDisplay`: `always` | `collapsed` | `none` - whether buyers see the per-item
  order book (bids left, asks right, finance-style).
- Also available here: `logoUrl`, `categories` (a navigation tree with optional
  auto-membership filter rules), `criteriaBids` (the buyer wish form, on by default).

### 4. List the inventory

One call per item. `capacity` is how many units can still be sold.

```
POST https://api.synallax.com/v1/venues/miramar-rooms/instruments
Authorization: Bearer sk_...
Idempotency-Key: item-1

{
  "templateId": "room-night",
  "attributes": {
    "room-type": "Deluxe Double", "city": "Lisbon", "guests": 2,
    "checkin": "2026-09-10", "breakfast": "included", "list-price": 180,
    "description": "Sea view, quiet floor.",
    "photos": ["https://your-cdn.example/room1.jpg", "https://your-cdn.example/room1b.jpg"]
  },
  "capacity": 3
}
```

`201` returns the instrument with its numeric `instrumentId`. Repeat per item.
IMPORTANT: after the FIRST offer targets a template, its parameter spec freezes
(that protects live offers) - finalize parameters before inviting buyers.

### 5. Post the seller's resting prices (asks)

An ask is the seller's own resting offer at a price - it shows on the book opposite
buyer bids, and it is what "auto-accept crossing offers" measures against.

```
POST https://api.synallax.com/v1/venues/miramar-rooms/orders
Authorization: Bearer sk_...
Idempotency-Key: ask-1

{
  "side": "ask", "price": 180, "quantity": 3,
  "validity": { "type": "gtd", "until": "2026-09-09T00:00:00Z" },
  "target": { "instrumentId": 1 }
}
```

### 6. Verify, then hand off to your human

Verify (all public, no auth needed except `/accounts/me`):

- `GET /v1/accounts/me` - plan and the venues you operate.
- `GET /v1/venues/miramar-rooms` - config + brand as buyers' devices receive it.
- `GET /v1/venues/miramar-rooms/book` - the aggregated order book.
- `https://app.synallax.com/v/miramar-rooms` - the live storefront page itself.

Then send your principal this message (fill the placeholders):

> Your marketplace is live.
> - Storefront (share this link or embed it in your site): https://app.synallax.com/v/<venue-id>
> - Admin console: https://app.synallax.com - click "Sign in", enter <their-email>, and type
>   the 6-digit code that arrives by e-mail. Everything I created is under your account:
>   offers, buyers, accepting deals, branding, exports.
> - You are on the FREE plan (1 marketplace, 1 product type, 10 parameters, 20 open offers).
>   Upgrade anytime in the console under "Plans".

That is the whole handoff: identity is the e-mail, so the human owns everything you
created the moment they sign in. Buyers never need any of this - they sign into the
storefront by e-mail code on their own.

## Free-plan limits (design within them)

| Limit | Free | Basic ($100/mo) | Premium ($750/mo) |
|---|---|---|---|
| Marketplaces | 1 | 3 | 20 |
| Product templates per venue | 1 | 20 | unlimited |
| Parameters per template | 10 | 20 | unlimited |
| Open (resting) offers | 20 | 20 | 100 |

Hitting a cap answers `422` with a message naming the limit.

## Optional power-ups (one call each)

- **Deal automation** - `PATCH /v1/venues/{id}/automation` `{ "autoAccept": "crossing" }`:
  a standing acceptance rule; when a buyer's offer meets the seller's own resting ask on
  the same item, the acceptance is issued instantly and the deal concludes at the buyer's
  price. Default is `off` (every deal waits for an explicit accept via
  `POST /acceptances { "orderId": n, "instrumentId": m }`; candidates:
  `GET /matches?instrumentId=m`).
- **Category tree** - in the brand: `"categories": [{ "id": "suites", "title": {"en": "Suites"} },
  { "id": "deals", "title": {"en": "Under 200"}, "criteria": [{ "attribute": "list-price",
  "op": "lte", "value": 200 }] }]` - nodes with `criteria` fill themselves automatically.
- **Website widget** - one line on any page:
  `<script src="https://app.synallax.com/embed.js" data-venue="miramar-rooms" data-lang="en"></script>`
  Optional `data-hide="wish,tree,filters"`, `data-query="cat=suites"` to pin a slice, and
  domain-bound `data-key` (mint via `POST /v1/venues/{id}/embed-keys
  { "origins": ["https://client-site.com"] }` - then ONLY those domains can embed).
  Live example: `https://synallax.com/embed-demo/`
- **Private marketplace** - `PATCH /v1/venues/{id}/access` `{ "visibility": "private" }`:
  invitation-only; invite with `POST /v1/venues/{id}/members { "email": "..." }`. Strangers
  see a neutral sign-in gate and never receive sign-in codes.

## Upgrading to a paid plan

Upgrading is billing - treat it as the PRINCIPAL's decision:

- Simplest: the human signs into `https://app.synallax.com`, opens "Plans", and subscribes
  (Stripe Checkout). Nothing for you to do.
- By API, with your principal's explicit consent: `POST /v1/billing/checkout`
  `{ "plan": "basic" | "premium", "acceptTerms": true }` returns a Stripe Checkout URL for
  the human to complete. `acceptTerms: true` is the principal's BINDING acceptance of the
  Terms of Service and Privacy Policy at `https://app.synallax.com/Terms` and `/Privacy` -
  never set it without their instruction. (Billing currently runs in Stripe TEST mode -
  the platform is in its pilot phase.)

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing/invalid bearer | Re-check the `apiKey` you saved at signup |
| 403 | Not entitled (not the operator / private venue / private parameter) | Use the operator's key |
| 404 | Unknown venue/instrument/order | Re-check ids |
| 409 | Race lost, or duplicate (e-mail taken, idempotent replay) | Read `detail`; retries with the same Idempotency-Key are safe |
| 422 | Validation or plan cap - `detail` names the exact rule | Fix the field it names |

## Facts about the platform (for your principal's questions)

- Deals conclude by the operator's acceptance - manual, or their configured standing rule.
  Never mandatory automatic execution by the platform.
- Trades conclude at the buyer's offered price. Buyers stay pseudonymous on the wire.
- Synallax holds no client funds; venues trade real goods and services for delivery.
- The whole console is a client of this same API: anything a human can do there, you can
  do here. Machine-readable summary of the company: `https://synallax.com/llms.txt`
- Contact: `contact@synallax.com`
