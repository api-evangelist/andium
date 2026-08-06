---
name: Buy from the Andium Store
description: >-
  Discover, cart and purchase Andium merchandise through the Universal Commerce Protocol MCP
  endpoint at shop.andium.com, respecting the buyer-approval invariant on payment.
api: mcp/andium-mcp.yml
endpoint: https://shop.andium.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-06'
method: generated
source: mcp/andium-mcp-tools.json
---

# Buy from the Andium Store

All calls are JSON-RPC 2.0 `tools/call` requests to `POST https://shop.andium.com/api/ucp/mcp`
with `Content-Type: application/json` and `Accept: application/json, text/event-stream`.
Every tool takes a `meta` object; `meta["ucp-agent"].profile` (a resolvable agent profile URI)
is **required** on every call. Omitting it returns JSON-RPC error `-32001` / `invalid_profile_url`.

## 0. Confirm capabilities

`GET https://shop.andium.com/.well-known/ucp` — returns the merchant profile: supported protocol
versions (`2026-04-08`, `2026-01-23`), the `dev.ucp.shopping` service endpoint, the capability set
(`cart`, `checkout`, `fulfillment`, `discount`, `order`, `catalog.search`, `catalog.lookup`) and the
enabled payment handlers (`com.google.pay`, `dev.shopify.card`, `dev.shopify.shop_pay`).

## 1. Find the product

- `search_catalog` — free-text product search over the storefront. Pass buyer context
  (`context.address_country`, `context.currency`) so pricing and availability are correct.
- `lookup_catalog` — resolve several known product/variant identifiers at once.
- `get_product` — full detail for one product, including its variants.

## 2. Build the cart

- `create_cart` — creates a cart from the chosen variants; returns a cart `id`.
- `update_cart` — change quantities or line items.
- `get_cart` — re-read contents and totals.
- `cancel_cart` — abandon it.

## 3. Move to checkout

- `create_checkout` — returns checkout detail with line items, totals, discounts and taxes.
- `update_checkout` — set the shipping address and shipping method. Addresses use
  `street_address`, `extended_address`, `address_locality`, `address_region`, `postal_code`,
  `address_country` (ISO 3166-1 alpha-2), `first_name`, `last_name`.
- `get_checkout` — re-read before completing.

## 4. Complete — with a human in the loop

`complete_checkout` finalizes payment. Two rules are non-negotiable and both are published by the
provider:

1. **Explicit, contemporaneous buyer approval is required before payment.** Scripted form fills and
   end-to-end agent flows that finalize payment without approval are disallowed by
   `https://shop.andium.com/robots.txt`. If you cannot get approval at the moment of payment, route
   the purchase through the Shop skill (`https://shop.app/SKILL.md`) instead.
2. **Send `meta["idempotency-key"]`.** It is the only idempotency key in the whole tool set and it
   exists precisely so a retried completion does not place a duplicate order. Reuse the same key on
   every retry of the same completion.

`cancel_checkout` abandons an unpaid checkout. `get_order` reads the resulting order.

## Errors and backoff

Errors come back as JSON-RPC 2.0 error objects — `error.code`, `error.message`, and a `data` object
carrying `code`, `content` and sometimes `continue_url`. There is no RFC 9457 problem+json here.
The endpoint is rate-limited per IP; back off on `429`.

## Read-only alternative

If you only need to read, no MCP call is required:
`GET /collections/all`, `GET /products/{handle}.json`,
`GET /collections/{handle}/products.json`, `GET /search?q={query}&type=product`, `GET /sitemap.xml`.

## Scope warning

This is Andium's **merchandise storefront**. It is not Andium's industrial field-monitoring
platform. Nothing here reads wellsite, tank, flare, methane or emissions data — Andium publishes no
public API for any of that.
