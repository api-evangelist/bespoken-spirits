---
name: Browse the Bespoken Spirits catalog without credentials
description: Read the Bespoken Spirits product catalog and store policies with no
  authentication, using the storefront JSON endpoints and the anonymous storefront MCP
  server.
api: mcp/bespoken-spirits-mcp.yml
transport: HTTPS GET and MCP over HTTP (JSON-RPC 2.0) at https://bespokenspirits.com/api/mcp
operations:
  - search_catalog
  - get_product_details
  - search_shop_policies_and_faqs
generated: '2026-08-07'
method: generated
source: https://bespokenspirits.com/llms.txt
---

# Browse the Bespoken Spirits catalog without credentials

Use this when you only need to read — pricing research, product lookup, policy
questions — and do not intend to transact. Nothing here needs a key, a token, or a UCP
agent profile.

## Plain HTTP, no protocol

The store documents these itself in `https://bespokenspirits.com/llms.txt`:

| What | Request |
|---|---|
| Whole catalog | `GET https://bespokenspirits.com/products.json` |
| One product | `GET https://bespokenspirits.com/products/{handle}.json` |
| One collection | `GET https://bespokenspirits.com/collections/{handle}/products.json` |
| Search | `GET https://bespokenspirits.com/search?q={query}&type=product` |
| Sitemap | `GET https://bespokenspirits.com/sitemap.xml` |

`/products.json` returned 30 products by default and 47 with `?limit=250` at last
probe, spanning the Bespoken Spirits and Kentucky Bourbon Trail vendor names.
Pagination is via the platform's `limit` and `page` query parameters.

## Storefront MCP

`POST https://bespokenspirits.com/api/mcp` with `Content-Type: application/json` and
`Accept: application/json, text/event-stream`. `initialize` reports
`storefront-renderer 0.1.0`, MCP protocol `2025-06-18`. `tools/list` is anonymous and
returns five tools:

1. **`search_catalog`** — free-text product search over the store.
2. **`get_product_details`** — look up a product by id; pass `options` to pin a variant.
3. **`get_cart`** — read a cart, including shipping options, discount info and the
   checkout URL.
4. **`update_cart`** — add, remove or update line items and buyer information.
5. **`search_shop_policies_and_faqs`** — ask questions about the store's policies,
   products or services in natural language.

This surface has **no checkout and no order tools**. If you need to transact, switch to
the UCP endpoint at `/api/ucp/mcp` and follow
`skills/bespoken-spirits-buy-a-bottle.md`.

## Rules

- **Respect robots.txt.** Product, collection, page, blog, policy and cart HTML is
  explicitly crawlable. Completing a checkout by automation is explicitly forbidden.
- **Back off on 429.** Rate limits are per IP and no headers are published.
- **Prefer the JSON endpoints for bulk reads.** They are cheaper than driving MCP for
  data you could fetch in one GET.
