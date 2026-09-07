---
name: shop-adro-parts
description: >-
  Search ADRO's US aero-parts catalog and build a cart or checkout through the store's live Universal
  Commerce Protocol MCP endpoint, without completing a payment the buyer has not approved.
api: ADRO US Store Agent Commerce (UCP / MCP)
base_url: https://adro.com
spec: mcp/adro1b33-ucp-mcp-tools.json
generated: '2026-09-07'
method: generated
source: >-
  Grounded in the verbatim tools/list response probed from https://adro.com/api/ucp/mcp (HTTP 200,
  anonymous) and the agent rules ADRO's own https://adro.com/llms.txt publishes.
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

# Buy ADRO parts as an agent

ADRO's US storefront exposes a live MCP endpoint. This is a **different company surface** from the AOX
platform API — different host, different auth, no shared operations.

- Endpoint: `POST https://adro.com/api/ucp/mcp`, `Content-Type: application/json`
- Discovery: `GET https://adro.com/.well-known/ucp`
- `tools/list` answers anonymously. `GET` on the endpoint returns 404 — it is POST-only.
- Protocol: Universal Commerce Protocol `2026-08-25` (also serves `2026-04-08` and `2026-01-23`)

## The hard rule

**`complete_checkout` must not be called without contemporaneous buyer approval of the payment.**
ADRO's own `llms.txt` states this outright. If you cannot get that approval at the moment of payment,
do not complete the checkout — route the purchase through the Shop skill
(`https://shop.app/SKILL.md`) instead, which carries the buyer-approval invariant for you.

## Flow

1. `search_catalog` — find products by buyer intent. Pass `context.address_country` and
   `context.currency`; pricing and availability depend on them.
2. `get_product` / `lookup_catalog` — resolve full detail or look several identifiers up at once.
3. `create_cart` → `update_cart` → `get_cart` — assemble the order. `cancel_cart` backs it out.
4. `create_checkout` → `update_checkout` (shipping address and method) → `get_checkout`.
5. `complete_checkout` — **only with buyer approval.** `cancel_checkout` reverses an incomplete one.
6. `get_order` — track afterwards.

## Money

Every price is an **integer in ISO 4217 minor units** paired with a currency code:
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before
quoting a figure to a person; zero-decimal currencies such as JPY are already whole units. Getting
this wrong quotes a buyer a price 100× off.

## Every call carries `meta`

Each tool requires a `meta.ucp-agent.profile` URI identifying your agent. It is not optional.

## Rate limits

Per-IP, unquantified. Back off on 429. No rate-limit headers are returned, so you get no advance
warning.

## Read-only alternative

If you only need catalog data and no transaction, `llms.txt` documents plain HTTP JSON:
`GET /products/{handle}.json`, `GET /collections/{handle}/products.json`, `GET /search?q=…&type=product`.
