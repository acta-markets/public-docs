# Acta Maker Quickstart

JSON-layer path for a maker connection: auth, subscribe, quote, handle fills, reconnect. Rust integrations should start with [`maker-rust-sdk.md`](maker-rust-sdk.md). Payload examples are in [`maker-wire-examples.md`](maker-wire-examples.md); onboarding is in [`../reference/sandbox.md`](../reference/sandbox.md); units and envelopes are in [`../reference/ws-common.md`](../reference/ws-common.md); the message catalogue is in [`../reference/maker-api.md`](../reference/maker-api.md).

---

## Connection and authentication

The first message from the client is `Hello`. Version compatibility is semver-based: the server accepts any client `protocol_version` that is `>= min_supported_version`, otherwise it replies with `VersionMismatch` and closes the connection. `features` is an opt-in list. The `quote_expired` feature, when enabled, causes the server to emit explicit `QuoteExpired` events in place of silent expiry. The `cancel_on_disconnect` feature makes the engine cancel all of the session's open quotes when the connection ends; without it, resting quotes stay live while the maker is offline. Check `Welcome.enabled_features` to confirm what the server accepted.

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

A resumed maker auth session restores server-side routing and mint scope. Omitted or null mint fields leave the saved scope unchanged; explicit empty lists clear mint filters. After fresh auth, establish subscriptions again. The managed SDK restores its own desired target after either path, sending both mint lists explicitly, including empty lists.

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "<uuid>",
    "channels": ["rfqs", "chain_events"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

`request_id` (UUID) is required; the server echoes it in `SubscribeAck`.

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. Each mutation returns `SubscriptionUpdated` with the current subscription state.

## Quoting

On `RfqBroadcast`, first verify the [market PDA and terms](../reference/maker-api.md#quote-flow) against your configured Acta program. Then pick a strike from `rfq.strike` or `rfq.order_options`, compute the premium, set `now + 100s <= valid_until <= market.expiry_ts`, build the 182-byte order preimage from [`../reference/maker-api.md`](../reference/maker-api.md) (Quote rules), hash it to `order_id`, sign the 32-byte hash with `quote_signing`, and send `Quote`:

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "...",
    "strike": 160000000000,
    "price": 50000000,
    "valid_until": 1710000350,
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
| `QuoteSelected` | Quote locked; awaiting the taker signature. |
| `QuoteFilled` | Position opened on-chain. Carries `position_pda` and `tx_signature`. |
| `QuoteCancelled` | Terminal. `reason` ∈ {`requested`, `risk_check`, `rfq_accepted`, `maker_disconnected`}. |
| `QuoteExpired` | Emitted only when `quote_expired` was enabled in `Hello`. |
| `RfqAvailableAgain` | Settlement reverted; the auction has reopened. The maker may re-quote with a fresh `order_id`. |
| `RfqClosed` | Terminal RFQ event. Drop per-RFQ state. |

## On-chain responsibilities

Quoting and the lifecycle above are entirely off-chain (WebSocket). Your only on-chain actions as a maker are funding-related:

- `DepositPremium` — deposit program quote balance; **required before quoting** (the fill's premium debit draws from it). `WithdrawPremium` retrieves idle balance.
- `DepositFundsToPosition` — optional, after a fill: fund the settlement leg (`open` → `funded`) to avoid the ITM-unfunded liquidation loss.

You do **not** settle or liquidate. Publishing the settlement price and finalizing markets is operator-side; settlement is keeper-driven; liquidating ITM-unfunded positions is permissionless. Your downside is bounded — fund the settlement leg, or a third party liquidates and fronts the taker payout. See [`../reference/protocol-flow.md`](../reference/protocol-flow.md) for the risk model.

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

Cancel-on-disconnect (COD) applies when the client requests `cancel_on_disconnect` in `Hello.features` and the server includes it in `Welcome.enabled_features`. On disconnect, Core removes the connection's active and retained non-winning quotes; selected/executing obligations survive. Without COD, resting quotes can remain fillable while offline. Resume restores server subscriptions; fresh auth requires restoring them. The server does not replay events missed during the disconnect window. Events already in flight can arrive again after recovery, so process lifecycle events idempotently by `order_id`.

After reauthentication, use `/maker/data` for recovery reads and `/maker` for
quote-plane subscription state:

```json
{ "type": "GetMyQuotes",       "data": { "request_id": "...", "scope": "live" } }
{ "type": "GetActiveRfqs",     "data": { "request_id": "..." } }
{ "type": "GetMakerPositions", "data": { "request_id": "..." } }
{ "type": "GetMyTrades",       "data": { "request_id": "..." } }
```

`GetMyQuotes { scope: "live" }` returns the full unpaged owner set, including retained/selected/executing quotes. `scope: "history"` returns only the paginated DB projection. An empty Live response is not proof of nonexecution for an order whose ACK was lost; use `GetOrderStatus` and retain unresolved obligations. `GetMyTrades` supports keyset pagination via `cursor` and `cursor_id`, and a `market` filter; see [`../reference/maker-api.md`](../reference/maker-api.md).

For MM dashboard bootstrap, send `GetMmSummary` on `/maker/data` after auth or reconnect. Do not poll it; use manual refresh or drift recovery if a full snapshot is needed later.
If many maker workers can reconnect at once, add a small random delay before expensive recovery
reads so they do not all hit the data plane in the same second.

## Discovery

Fetch static metadata on `/maker/data` at startup and refresh it when markets or tokens change. Do not refresh these requests on a fixed interval.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "...", "active_only": true } }
{ "type": "GetExpiries",          "data": { "request_id": "..." } }
{ "type": "GetTokens",            "data": { "request_id": "...", "active_only": true } }
{ "type": "GetMarketsForMaker",   "data": { "request_id": "..." } }
```

`active_only: true` filters to tradable markets (non-finalized, non-disabled, before the pre-expiry trading cutoff); `false` returns settled/expired markets in addition.

## Operational defaults

| Topic | Recommendation |
|---|---|
| Application Ping | Approximately every 30 seconds. Each `Pong` carries an updated `server_time_unix_ms`. |
| Reconnect backoff | Exponential with jitter, e.g. 250 ms initial, 5 s cap, ±20%. |
| `valid_until` margin | `rfq.expires_at + 100s` plus a little slack, capped at `market.expiry_ts`. Values below `now + 100s` or after market expiry are rejected; going far beyond the auction plus the settlement buffer only widens the maker's exposure window. |
| Clock skew | Track `offset = server_time − local_time` from `Welcome` and `Pong`. Apply when computing `valid_until`. |
| Quote concurrency | One active quote per `(rfq_id, strike)`. Repricing via `ReplaceQuote`. |
| Message rate | `30 msg/s` sustained, `60` burst per WebSocket connection. |
| Query rate | `20 query tokens/s` sustained, `40` burst per WebSocket connection. |
| Message size | `32 KiB` inbound WS message limit. |
| Batch quotes | Hard max `50` quote elements; cost is `max(1, quotes.length)` quote tokens. |

The server sends WebSocket protocol pings every 30 seconds and closes idle connections after 90 seconds. WebSocket libraries that do not answer protocol pings will disconnect. If a correct client still drops, check proxies or firewalls that close idle TCP connections.
