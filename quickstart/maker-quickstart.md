# Acta Maker Quickstart

JSON-layer path for a maker connection: auth, subscribe, quote, handle fills, reconnect. Rust integrations should start with [`maker-rust-sdk.md`](maker-rust-sdk.md). Payload examples are in [`maker-wire-examples.md`](maker-wire-examples.md); onboarding is in [`../reference/sandbox.md`](../reference/sandbox.md); units and envelopes are in [`../reference/ws-common.md`](../reference/ws-common.md); the message catalogue is in [`../reference/maker-api.md`](../reference/maker-api.md).

The wire is closed. Every field is present on every frame, absent facts are explicit `null`, and state-dependent shapes are tagged unions. Unknown message types, error codes, enum values and missing fields are rejected, by the server and by your decoder.

---

## Connection and authentication

The first message from the client is `Hello`. All four fields are required.

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "on_disconnect": "cancel_quotes",
    "client_name": "maker-bot",
    "client_version": "0.1.0"
  }
}
```

- `protocol_version` must equal the server's version, `1.0.0`. There is no version range. Any other value gets an `Error` with `Generic` code `unsupported_protocol_version`, then a close with code `4006`.
- `on_disconnect` sets what happens to this connection's quotes when it ends. With `cancel_quotes`, the engine cancels the session's active and retained non-winning quotes. With `keep_quotes`, resting quotes stay live while the maker is offline. Selected and executing orders survive both.
- There is no feature negotiation. `QuoteExpired` events are always sent.

The server responds with `Welcome` (`server_time_unix_ms` is for clock sync), then `AuthRequest` with a challenge string. Validate the entire [canonical auth challenge](../reference/ws-common.md#what-to-sign) before invoking your signer. Then sign its original UTF-8 bytes with `quote_signing`, base58-encode the signature, and reply with `AuthChallenge`:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "<from AuthRequest>",
    "signature": "<base58 ed25519 signature>",
    "pubkey": "<maker_owner pubkey, base58>"
  }
}
```

`pubkey` is the `maker_owner` wallet registered on-chain, not the `quote_signing` key. The server looks up the registered signing key from `maker_owner` and verifies against it. Auth must finish within 15 seconds of the challenge; the default limit allows three failed attempts and closes the connection on the fourth.

## Subscription

