---
name: Build a 1MORE cart and take it to checkout
description: Assemble a cart, create and update a checkout, and stop at the payment boundary — 1MORE requires contemporaneous human approval to complete a purchase.
api: mcp/1more-mcp.yml
endpoint: https://usa.1more.com/api/ucp/mcp
operations: [create_cart, get_cart, update_cart, cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout, cancel_checkout, get_order]
generated: '2026-09-05'
method: generated
source: mcp/1more-ucp-tools-list.json, https://usa.1more.com/agents.md, https://usa.1more.com/policies/refund-policy
---

# Build a 1MORE cart and take it to checkout

This is the write path of 1MORE's UCP commerce surface. Read
`skills/1more-browse-catalog.md` first — you need product identifiers before any of
this works.

## The one rule that outranks the rest

The store states it plainly in its own `agents.md`:

> Checkout requires human approval. Agents must not complete payment without explicit
> buyer consent.

Do not call `complete_checkout` without contemporaneous buyer approval **at the moment
of payment**. If you cannot get it, stop at `update_checkout` and hand the buyer the
checkout, or route the purchase through the Shop skill the store recommends
(`https://shop.app/SKILL.md`).

## Steps

1. `create_cart` — pass `cart.line_items[]` as `{id, quantity}`, plus
   `cart.buyer.email` and `cart.context` (`address_country`, `currency`,
   `address_region`, `postal_code`).
2. `update_cart` / `get_cart` — adjust quantities, apply `cart.discounts.codes[]`,
   choose `cart.fulfillment.methods[]`. `cancel_cart` abandons it.
3. `create_checkout` — returns a checkout id in the form
   `gid://shopify/Checkout/abc123`. Read totals, taxes and discounts off the response.
4. `update_checkout` — set the shipping address and shipping method. The store's UCP
   profile declares `method_combinations: [["shipping"]]` and no multi-destination
   support, so one destination, shipping only.
5. **Get approval.** Show the buyer the line items and the total in major units.
6. `complete_checkout` — requires `meta.idempotency-key`, a string you generate and
   reuse verbatim on any retry of that same completion. This is the **only** tool that
   takes one, so it is the only write you can safely retry.
7. `get_order` — read the resulting order by id. It is read-only; there is no order
   cancel or refund tool.

## Reversibility

- Before completion: `cancel_checkout` and `cancel_cart` undo everything, with no
  published time limit.
- After completion: there is **no API reversal**. The buyer emails
  `support@1moreusa.com` for a refund or exchange, **within 30 days of the purchase
  date** (https://usa.1more.com/policies/refund-policy). Expedited shipping and
  shipping protection are not refunded. After 30 days the one-year limited warranty
  applies instead. Tell the buyer this before they approve.

## Errors and limits

- Errors arrive as JSON-RPC 2.0 error objects, not `application/problem+json`.
- `429` means back off; the endpoint is rate-limited per IP and publishes no numbers
  and no `Retry-After`.
- Only `complete_checkout` is replay-safe. Treat every other write as
  at-most-once — check state with `get_cart` / `get_checkout` before retrying.
