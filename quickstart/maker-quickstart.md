# Acta Maker Quickstart

JSON-layer path for a maker connection: auth, subscribe, quote, handle fills, reconnect. Rust integrations should start with [`maker-rust-sdk.md`](maker-rust-sdk.md). Payload examples are in [`maker-wire-examples.md`](maker-wire-examples.md); onboarding is in [`../reference/sandbox.md`](../reference/sandbox.md); units and envelopes are in [`../reference/ws-common.md`](../reference/ws-common.md); the message catalogue is in [`../reference/maker-api.md`](../reference/maker-api.md).

---

## Connection and authentication

The first message from the client is `Hello`. The server requires `protocol_version` to match its expected version exactly; mismatches result in connection closure. `features` is an opt-in list. The `quote_expired` feature, when enabled, causes the server to emit explicit `QuoteExpired` events in place of silent expiry.

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "features": [],
    "client_name": "maker-bot",
    "client_version": "0.1.0"
  }
}
```

The server responds with `Welcome` (`server_time_unix_ms` is for clock sync), then `AuthRequest` with a challenge string. Sign the UTF-8 challenge bytes with `quote_signing`, base58-encode the signature, and reply with `AuthChallenge`:

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

`pubkey` is the `maker_owner` wallet registered on-chain, not the `quote_signing` key. The server looks up the registered signing key from `maker_owner` and verifies against it. Auth must finish within 15 seconds of the challenge; after three failed attempts the server closes the connection.

## Subscription

Subscriptions do not persist across disconnects. The client must reissue `Subscribe` after each successful authentication. A `null` mints filter selects all mints.

```json
{
  "type": "Subscribe",
  "data": {
    "channels": ["rfqs", "chain_events"],
    "underlying_mints": null,
    "quote_mints": null
  }
}
```

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. Each mutation returns `SubscriptionUpdated` with the current subscription state.

## Quoting

On `RfqBroadcast`, pick a strike from `rfq.strike` or `rfq.order_options`, compute the premium, set `valid_until >= now + 310s`, build the 182-byte order preimage from [`../reference/maker-api.md`](../reference/maker-api.md) (Quote rules), hash it to `order_id`, sign the 32-byte hash with `quote_signing`, and send `Quote`:

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "...",
    "strike": 160000000000,
    "price": 50000000,
    "valid_until": 1710000310,
    "nonce": 42,
    "order_id": "0x<64 hex>",
    "signature": "<base58 of ed25519(order_id)>"
  }
}
```

When an RFQ permits multiple strikes, each strike requires a distinct `Quote` (or several can be batched in a single `BatchQuotes`). Each quote requires its own `order_id` and `nonce`. A new quote on the same `(rfq_id, strike)` replaces the prior quote for that pair.

Use `ReplaceQuote` for repricing: the swap is atomic and single-RTT. `CancelQuote` followed by `Quote` introduces a gap in the book during the cancellation window and incurs an additional round-trip.

`is_taker_buy` in the order-id preimage is fixed at `0`; the taker is always the option writer. Submitting `1` causes preimage validation to fail. Implementations whose preimage builders default this field to `true` must override it explicitly.

`QuoteRejected` carries `reason`. Fix the cause before retrying; the same payload will fail again.

## Lifecycle events

Lifecycle events are keyed by `order_id`. `RfqClosed` is the terminal event for an RFQ; `QuoteFilled` is a fill-details event and is followed by `RfqClosed`.

| Event | Description |
|---|---|
| `QuoteAcknowledged` | Server accepted the quote. On a replace, includes `replaced_order_id`. |
| `QuoteRejected` | Server refused the quote; see `reason`. |
| `QuoteBestStatus` | Quote is currently the best in the book. |
| `QuoteOutbid` | Quote has been displaced by a better one. |
| `QuoteRefreshRequested` | Settlement-buffer cutoff is approaching; resubmit with `valid_until ≥ min_valid_until` before the cutoff. |
| `QuoteSelected` | Taker selected this quote; sponsored-tx settlement is in flight. |
| `QuoteFilled` | Position opened on-chain. Carries `position_pda` and `tx_signature`. |
| `QuoteCancelled` | Terminal. `reason` ∈ {`requested`, `risk_check`, `rfq_accepted`}. |
| `QuoteExpired` | Emitted only when `quote_expired` was enabled in `Hello`. |
| `RfqAvailableAgain` | Settlement reverted; the auction has reopened. The maker may re-quote with a fresh `order_id`. |
| `RfqClosed` | Terminal RFQ event. Drop per-RFQ state. |

## Indicative pricing

If the account is enrolled in pre-trade pricing, the server periodically emits `IndicativePricesRequest` for reference prices to be displayed in the taker UI. Indicative quotes are non-binding and operate under a tighter latency budget than auction quotes. The response is correlated by `request_id`:

```json
{
  "type": "IndicativePricesResponse",
  "data": {
    "request_id": "<from request>",
    "market": "<market PDA, base58>",
    "position_type": "covered_call",
    "prices": [
      { "strike": 150000000000, "price": 45000000 },
      { "strike": 160000000000, "price": 50000000 }
    ]
  }
}
```

## Reconnection

Quotes survive disconnects until `valid_until` or `RfqClosed`. Subscriptions do not; resend them after auth. The server does not replay events missed during the disconnect window. Events already in flight can arrive again after recovery, so process lifecycle events idempotently by `order_id`.

After reauthentication, the session view is rehydrated by issuing the relevant `Get*` queries:

```json
{ "type": "GetMyQuotes",       "data": { "request_id": "...", "active_only": true } }
{ "type": "GetActiveRfqs",     "data": { "request_id": "..." } }
{ "type": "GetMakerPositions", "data": { "request_id": "..." } }
{ "type": "GetMyTrades",       "data": { "request_id": "..." } }
```

`GetMyQuotes` with `active_only: true` returns live quotes. With `active_only: false`, the backend also appends historical quotes from DB; pass `limit` to cap the historical slice. `GetMyTrades` supports keyset pagination via `cursor` and `cursor_id`, and a `market` filter; see [`../reference/maker-api.md`](../reference/maker-api.md).

## Discovery

Fetch static metadata at startup and refresh it when markets or tokens change:

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "...", "active_only": true } }
{ "type": "GetExpiries",          "data": { "request_id": "..." } }
{ "type": "GetTokens",            "data": { "request_id": "...", "active_only": true } }
{ "type": "GetMarketsForMaker",   "data": { "request_id": "..." } }
```

`active_only: true` filters to non-finalized markets with `expiry_ts > now`; `false` returns settled markets in addition.

## Operational defaults

| Topic | Recommendation |
|---|---|
| Application Ping | Approximately every 30 seconds. Each `Pong` carries an updated `server_time_unix_ms`. |
| Reconnect backoff | Exponential with jitter, e.g. 250 ms initial, 5 s cap, ±20%. |
| `valid_until` margin | `now + 320..360s`. Values below 310s are rejected; values significantly above 360s increase the maker's exposure window without functional benefit. |
| Clock skew | Track `offset = server_time − local_time` from `Welcome` and `Pong`. Apply when computing `valid_until`. |
| Quote concurrency | One active quote per `(rfq_id, strike)`. Repricing via `ReplaceQuote`. |

The server sends WebSocket protocol pings every 30 seconds and closes idle connections after 90 seconds. WebSocket libraries that do not answer protocol pings will disconnect. If a correct client still drops, check proxies or firewalls that close idle TCP connections.