A resumed maker auth session restores server-side routing and mint scope. After fresh auth, establish subscriptions again. `Subscribe` replaces both the product selection and the mint lists on either path, so send it after every connect.

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "products": ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
    "channels": ["rfqs", "chain_events"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

All fields are required. `request_id` is a UUID that the server echoes in `SubscribeAck`.

- `products` is a nonempty list. An empty list is rejected with `invalid_products`. RFQ fan-out follows it.
- `[]` in `underlying_mints` or `quote_mints` means every mint.
- Channels are additive across `Subscribe` calls. Products and mints are replaced.

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. `AddMints` and `RemoveMints` take both mint lists, and `[]` means no change. Each mutation returns `SubscriptionUpdated` with the current channels and both mint lists.

## Products

Every RFQ, quote, position and trade carries `terms` and `side`.

| `terms.type` | Fields | Settlement |
|---|---|---|
| `covered_call`, `cash_secured_put` | `strike` | physical |
| `call_spread`, `put_spread` | `lower_strike`, `upper_strike` | cash, in the quote mint |
| `cash_put` | `strike` | cash, in the quote mint |

- `side` is `sell` or `buy`. Only `sell` is executable. `buy` is rejected with `taker_buy_unsupported`.
- `cash_put` executes on chain as a put spread with legs `(0, K)`.
- Strikes are 1e9-scaled. Spread strikes and `cash_put` strikes must be multiples of the underlying mint's `strike_tick` from `GetTokens`.
- A market's `products` lists only the pair's products on that market's side. Call markets list `covered_call` and `call_spread`. Put markets list `cash_secured_put`, `put_spread` and `cash_put`.
- Spreads and cash puts settle in cash from `open` to `settled`. They have no funded state and no liquidation.

Details: [Products and terms](../reference/maker-api.md#products-and-terms).

## Quoting

Units: `rfq.quantity` (and `size_rule`, `QuoteFilled.quantity`) is in underlying atoms; strikes and `price` are 1e9-scaled per underlying token, independent of decimals. Decimals are not in `RfqBroadcast`: read `underlying_decimals` and `quote_decimals` from `GetMarketDescriptors` and cache them by `market.market_pda`. The gross premium (before protocol fees) in quote atoms is `floor(price * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`, the contract's rounding.

On `RfqBroadcast`, first verify the [market PDA and terms](../reference/maker-api.md#quote-flow) against your configured Acta program. The program ID is the same on devnet and mainnet; `GET /api/v1/venue` returns it as `program_id`. Then choose the terms to quote:

- Covered call and cash-secured put: the strike of `rfq.terms` or of an entry in `rfq.order_options`.
- Spread and cash put: exactly `rfq.terms`.

Compute the premium, set `now + min_quote_validity_seconds <= valid_until <= market.expiry_ts` (read `min_quote_validity_seconds` from `GET /api/v1/venue` at startup; do not hardcode it) and keep `valid_until − settlement_buffer_seconds` after now. Build the [order preimage](../reference/maker-api.md#order-id-preimage): 182 bytes for covered calls and cash-secured puts, 190 bytes for spreads and cash puts. Its `maker` field is the `maker_owner` pubkey. Check your builder against the [test vectors](../reference/maker-api.md#test-vectors). Hash it to `order_id`, sign the 32-byte hash with `quote_signing`, and send `Quote`:

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "price": 5000000000,
    "valid_until": 1710000120,
    "nonce": 42,
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "signature": "<base58 of ed25519(order_id)>"
  }
}
```

A spread quote repeats the RFQ's two strikes:

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "terms": { "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 },
    "side": "sell",
    "price": 5000000000,
    "valid_until": 1799999900,
    "nonce": 999,
    "order_id": "0x0101010101010101010101010101010101010101010101010101010101010101",
    "signature": "<base58 of ed25519(order_id)>"
  }
}
```

Spread and cash-put prices must satisfy `0 < price < width`, where `width` is `upper_strike − lower_strike` (`strike` for a cash put). The gross premium in quote atoms must be positive and below the collateral `C = ceil(width * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`. Otherwise the quote is rejected with `spread_premium_invalid`. For the 100/120 call spread above with 1 SOL (`quantity` 1000000000, 9 and 6 decimals), `C` is 20000000 and the gross premium is 5000000.

The protocol fee is `min(premium fee, volume fee)`. The venue publishes `protocol_fee_bps_premium` and `protocol_fee_bps_volume` on `GET /api/v1/venue`. The formulas are under [Cash settlement and fees](../reference/maker-api.md#cash-settlement-and-fees). The authoritative net premium for a fill is `total_premium` on its position row. The maker's cost is the gross premium, and the fee is that cost minus `total_premium`.

When an RFQ permits several strikes, each strike requires a distinct quote. Prefer `BatchQuotes` for several strikes or RFQs: one frame and one `BatchQuotesAck` instead of one message per quote. Each quote requires its own `order_id` and `nonce`. A new quote on the same `(rfq_id, terms)` replaces the prior quote for that pair.

Use `ReplaceQuote` for repricing: the swap is atomic and single-RTT. `CancelQuote` followed by `Quote` introduces a gap in the book during the cancellation window and incurs an additional round-trip.

`is_taker_buy` in the order-id preimage is `0` for side `sell`; the taker is always the option writer. Side `buy` sets it to `1` and is rejected. Implementations whose preimage builders default this field to `true` must override it explicitly.

`QuoteRejected` carries `reason` and a `message`. Fix the cause before retrying; the same payload will fail again. Two reasons are not about the payload: `rate_limited` (the connection's quote bucket is empty; back off and re-sign with a fresh nonce) and `trading_paused` (the venue is paused; wait for it to resume).

## Lifecycle events

Lifecycle events are keyed by `order_id`. `RfqClosed` is the terminal event for an RFQ; `QuoteFilled` is a fill-details event that arrives after `RfqClosed`.

| Event | Description |
|---|---|
| `QuoteAcknowledged` | Server accepted the quote. `replaced_order_id` is set on a replace and `null` otherwise. |
| `QuoteRejected` | Server refused the quote; see `reason`. |
| `QuoteBestStatus` | Quote is currently the best in the book. |
| `QuoteOutbid` | Quote has been displaced by a better one. |
| `QuoteRefreshRequested` | Settlement-buffer cutoff is approaching; resubmit with `valid_until ≥ min_valid_until` before the cutoff. Identified by `(rfq_id, terms)`. `reason` is `expired` or `expiring_soon`. |
| `QuoteSelected` | Quote locked; awaiting the taker signature. |
| `QuoteFilled` | Position opened on-chain. Carries `position_pda`, `tx_signature`, `terms` and `side`. It has no fee or net-premium field. |
| `QuoteCancelled` | Terminal. `reason` ∈ {`requested`, `risk_check`, `rfq_accepted`, `maker_disconnected`}. |
| `QuoteExpired` | Always sent when a quote expires. `reason` ∈ {`expired`, `expiring_soon`}. |
| `RfqAvailableAgain` | Settlement reverted; the auction has reopened. The maker may re-quote with a fresh `order_id`. |
| `RfqClosed` | Terminal RFQ event. Drop per-RFQ state. |

Book money from the position row, not from the fill. `PositionUpdated` carries the full row with `total_premium`, the net premium the taker received, in quote atoms. The maker's cost is the gross premium `floor(price * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`, and the fee is that cost minus `total_premium`. Each row has a `revision`; apply a row only if its revision is newer than the stored one for the same `pda`. Position `state` is `open`, `funded`, `liquidated` or `settled`. A settled row carries `settlement`, either `physical` or `cash`.

## On-chain responsibilities

Quoting and the lifecycle above are entirely off-chain (WebSocket). Your only on-chain actions as a maker are funding-related:

- `DepositPremium` — deposit program quote balance; **required before quoting** (the fill's premium debit draws from it). [Layout](../reference/sandbox.md#on-chain-maker-instructions). `WithdrawPremium` retrieves idle balance.
- `DepositFundsToPosition` — optional, after a covered-call or cash-secured-put fill: fund the settlement leg (`open` → `funded`) to avoid the ITM-unfunded liquidation loss. Spreads and cash puts have no funded step. [Layout](../reference/sandbox.md#on-chain-maker-instructions).

You do **not** settle or liquidate. Publishing the settlement price and finalizing markets is operator-side; settlement is keeper-driven; liquidating ITM-unfunded physical positions is permissionless. Your downside is bounded — fund the settlement leg of a physical position, or a third party liquidates and fronts the taker payout. See [`../reference/protocol-flow.md`](../reference/protocol-flow.md) for the risk model.

## Indicative pricing

Indicative pricing covers covered calls and cash-secured puts. If the account is enrolled in pre-trade pricing, the server periodically emits `IndicativePricesRequest` for reference prices to be displayed in the taker UI. Indicative quotes are non-binding and operate under a tighter latency budget than auction quotes. The response is correlated by `request_id`:

```json
{
  "type": "IndicativePricesResponse",
  "data": {
    "request_id": "<from request>",
    "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "position_type": "covered_call",
    "prices": [
      { "strike": 150000000000, "price": 4500000000 },
      { "strike": 160000000000, "price": 5000000000 }
    ]
  }
}
```

## Reconnection

With `on_disconnect: "cancel_quotes"`, the engine removes the connection's active and retained non-winning quotes on disconnect; selected/executing obligations survive. With `keep_quotes`, resting quotes can remain fillable while offline. Resume restores server subscriptions; fresh auth requires restoring them. The server does not replay events missed during the disconnect window. Events already in flight can arrive again after recovery, so process lifecycle events idempotently by `order_id`.

Recover in this order after reauthentication. Use `/maker/data` for the data reads and `/maker` for the quote plane.

**1. Quote plane, `/maker`.** This is what the managed Rust SDK does before it reports `Ready`.

- Send `GetSubscriptions`. If the server holds channels you do not want, send `Unsubscribe` for them.
- Send `Subscribe` with your desired products, channels and both mint lists. Wait for `SubscribeAck`.
- Send `GetActiveRfqs` and `GetMyQuotes` with `scope: { "type": "live" }`.

```json
{ "type": "GetSubscriptions", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000004" } }
{ "type": "GetActiveRfqs",    "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000001" } }
{ "type": "GetMyQuotes",      "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000002", "scope": { "type": "live" } } }
```

**2. Data plane, `/maker/data`.** Send `GetMmSummary`. Owner pushes (`PositionUpdated`, `TradeExecuted`, `MakerCapsUpdated`) start after it is answered. Do not poll it.

```json
{ "type": "GetMmSummary", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000003" } }
```

**3. Position book.** Apply `MmSummary.positions` by revision. The summary holds the newest 500 positions. If `positions_has_more` is `true`, page `GetMakerPositions` to the end.

**4. Money.** Book net premium from `total_premium` on the position rows. The maker's cost is the gross premium; the fee is that cost minus `total_premium`.

**5. Unfinished orders.** For each local order that has no terminal outcome and is not in the Live set, send `GetOrderStatus` and wait for the response before the next one. Stop at the first rejection (`RequestError`), fix the cause, and resume from that order.

```json
{ "type": "GetOrderStatus", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000021", "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a" } }
```

`state` is `pending`, `confirmed` with `position_pda`, or `unknown`. An empty Live response, a timeout and `unknown` do not prove that an order did not execute. Keep it as an obligation.

**6. Quote again.** Do not replay pre-disconnect quotes. Sign new ones with a fresh `valid_until`, `nonce` and `order_id`.

### Paging

Position, trade and quote-history reads are paged. `limit` and `cursor` are required. A limit outside the range is rejected with `invalid_limit`; it is never defaulted or clamped.

| Request | `limit` | Cursor `at` | Cursor `id` |
|---|---|---|---|
| `GetMakerPositions` | `1..=500` | `created_at` | `pda` |
| `GetMyQuotes` with `scope.type = "history"` | `1..=1000` | `created_at` | `order_id` |
| `GetMyTrades` | `1..=200` | `confirmed_at` | trade `id` |

Pages run newest first. Send the first page with `{ "type": "start" }`. Send the next page with the last row's keys, and stop when `has_more` is `false`:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000011",
    "market": { "type": "all" },
    "underlying_mint": { "type": "all" },
    "status": [],
    "min_expiry_ts": { "type": "unbounded" },
    "limit": 500,
    "cursor": { "type": "after", "at": 1710000042, "id": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP" }
  }
}
```

`status: []` matches every status, and `{ "type": "all" }` matches every market or mint. `GetMyQuotes` with `scope: { "type": "live" }` is complete and unpaged; never substitute History for recovery. See [`../reference/maker-api.md`](../reference/maker-api.md#filters-and-pages).

If many maker workers can reconnect at once, add a small random delay before expensive recovery
reads so they do not all hit the data plane in the same second.

## Discovery

Fetch static metadata on `/maker/data` at startup and re-read it when the server sends `ReferenceChanged`. Do not refresh these requests on a fixed interval.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000009" } }
{ "type": "GetExpiries",          "data": { "request_id": "a1b2c3d4-0000-4000-8000-00000000000b", "underlying_mint": { "type": "all" }, "quote_mint": { "type": "all" }, "is_put": { "type": "all" } } }
{ "type": "GetTokens",            "data": { "request_id": "a1b2c3d4-0000-4000-8000-00000000000a" } }
{ "type": "GetMarketsForMaker",   "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000012", "underlying_mints": [], "quote_mints": [], "min_expiry_ts": { "type": "unbounded" }, "max_expiry_ts": { "type": "unbounded" }, "is_put": { "type": "all" } } }
```

Discovery returns tradable markets only (non-finalized, non-disabled, before the pre-expiry trading cutoff). The same data is on HTTP: `GET /api/v1/venue`, `/api/v1/tokens` and `/api/v1/markets` (`active=false` adds closed and finalized markets), with ETag revalidation. See [Reference data](../reference/ws-common.md#reference-data).

## Operational defaults

| Topic | Recommendation |
|---|---|
| Application Ping | Approximately every 30 seconds. Each `Pong` carries an updated `server_time_unix_ms`. |
| Reconnect backoff | Exponential with jitter, e.g. 250 ms initial, 5 s cap, ±20%. |
| `valid_until` margin | `rfq.expires_at + min_quote_validity_seconds` plus a little slack, capped at `market.expiry_ts`, with `min_quote_validity_seconds` from `GET /api/v1/venue`. Values below `now + min_quote_validity_seconds` or after market expiry are rejected; going far beyond the auction plus the settlement buffer only widens the maker's exposure window. |
| Clock skew | Track `offset = server_time − local_time` from `Welcome` and `Pong`. Apply when computing `valid_until`. |
| Quote concurrency | One active quote per `(rfq_id, terms)`. Repricing via `ReplaceQuote`. |
| Rate limits | [Rate limits and admission](../reference/ws-common.md#rate-limits-and-admission). A breach of the message rate closes the connection with `4003`; on `/maker` with `on_disconnect: "cancel_quotes"` the disconnect then removes active and retained non-winning quotes. |
| Message size | `32 KiB` inbound WS message limit. |
| Batch quotes | Prefer one `BatchQuotes` over many single `Quote` messages. |

The server sends WebSocket protocol pings every 30 seconds and closes idle connections after 90 seconds. WebSocket libraries that do not answer protocol pings will disconnect. If a correct client still drops, check proxies or firewalls that close idle TCP connections.
