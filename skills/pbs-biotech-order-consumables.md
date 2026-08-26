---
name: pbs-biotech-order-consumables
description: >-
  Buy PBS Biotech bioreactor consumables (single-use vessels, starter packs, sampling conicals) through
  the PBS Biotech online store's Universal Commerce Protocol MCP endpoint, up to the point of payment —
  which a human must approve.
api: PBS Biotech Store Agent Commerce (UCP/MCP)
endpoint: https://shoppbsbiotech.com/api/ucp/mcp
generated: '2026-08-26'
method: generated
source: >-
  Tool names, categories and required fields taken verbatim from a probed tools/list response
  (mcp/pbs-biotech-mcp-tools.json, HTTP 200, 2026-08-26). Flow order taken from the store's own
  /agents.md. Nothing here is invented.
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Ordering PBS Biotech consumables as an agent

PBS Biotech sells Vertical-Wheel bioreactors as capital equipment by quotation, and sells the
consumables around them — single-use vessels, starter packs, sampling conicals — through an online store
at `https://shoppbsbiotech.com`. Only the store is agent-callable. There is no API for the bioreactors
themselves and none for their control software.

## Before you start

1. Confirm capabilities with `GET https://shoppbsbiotech.com/.well-known/ucp`. It states the UCP version
   in force (`2026-04-08` at time of writing, with `2026-01-23` still supported) and which capabilities
   and payment handlers the merchant has enabled.
2. Every `tools/call` must carry `meta.ucp-agent.profile`, a resolvable agent profile URI. Without it the
   endpoint returns **HTTP 422** with JSON-RPC error `-32001` / `invalid_profile_url` and the tool never
   runs. This is a hard precondition, not an optional header.

## The flow

1. **Find the item** — `search_catalog` for a buyer intent, or `lookup_catalog` / `get_product` when you
   already hold an identifier. Prices come back as integers in ISO 4217 **minor units** paired with a
   currency code: `{"amount": 2500, "currency": "USD"}` is $25.00. Convert before quoting a price.
2. **Build the cart** — `create_cart`, then `update_cart` to adjust quantities. `get_cart` reads it back.
3. **Open a checkout** — `create_checkout`. The response carries line items, totals, discounts and taxes.
4. **Fulfilment details** — `update_checkout` to set the shipping address and method. The merchant
   profile declares single-destination shipping only (`allows_multi_destination.shipping: false`), so do
   not attempt to split a shipment.
5. **Complete** — `complete_checkout`. This tool alone requires `meta.idempotency-key` in addition to
   `meta.ucp-agent`. Generate the key once per intended purchase and reuse it on every retry; a fresh
   key on a retry is a second order.
6. **Confirm** — `get_order` with the resulting order identifier.

## Rules you must not break

- **Do not finalize payment without contemporaneous human approval.** The store states this in both
  `/robots.txt` and `/llms.txt`. Reaching `complete_checkout` autonomously is out of bounds; get the
  buyer's approval at the moment of payment, or route the purchase through the Shop Pay skill the store
  recommends.
- **Back off on 429.** The endpoint is rate-limited per IP. No numeric limit, window, or rate-limit
  response header is published, so treat the 429 itself as the only signal.
- **Do not scrape `/cart.js` or `/recommendations/products`.** `robots.txt` disallows them for agents and
  points at UCP/MCP instead.

## Undoing things

`cancel_cart` and `cancel_checkout` exist and are the only reversal tools on this surface. **No window is
published for either**, and there is no refund or void tool — refunds fall to the store's human refund
policy at `https://shoppbsbiotech.com/policies/refund-policy`. Do not promise a buyer a cancellation
window; nothing on this surface states one.

## What this skill cannot do

Quote or order a bioreactor system (PBS-3, PBS-15, PBS-80, PBS-Mini). Those go through
`https://pbsbiotech.com/request-a-quote`, which is a human form with no programmatic equivalent. Nor can
it touch bioreactor control software, process data, or run records — PBS Biotech publishes no interface
of any kind to those.
