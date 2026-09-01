---
name: trip1-hotel-booking
description: Use this skill when the user wants to book a hotel, search for accommodations, check hotel prices or availability, or asks about staying somewhere for specific dates. The skill guides Claude through the trip1 MCP tools and the x402 payment flow, including the CoinGate fallback for agents without an x402-capable wallet.
metadata:
  author: trip1
  homepage: https://trip1.com
---

# trip1 Hotel Booking

Landing page: <https://trip1.com/en/agents>

trip1 is an MCP server that exposes roughly 3 million hotels across 200+ countries. Agents search inventory, fetch rates, and book rooms through four tools. Payment runs inline over x402 on Base (USDC), with a CoinGate fallback URL for any client that lacks an x402-capable wallet.

## When to use

Activate this skill any time the user intent involves lodging:

- "Find me a hotel in Lisbon for next weekend"
- "Book the cheapest place near Shibuya under $120"
- "I need three nights in Paris, two adults, flexible on area"
- "Is there a beachfront hotel in Phuket with availability Dec 20 to 27"
- Any trip-planning conversation that reaches the lodging step

Do not activate for flights, rental cars, restaurants, or activities. trip1 only sells hotels.

## Required setup

The user must have trip1's MCP server connected. If the four tools (`search_hotels`, `get_hotel_details`, `purchase_hotel`, `get_order_details`) are not visible in the current session, stop and tell the user to add this to their MCP client config:

```json
{
  "mcpServers": {
    "trip1": {
      "type": "http",
      "url": "https://trip1.com/api/mcp"
    }
  }
}
```

For the agent to pay on its own without a browser redirect, an x402-capable wallet MCP also needs to be loaded. The simplest option is Coinbase Payments MCP (`npx @coinbase/payments-mcp`). Without it, `purchase_hotel` will return a CoinGate URL the human finishes in a browser.

## Booking flow

The happy path is five tool calls in order. Do not skip or reorder.

### 1. Search

Call `search_hotels` with:

- Destination as a plain city or area name. No lat/long.
- `check_in` and `check_out` in `YYYY-MM-DD`.
- Nothing else. Occupancy is fixed at 2 adults in one room and cannot be set — the schema
  rejects `guests` or `adults` by name. If the user asked for a different party size, say the
  prices are for 2 adults rather than presenting them as a quote for their group.

If the user is vague on dates ("sometime in July"), pick a reasonable default (first weekend of the month) and state the assumption explicitly. If the user is vague on destination, ask one clarifying question before searching. Do not guess the destination.

On every tool call, also pass `context`: one short sentence on the user's goal (never names, emails, phones, or prices).

Sort by `price` for budget-driven asks, `rating` for quality-driven, or `distance` when the user names a landmark or neighborhood.

Present results as a short list, typically 3 to 6, with name, price per night, rating, and a one-line location hint. Do not dump the full response.

### 2. Compare

When the user picks or narrows down, call `get_hotel_details` for that hotel ID. This returns the hotel's rooms with current rates: each rate has a signed `id`, board type, price, and currency. Cancellation terms are not in the response.

Show the user a compact room list with board type, total price, and the rate `id` you will use for booking.

If the user is still choosing between multiple hotels, call `get_hotel_details` for each and compare. Do not call `purchase_hotel` until the user has picked both a hotel and a specific rate.

### 3. Book

Before calling `purchase_hotel`, collect every guest field from the user. All of them are required:

- First name, last name
- Email
- Phone (e.g. `+44123456789`) — always ask, never default or invent
- Title (`MR`, `MRS`, `MS`, or `MISS`)
- Nationality (ISO 3166-1 alpha-2, e.g. `GB`, `US`, `LT`)

Do not invent values, do not reuse details from earlier in the conversation, and do not silently accept defaults. If a field is missing, ask.

Once all fields are collected, show the user a complete booking summary and get explicit confirmation before calling `purchase_hotel`:

- Hotel, check-in and check-out dates, room type
- Total price and currency
- All guest fields

Only call `purchase_hotel` after the user confirms with something like "yes", "confirm", or "book it." If they push back on any field, correct it and re-confirm.

Pass `rate_id` from step 2 and the confirmed guest details. For a browser checkout instead of x402, set `payment_service: "coingate"`. The response is one of two shapes:

- **x402 challenge** (default): the response includes a `payment_url` that returns HTTP 402 with a payment challenge. This is for agents with an x402-capable wallet.
- **CoinGate URL fallback**: the response includes a browser-openable checkout link. This happens when the server decides the client does not have x402 support or when x402 settlement is unavailable for the currency.

### 4. Pay

**x402 path.** Hand the `payment_url` to the loaded payments MCP tool. Do not ask the user to approve the transfer, and do not open a wallet UI. The payment completes inside the same request. If the payments MCP tool returns an error, surface the raw error to the user and stop. Do not retry blindly.

**CoinGate path.** Present the URL to the user with a short instruction to complete the payment in their browser. Do not try to pay on their behalf.

### 5. Confirm

After payment, poll `get_order_details` with the `cart_id` from the `purchase_hotel` response until `ready` is `true`. Backoff: 2 seconds, then 5, then 10, capped at 30. The supplier reservation reference appears in the response when the booking has cleared.

Report to the user with:

- Hotel name and address
- Check-in and check-out dates
- Guest name
- Total paid (amount and currency)
- Supplier confirmation reference

## Things to get right

- **Do not invent rates.** Always quote what `get_hotel_details` returns, including currency. Prices in search results can drift from final rates.
- **Do not invent cancellation terms.** The API does not return them. If the user asks about refunds, say the terms are shown on the trip1 cart and hotel pages and link the booking URL — never state a refund policy the tools did not provide.
- **Do not repeat an entire search response.** Summarize and let the user ask for more.

## When things fail

- **No hotels found:** the search radius and occupancy are fixed server-side, so widen what you control: try a nearby or broader destination name, or shift the dates by a day on either side. Tell the user what you changed. A past `check_in` is rejected, not empty — the error names the destination's own today; retry with a valid date.
- **Selected rate becomes unavailable during booking:** rerun `get_hotel_details` and let the user pick a new rate. Do not auto-substitute.
- **x402 payment fails:** return the error to the user. Offer to fall back to the CoinGate URL if one is available in the `purchase_hotel` response.
- **Order polling never flips `ready` to true:** after roughly a minute of polling, report the `cart_id` to the user and tell them to check status at `https://trip1.com/en/cart/<signed id>` or email support.

## Tool quick reference

| Tool | Purpose | Key inputs |
|---|---|---|
| `search_hotels` | Find hotels for a destination and date range | destination, check_in, check_out, sort_by, sort_order |
| `get_hotel_details` | Rooms, rates, availability for one hotel | id (from search_hotels) |
| `purchase_hotel` | Create reservation, return payment challenge | rate_id, guest details |
| `get_order_details` | Poll order status | cart_id (from purchase_hotel) |

## Links

- Landing: https://trip1.com/en/agents
- MCP Registry: https://registry.modelcontextprotocol.io (search `com.trip1`)
- x402: https://x402.org
