---
name: Browse the 1MORE catalog
description: Search, look up and read 1MORE product detail from the store's anonymous UCP/MCP endpoint without touching a cart or a payment.
api: mcp/1more-mcp.yml
endpoint: https://usa.1more.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-09-05'
method: generated
source: mcp/1more-ucp-tools-list.json, https://usa.1more.com/agents.md
---

# Browse the 1MORE catalog

1MORE sells wired and wireless earbuds and headphones. Its US store exposes a read-only
catalog to agents with **no credentials at all**. Use this skill when the buyer wants to
find, compare or price a product and has not asked you to buy anything.

## Connect

POST JSON-RPC 2.0 to `https://usa.1more.com/api/ucp/mcp` with
`Content-Type: application/json` and `Accept: application/json, text/event-stream`.
No `Authorization` header. Send `initialize`, then `tools/list` if you want the live
schemas.

Every call takes a required `meta.ucp-agent.profile` — the URI of your own agent
profile for UCP discovery. Send it on every request.

## Steps

1. **Search** — call `search_catalog` with `catalog.query` set to the buyer's words.
   Narrow with `catalog.filters` (`categories`, `price`, `available`) and page with
   `catalog.pagination.cursor` / `catalog.pagination.limit`.
2. **Set buyer context** — always pass `catalog.context.address_country` and
   `catalog.context.currency`. Prices and availability are wrong without them.
3. **Look up known items** — call `lookup_catalog` when you already hold product or
   variant identifiers from a previous turn, rather than re-searching.
4. **Read detail** — call `get_product` with `catalog.id`, and pass
   `catalog.selected[]` as `{name, label}` option pairs (colour, pack size) to resolve
   a specific variant.

## Rules

- **Money is in minor units.** `{"amount": 2500, "currency": "USD"}` is $25.00. Divide
  by 100 for two-decimal currencies before you quote a price. Zero-decimal currencies
  such as JPY are already whole units.
- **Back off on 429.** The endpoint is rate-limited per IP and publishes no numbers.
- **Do not create a cart to browse.** `create_cart` is a write; this flow is read-only.
- A plain HTTP alternative exists if you only need raw catalog data:
  `GET https://usa.1more.com/products.json`,
  `GET https://usa.1more.com/products/{handle}.json`,
  `GET https://usa.1more.com/collections/{handle}/products.json`.
