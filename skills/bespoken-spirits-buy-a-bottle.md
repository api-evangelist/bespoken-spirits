---
name: Buy a bottle from Bespoken Spirits
description: Search the Bespoken Spirits catalog, build a cart, and take a checkout to
  the point of buyer approval over the store's Universal Commerce Protocol MCP endpoint.
api: mcp/bespoken-spirits-mcp.yml
transport: MCP over HTTP (JSON-RPC 2.0) at https://bespokenspirits.com/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - get_order
generated: '2026-08-07'
method: generated
source: mcp/bespoken-spirits-ucp-mcp-tools.json
---

# Buy a bottle from Bespoken Spirits

Bespoken Spirits sells bourbon and custom-finished whiskey from a Shopify storefront
that implements the Universal Commerce Protocol. There is no REST API and no API key.
Everything below runs against one JSON-RPC endpoint.

**Endpoint:** `POST https://bespokenspirits.com/api/ucp/mcp`
**Headers:** `Content-Type: application/json`, `Accept: application/json, text/event-stream`

## Before you start

- **Host a UCP agent profile.** Every `tools/call` must carry
  `params.meta.ucp-agent.profile` — a URI pointing at *your* agent profile. Without it
  the server answers JSON-RPC `-32001` / `invalid_profile_url` and does nothing else.
  `initialize` and `tools/list` are the only anonymous methods.
- **Pass buyer context.** Set `context.address_country` and `context.currency` on cart
  and checkout calls so pricing, availability and eligibility are correct.
- **This is alcohol.** Availability is jurisdictional. Treat an empty or filtered
  catalog result as a real answer, not a bug.

## Steps

1. **Confirm capabilities.** `GET https://bespokenspirits.com/.well-known/ucp` and check
   `ucp.version` (currently `2026-04-08`) against what you support. `fulfillment` and
   `discount` require a protocol minimum of `2026-04-08`.
2. **Find the product.** Call `search_catalog` with `catalog.query` and
   `catalog.context`. Narrow with `catalog.filters.categories`, `catalog.filters.price`
   and `catalog.filters.available`. Page with `catalog.pagination.cursor` +
   `catalog.pagination.limit`.
3. **Read the detail.** Call `get_product` (single) or `lookup_catalog` (several) to
   resolve variants and confirm the exact expression before adding it.
4. **Create the cart.** Call `create_cart` with `cart.line_items[]` (`id`, `quantity`),
   `cart.buyer` (`email`, `phone_number`) and `cart.context`. Keep the returned cart id.
5. **Adjust.** `update_cart` changes quantities, buyer details, fulfillment methods and
   `discounts.codes[]`. `get_cart` re-reads. `cancel_cart` abandons.
6. **Open the checkout.** `create_checkout` with `checkout.cart_id` set to the cart you
   built. Add `checkout.attribution` if you have referral or UTM context to pass through.
7. **Set shipping.** `update_checkout` with `checkout.fulfillment.methods[]` and the
   buyer's address. The store allows a single shipping destination per checkout and does
   not allow multi-destination fulfillment.
8. **Attach payment.** Put a payment instrument on `checkout.payment.instruments[]` —
   each needs `id`, `handler_id` and `type` (`card` for cards, `token` for Google Pay /
   Apple Pay wallets). The handlers this store advertises are `com.google.pay`,
   `dev.shopify.card` and `dev.shopify.shop_pay`.
9. **STOP. Get the buyer's approval.** Do not proceed without an explicit,
   contemporaneous human OK at the moment of payment. The store states this rule in
   `llms.txt`, in `agents.md`, and in `robots.txt`, which additionally forbids scripted
   form fills and end-to-end browser automation that finalizes payment. If you cannot get
   that approval in-band, route the purchase through the Shop skill at
   `https://shop.app/SKILL.md` instead.
10. **Complete.** `complete_checkout` requires **both** `meta.ucp-agent.profile` and
    `meta.idempotency-key`. Generate the idempotency key once per purchase attempt and
    reuse it on every retry — this is the only tool of the thirteen that takes one, and
    it is the only one that moves money.
11. **Confirm.** `get_order` with the returned order id.

## Rules

- **Retry only `complete_checkout` with the same key.** No other tool is declared
  idempotent, so a blind retry of `create_cart` or `create_checkout` creates a duplicate.
- **Back off on 429.** The endpoint is rate-limited per IP. No rate-limit headers are
  published, so use exponential backoff.
- **Read the error `data` block.** Failures arrive as JSON-RPC error objects with
  `error.data.code`, `error.data.content` and a `error.data.continue_url` a human can
  open. See `errors/bespoken-spirits-problem-types.yml`.
- **Do not cache prices.** Pricing and availability are computed from the buyer context
  you pass; a cached price from a different `address_country` or `currency` is wrong.
