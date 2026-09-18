# @pipeworx/reverb

Live listings from Reverb.com, the musical-instrument marketplace — guitars,
amps, pedals, synths, drums and pro audio, with make/model/year/finish/condition
and the seller's **asking price**.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1558+ live data sources.

## Tools

- `reverb_search(query, ...)` — listing search with the gear filters (make,
  model, product_type/category, condition, price band, year range, region,
  handmade, sort/order). Answers "what is a 1960s Gibson electric going for
  right now, and who is selling it".
- `reverb_listing(listing_id)` — one listing in full: seller description,
  asking and buyer price, offer/inventory state, photos, shipping rates by
  region.
- `reverb_shop(shop_slug)` — seller profile: feedback count, rating,
  preferred-seller status, location, shipping posture.
- `reverb_categories(root_slug?)` — the category tree, so a caller can fill
  `product_type` / `category` on `reverb_search` without guessing slugs.

## Auth

Keyless. Every endpoint this pack calls answers without a token — verified
2026-09-05. Only account-scoped Reverb endpoints (`/api/my/*`, orders, selling)
need OAuth, and none of those are exposed here, so there is no
`PLATFORM_REVERB_KEY` and no BYOK path.

## Asking prices only — there is no sold data here

Reverb's public Price Guide is **retired**. Both `/api/priceguide` and
`/api/priceguide/transactions` answer:

```
HTTP 403  { "Error": "This endpoint is no longer publicly available." }
```

for every query (verified 2026-09-05). Everything this pack returns is a live
asking price, not a realized sale, and every response says so in its
`price_basis` and `sold_data` fields. Do not use it as comps. For realized
instrument prices, Reverb is not the source.

## Data sources

- <https://api.reverb.com/api/listings/all> — listing search. Params used:
  `query, make, model, product_type, category, condition, price_min, price_max,
  currency, year_min, year_max, item_region, ships_to, handmade, sort, order,
  page, per_page`.
- <https://api.reverb.com/api/listings/{id}> — listing detail.
- <https://api.reverb.com/api/shops/{slug}> — seller profile.
- <https://api.reverb.com/api/categories/flat> — the full category tree (roots plus sub-categories, 320 entries). Plain `/api/categories` returns only the 14 roots.

Docs live at <https://www.reverb-api.com> (`dev.reverb.com` redirects there).

Things worth not rediscovering:

- The `Accept-Version: 3.0` and `Accept: application/hal+json` headers are what
  select the JSON API. Without them you can get HTML.
- **`total_pages` is capped at 50** regardless of `total` — a 77,081-result
  query still reports `total_pages: 50`. Requesting `page=51` nevertheless
  returns results, so treat the cap as a display artifact, not a paging bound.
  `reverb_search` returns a `total_pages_note` saying this.
- `price` and `buyer_price` differ when tax is involved; `listing_currency` is
  the seller's currency, while `price.currency` follows the `currency` param.
- Listing detail carries the shop's `feedback_count` and `rating_percentage`,
  which the search payload omits.
- `reverb.com/llms.txt` is a 404 as of 2026-09-05, despite reports otherwise.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "reverb": {
      "url": "https://gateway.pipeworx.io/reverb/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/reverb/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1558+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/reverb_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"stratocaster","make":"Fender","product_type":"electric-guitars","condition":"used","price_min":1000,"price_max":5000,"sort":"price","order":"asc","per_page":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/reverb_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "reverb": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-reverb"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-reverb
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Reverb data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
