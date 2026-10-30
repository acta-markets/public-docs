# Acta Taker API Reference

## Endpoint

```
wss://devnet-api.acta.markets/taker
wss://beta-api.acta.markets/taker
```

Placeholders such as `MarketPdaBase58` stand in for real base58 values in the examples. UUIDs, order IDs (64 hex characters, optional `0x`) and timestamps must be real values.

## Wire rules

The protocol version is `1.0.0`. The wire is closed:

- Every field is present on every frame. A fact that is absent is an explicit `null`, for example `settlement_price: null` on a market that is not settled.
- State-dependent shapes are tagged unions (`terms`, `state`, `settlement`, `role`, query filters, cursors).
- A client sends all required fields. The server rejects unknown message types, unknown enum values and missing fields.
- Prices and strikes are `u64` with a `1e9` scale. Quantities are underlying atoms. Premiums and collateral are quote atoms. Timestamps are Unix seconds unless the field name ends in `_ms`.

Envelope, units and error codes: [WS common conventions](ws-common.md).

## Connection flow

```
Connect -> Hello -> Welcome -> Auth -> AuthSuccess -> Snapshot -> Subscribe -> ...
```

Auth is one of:
- **Resume** (returning user): `ResumeAuth { session_id }` -> `AuthSuccess`
- **Fresh sign**: `StartAuth { pubkey }` -> `AuthRequest` -> `AuthChallenge` -> `AuthSuccess`

### Hello and Welcome

`Hello` must be the first client message. The server waits `5000ms` for it. A client that sends another message first gets `hello_required`; a missing `Hello` gets `hello_timeout` and close `4005`.

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "on_disconnect": "keep_quotes",
    "client_name": "my-app",
    "client_version": "0.1.0"
  }
}
```

| Field | Rule |
|---|---|
| `protocol_version` | Must be exactly `1.0.0`. |
| `on_disconnect` | `cancel_quotes` or `keep_quotes`. Required on every connection. It sets how a maker's quotes are handled on disconnect. |
| `client_name`, `client_version` | Required strings. |

`Hello` accepts no other fields. A second `Hello` gets `hello_already_sent`. Any other `protocol_version` gets an `Error` with `unsupported_protocol_version`, then the server closes the connection with code `4006`. There is no feature negotiation.

```json
{
  "type": "Welcome",
  "data": {
    "protocol_version": "1.0.0",
    "server_version": "0.1.0",
    "min_supported_version": "1.0.0",
    "server_time_unix_ms": 1710000000000,
    "min_quote_validity_seconds": 60,
    "settlement_buffer_seconds": 50
  }
}
```

`min_quote_validity_seconds` and `settlement_buffer_seconds` are the venue's quote window, also served by `GET /api/v1/venue`.

### Connection policy defaults

- `auth_deadline`: `15s` after the auth challenge was issued
- `idle_timeout`: `90s`
- server WebSocket ping interval: `30s`
- max consecutive parse errors before close: `3`
- max message size: `32 KiB`
- max connections per IP: `50`

Rate limits: [Rate limits and admission](ws-common.md#rate-limits-and-admission). Retry with exponential backoff and jitter.

### Taker auth lifecycle

- Taker auth is lazy: `StartAuth` and `ResumeAuth` can be sent right after `Welcome` or later.
- `Snapshot` is sent after `AuthSuccess`.

### ResumeAuth (session resume)

```json
{ "type": "ResumeAuth", "data": { "session_id": "6d1f0c2a-9b7e-4a35-8c41-2e5f7a9b0d13" } }
```

The server answers with `AuthSuccess` if the session is valid, or `AuthError { reason: "session_expired", message }` if it is invalid, expired or revoked. On `AuthError`, fall back to `StartAuth`.

A failed `ResumeAuth` counts as an auth attempt. The default allows three failed attempts; the fourth gets `too_many_auth_attempts` and the connection closes.

**Session lifetime:**
- Sessions have a default TTL of 24 hours and survive WebSocket disconnects.
- `expires_at` extends on activity (sliding window) up to an absolute maximum of 7 days.
- Persist `session_id` and `expires_at`, and use `ResumeAuth` on reconnect while `now < expires_at`.

### StartAuth / AuthRequest / AuthChallenge (fresh sign)

```json
{ "type": "StartAuth", "data": { "pubkey": "WalletPubkeyBase58" } }
```

```json
{
  "type": "AuthRequest",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: WalletPubkeyBase58\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2026-09-15T00:00:00Z\n"
  }
}
```

Validate the challenge ([What to sign](ws-common.md#what-to-sign)), sign its UTF-8 bytes with the wallet key, and reply:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "<AuthRequest.challenge>",
    "signature": "<base58 ed25519 signature>",
    "pubkey": "WalletPubkeyBase58"
  }
}
```

### AuthSuccess

```json
{
  "type": "AuthSuccess",
  "data": {
    "session_id": "6d1f0c2a-9b7e-4a35-8c41-2e5f7a9b0d13",
    "expires_at": 1710086400,
    "maker_pda": null
  }
}
```

`expires_at` is the resume credential deadline in Unix seconds, not the lifetime of the authenticated socket. `maker_pda` is `null` for a taker. Persist `session_id` and `expires_at`.

### AuthError

```json
{
  "type": "AuthError",
  "data": {
    "reason": "session_expired",
    "message": "session expired"
  }
}
```

`reason` is a code from the [WS error code catalog](ws-common.md#ws-error-codes). `message` is a required string.

### Subscribe

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "4e9a2b17-8c6d-4f31-a5e0-7d3b1c9f2a64",
    "channels": ["chain_events", "trades"],
    "products": ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

Every field is required.

- `channels`: `stats`, `rfqs`, `trades`, `chain_events`, `markets`, `vaults`. A taker needs none of them for its own RFQ, quote and order events, which arrive on the session. See [Supported channels](ws-common.md#supported-channels).
- `products`: a non-empty list of unique product names. The server applies it to maker and vault sessions only; it does not filter what a taker receives.
- `underlying_mints`, `quote_mints`: mint filters. `[]` means every mint. `Subscribe` replaces both scopes together.

The server answers with `SubscribeAck { request_id, subscribed }`, where `subscribed` lists the channels this request added. Subscriptions do not persist across WebSocket disconnects; resend `Subscribe` after reconnect.

### Snapshot (server -> taker)

```json
{
  "type": "Snapshot",
  "data": {
    "markets": [
      {
        "pda": "MarketPdaBase58",
        "underlying": "So11111111111111111111111111111111111111112",
        "quote": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "expiry_ts": 1710600000,
        "is_put": false,
        "products": ["covered_call", "call_spread"],
        "status": "active",
        "settlement_price": null
      }
    ]
  }
}
```

`Snapshot.markets` lists the active markets as [`MarketInfo`](ws-common.md#marketinfo). `products` lists only the pair's products that fit the market's side: a put market (`is_put: true`) lists a subset of `cash_secured_put`, `put_spread` and `cash_put`; a call market lists a subset of `covered_call` and `call_spread`. A market with no product on its side is not listed. Use `GetMarketDescriptors` for the descriptor with `program_id`, `collateral_mint`, `settlement_mint` and oracle PDAs.

---

## Message index

### Client -> Server

- `Hello`, `ResumeAuth`, `StartAuth`, `AuthChallenge`, `Logout`
- `Subscribe`, `Unsubscribe`, `Ping`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `RfqRequest`, `AcceptQuote`, `CancelRfq`, `SubmitSignedSponsoredTx`
- `GetIndicativePrices`, `GetEarnSummary`, `GetTokenMarketsInfo`
- `GetMarkets`, `GetMarketDescriptors`, `GetExpiries`, `GetTokens`
- `GetPositions`, `GetMyActiveRfqs`, `GetOrderStatus`, `GetSubscriptions`
- `GetTokenCaps`
- `RedeemInvite`, `ClaimReferralCode`, `GetMyReferralInfo`

### Server -> Taker (direct)

- `Welcome`, `LogoutSuccess`
- `AuthRequest`, `AuthSuccess`, `AuthError`
- `Snapshot`, `ReferenceChanged`
- `CommandAccepted`
- `RfqCreated`, `QuoteReceived`, `QuotesUpdate`
- `SponsoredTxToSign`, `OrderAccepted`, `OrderSubmitted`, `OrderConfirmed`, `OrderFailed`
- `RfqAvailableAgain`, `RfqClosed`
- `PositionUpdated`
- `IndicativePrices`, `EarnSummary`
- `RequireInvite`, `InviteRedeemed`, `ReferralCodeClaimed`, `MyReferralInfo`
- `SubscribeAck`, `UnsubscribeAck`, `SubscriptionUpdated`
- `Error`, `RequestError`, `Pong`

### Server -> Taker (responses / query results)

- `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`
- `Positions`, `MyActiveRfqs`, `OrderStatus`, `Subscriptions`
- `TokenCaps`, `TokenMarketsInfo`

### Server -> Taker (broadcast if subscribed)

- `TradeExecuted`, `StatsUpdate`, `ChainEvent`
- `MarketCreated`, `MarketFinalized`
- `VaultCatalogChanged`, `VaultProjectionAdvanced` (`vaults` channel; see [ws-common](ws-common.md#vaultcatalogchanged))

---

## Products and terms

`terms` is a tagged union on `type`. It names the product and carries its strikes. Strikes are `1e9`-scaled.

| `terms.type` | Fields | Market side | Settlement |
|---|---|---|---|
| `covered_call` | `strike` | call market (`is_put: false`) | physical |
| `cash_secured_put` | `strike` | put market (`is_put: true`) | physical |
| `call_spread` | `lower_strike`, `upper_strike` | call market | cash, quote mint |
| `put_spread` | `lower_strike`, `upper_strike` | put market | cash, quote mint |
| `cash_put` | `strike` | put market | cash, quote mint |

```json
{ "type": "covered_call", "strike": 136000000000 }
{ "type": "call_spread", "lower_strike": 150000000000, "upper_strike": 160000000000 }
{ "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 }
{ "type": "cash_put", "strike": 136000000000 }
```

`cash_put` executes on chain as a put spread with legs `(0, strike)`. Its width is `strike`.

`side` is the taker's side. Takers sell: send `"sell"`. `"buy"` is rejected with `taker_buy_unsupported`.

Admission rules for `RfqRequest`:

- The product's side must match the market: a put product on a call market (or the reverse) fails with the typed error `InvalidPositionType`.
- The product must be enabled for the pair, otherwise `product_not_allowed`. `MarketInfo.products` lists the enabled products.
- Spreads need `0 < lower_strike < upper_strike`, otherwise `spread_invalid_strikes`. A put with a zero lower leg is a `cash_put`, which needs `strike > 0`.
- Both spread strikes, and the cash put strike, must be multiples of the underlying's `strike_tick` (a `TokenInfo` field, `1e9`-scaled); otherwise `spread_tick_violation`. A missing tick fails with `missing_tick_for_underlying_mint`.
- `covered_call` and `cash_secured_put` strikes must give a positive collateral amount that fits on chain (`invalid_strike_notional`).
- `quantity` must satisfy the underlying's size rule (`invalid_quantity_size_rule`).

### Economics of spreads and cash puts

Spreads and cash puts are cash-settled in the quote mint. Their lifecycle is Open -> Settled. They are never funded or liquidated.

- **Width** `W` is `upper_strike - lower_strike` for a spread and `strike` for a cash put.
- **Collateral** `C = ceil(W * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`, in quote atoms. The taker posts `C`.
- **Gross price** `G` is per underlying unit, `1e9`-scaled, and satisfies `0 < G < W`. A maker quote outside that range is rejected with `spread_premium_invalid`, so a taker never receives one.
- **Protocol fee** is `min(premium_fee, volume_fee)`. With `quote(x) = floor(x * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`, `premium_fee` is `quote(G) - quote(G - floor(G * protocol_fee_bps_premium / 10000))`. `volume_fee` is `floor(quote(exposure) * protocol_fee_bps_volume / 10000)`, where `exposure` is `W` for spreads and cash puts and `strike` for physical products. Read both rates from `GET /api/v1/venue`.
- **Net premium** is gross premium minus the fee, in quote atoms. The taker receives it at open. It is the `total_premium` of the position row.
- **At expiry** the maker receives `ceil(min(intrinsic, W) * quantity ...)` in quote atoms, capped at `C`. The taker receives the rest of `C`. `intrinsic` is `max(settlement_price - lower_strike, 0)` for a call spread, `max(upper_strike - settlement_price, 0)` for a put spread, and `max(strike - settlement_price, 0)` for a cash put.

Example: put spread `(130, 140)`, 5 SOL (`quantity = 5000000000`), SOL has 9 decimals and USDC has 6. The 5 SOL in these examples may exceed a deployment's `max_size`; read `size_rule` first.

| Quantity | Value |
|---|---|
| `W` | `10` (`10000000000`) |
| `C` | `50000000` (50 USDC) |
| `G` | `1.2` (`1200000000`), gross premium `6000000` |
| Fee at 50 bps premium, 10 bps volume | `min(30000, 50000) = 30000` |
| Net premium (`total_premium`) | `5970000` |
| Settlement price `133` | maker `35000000`, taker `15000000` |
| Settlement price `140` or above | maker `0`, taker `50000000` |
| Settlement price `130` or below | maker `50000000`, taker `0` |

The fee bps in the example are illustrative. `QuoteReceived.net_price` is `null` for spreads and cash puts; compute the net premium from the formulas above or read `total_premium` from the position row. `QuoteFilled` carries no fee or net fields.

---

## RFQ flow

### RfqRequest (taker -> server)

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "market": "MarketPdaBase58",
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "quantity": 5000000000,
    "timeout_seconds": 30
  }
}
```

`request_id` correlates the reply and is the idempotency key: repeating it returns the RFQ it created. The reply is exactly one `CommandAccepted { request_id }` or `RequestError { request_id, error, delivery }`; see [correlation semantics](ws-common.md#correlation-semantics). `quantity` is in underlying atoms for every product. `timeout_seconds` must be within the venue range `min_rfq_timeout_seconds` to `max_rfq_timeout_seconds`; a value outside it is rejected with `invalid_timeout`.

A wallet taker's collateral is checked before broadcast. The wallet's token account must hold the full amount, or the RFQ is rejected with `insufficient_collateral`. If the server cannot read the balance, the RFQ goes ahead and an underfunded fill fails on chain:

- covered call: `quantity` of the underlying;
- cash-secured put: the strike notional at the highest strike in `terms` and `order_options`;
- spread or cash put: `C`.

The premium does not count, although it reaches the wallet in the same transaction, before the collateral leaves. For a covered call on wSOL, native SOL counts too: the wSOL balance plus the wallet's SOL, less `1200000` lamports. The transaction wraps the shortfall. The keeper pays the transaction fee and the rent of new accounts.

### CommandAccepted (server -> client)

```json
{ "type": "CommandAccepted", "data": { "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42" } }
```

The server accepted `RfqRequest`, `AcceptQuote` or `SubmitSignedSponsoredTx`. It does not mean the order executed; follow the lifecycle pushes.

### RfqCreated (server -> taker)

```json
{
  "type": "RfqCreated",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 1,
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "expires_at": 1710000030,
    "created_at": 1710000000,
    "order_options": [
      { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 }
    ]
  }
}
```

`terms` and `side` echo the RFQ. `request_id` is `null` for a notification not caused by this session's request. `order_options` is the list of `terms` that makers may quote, the same list makers receive in `RfqBroadcast.order_options`. For a spread or cash put it contains exactly the requested `terms`. For `covered_call` and `cash_secured_put` it is a server-computed strike ladder of the same product; the number of strikes depends on the market and the index price.

### QuoteReceived (server -> taker)

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "maker": "MakerOwnerPubkeyBase58",
    "price": 1200000000,
    "valid_until": 1710000350,
    "nonce": 42,
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "net_price": null
  }
}
```

- `terms` is one of the RFQ's `order_options`.
- `price` is the gross price from the maker. It is hash-bound through `order_id`; use it for `AcceptQuote` and all order operations.
- `net_price` is `Price | null`, for display only. It is `null` for spreads and cash puts and when the server has no fee configuration. The contract computes the authoritative fee at open as `min(premium_fee, volume_fee)` after token-decimal scaling.

### QuotesUpdate (server -> taker)

The server sends the current executable quote snapshot to the RFQ owner whenever the quote set changes. An empty `quotes` array means no executable quote remains. Each element has the `QuoteReceived` shape.

```json
{
  "type": "QuotesUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "quotes": [
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "terms": { "type": "covered_call", "strike": 160000000000 },
        "side": "sell",
        "maker": "MakerOwnerPubkeyBase58",
        "price": 50000000,
        "valid_until": 1710000350,
        "nonce": 42,
        "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
        "net_price": 49750000
      }
    ]
  }
}
```

### AcceptQuote (taker -> server)

```json
{
  "type": "AcceptQuote",
  "data": {
    "request_id": "0c9e1d52-6c0b-4f7e-b1a9-7d8f3c2e5a10",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "maker": "MakerOwnerPubkeyBase58",
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111"
  }
}
```

### CancelRfq (taker -> server)

```json
{
  "type": "CancelRfq",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "request_id": "b3d5f7a9-1c2e-4f40-8b6d-9e0a2c4e6f81"
  }
}
```

Success has no `CommandAccepted`: the RFQ, active or awaiting your signature, closes with `RfqClosed { reason: "taker_cancelled" }`. Cancelling an RFQ of yours that is already closed gets no reply. A failure is a `RequestError` with the same `request_id`: `RfqNotFound`, or `Generic` `order_already_submitted` once your signed transaction is queued (resolve it with `GetOrderStatus`).

### SponsoredTxToSign (server -> taker)

```json
{
  "type": "SponsoredTxToSign",
  "data": {
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "tx_base64": "AQABAg...",
    "last_valid_block_height": 312456789,
    "signature_deadline": 1710000040
  }
}
```

`tx_base64` is a v0 `VersionedTransaction` with two signature slots. The keeper (fee payer) is slot 0 and the taker is slot 1. The keeper fills slot 0 after you submit. `last_valid_block_height` is the block height the transaction's blockhash is valid through. `signature_deadline` is Unix seconds. Byte-level signing: [Taker quickstart](../quickstart/taker-quickstart.md#sponsored-transaction-raw-signing).

### SubmitSignedSponsoredTx (taker -> server)

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": {
    "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f",
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "tx_base64": "AQABAg..."
  }
}
```

A failure is a `RequestError` whose `delivery` is `not_forwarded` or `unknown`. Unless every submission you made for the order returned `not_forwarded`, resolve it with `GetOrderStatus` before signing anything new. See [RequestError](ws-common.md#requesterror-request-correlated-errors).

### OrderAccepted (server -> taker)

Sent when `SubmitSignedSponsoredTx` is accepted and queued for execution. It is not chain confirmation. An exact `AcceptQuote` retry on a locked order may also receive this liveness ACK while its signing payload is being built, or after enqueueing. The initial `AcceptQuote` locks the quote and starts building `SponsoredTxToSign`.

```json
{
  "type": "OrderAccepted",
  "data": {
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "order_version": 1
  }
}
```

### OrderStatus (server -> taker)

`GetOrderStatus` is an owner-scoped execution lookup for authenticated takers and makers. It returns the strongest execution evidence currently available for `order_id`:

```json
{
  "type": "OrderStatus",
  "data": {
    "request_id": "e5a7c9e1-2b4d-4c68-9f1a-3c5e7a9b1d48",
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "state": { "type": "confirmed", "position_pda": "PositionPdaBase58" }
  }
}
```

| `state` | Meaning |
|---|---|
| `{ "type": "pending" }` | The venue still has a pending execution obligation. |
| `{ "type": "confirmed", "position_pda": "..." }` | Execution is confirmed; this is the resulting position. |
| `{ "type": "unknown" }` | Available evidence cannot establish the outcome. This is not proof of nonexecution. |

The response has no `order_version`, `rfq_id` or `tx_signature`. Lookup failures return a correlated `RequestError`. Neither an empty position list nor `unknown` authorizes replaying the order.

`order_version` is on lifecycle pushes: accepted `1`, submitted `2`, failed `3`, confirmed `4`. A later chain confirmation can supersede a local failure.

### OrderSubmitted (server -> taker)

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "tx_signature": "5eyk...base58sig",
    "order_version": 2
  }
}
```

### OrderConfirmed (server -> taker)

```json
{
  "type": "OrderConfirmed",
  "data": {
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "position_pda": "PositionPdaBase58",
    "order_version": 4
  }
}
```

### OrderFailed (server -> taker)

```json
{
  "type": "OrderFailed",
  "data": {
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "reason": "transaction simulation failed",
    "order_version": 3
  }
}
```

`reason` is a free-form diagnostic string, not an enum. Log it and show it; never branch on an exact match. Observed shapes include `on_chain: <program error>`, `blockhash_expired_before_send`, `sponsored transaction rejected: <detail>`, `order_expired` and `order_failed`.

No `OrderFailed` reason authorizes a client-side replay. Treat every one as non-retryable and reconcile execution instead.

The keeper retries retryable submission failures within a five-attempt budget. Exhausting it does not prove nonexecution or guarantee a WebSocket `OrderFailed` or `RfqAvailableAgain`. An uncertain keeper failure leaves the order queued; only a locally proven-unforwarded failure can reopen or close the RFQ. Follow `RfqAvailableAgain` and `RfqClosed`, and query `GetOrderStatus` for unresolved execution. Rollback discards the winner; refresh the available quotes instead of replaying that order.

### RfqAvailableAgain (server -> taker)

```json
{
  "type": "RfqAvailableAgain",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 3,
    "reason": "tx_failed",
    "available_again_at": 1710000015
  }
}
```

`reason` is `signature_timeout`, `tx_failed` or `tx_build_failed`.

### RfqClosed (server -> taker)

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "filled",
    "your_quote": null,
    "winner": {
      "maker": "MakerOwnerPubkeyBase58",
      "price": 1200000000,
      "tx_signature": "5eyk...base58sig"
    },
    "closed_at": 1710000012
  }
}
```

`reason` is `expired`, `taker_cancelled`, `filled`, `market_expired` or `ladder_timeout`. `your_quote` is `null` or `{ order_id, status, price }`. `winner` is `null` or `{ maker, price, tx_signature }`, and `tx_signature` is itself `null` when the signature is not known.

### Terminal semantics: OrderConfirmed vs RfqClosed

- `OrderConfirmed` is the order-level confirmation: the selected order is confirmed on chain. It carries `position_pda` and `order_version`. `tx_signature` is delivered earlier, on `OrderSubmitted`.
- `RfqClosed` is the terminal RFQ event. Treat the RFQ as closed for further quoting and accept actions.
- On a fill, takers receive `OrderConfirmed` followed by `RfqClosed` in close succession.

---

## Positions

### GetPositions (taker -> server)

Every filter is required. `[]` for `status` and `unbounded` for `min_expiry_ts` select everything.

```json
{
  "type": "GetPositions",
  "data": {
    "request_id": "f1b3d5e7-9a2c-4d84-8e6a-4b6c8d0e2f59",
    "market": { "type": "all" },
    "underlying_mint": { "type": "all" },
    "status": [],
    "min_expiry_ts": { "type": "unbounded" }
  }
}
```

| Field | Values |
|---|---|
| `market`, `underlying_mint` | `{ "type": "all" }` or `{ "type": "only", "value": "<base58>" }` |
| `status` | list of `open`, `funded`, `liquidated`, `settled`; `[]` means all statuses |
| `min_expiry_ts` | `{ "type": "unbounded" }` or `{ "type": "bounded", "value": <unix seconds> }`; positions that expire before `value` are excluded |

`GetPositions` has no `limit` or `cursor` field. Pages for the maker queries (`GetMakerPositions`, `GetMyQuotes`, `GetMyTrades`) are described in the [Maker API](maker-api.md).

`GetPositions.price` is the gross price the maker signed, taken from the on-chain event that opened the position. It is never derived from the net premium. If the server has not verified that event for a position, or later finds a gap in its verified chain history up to it, the whole read fails with a `RequestError` (`Generic` code `temporarily_unavailable`). Retry with backoff; it clears once the server has reconciled the chain history.

### Positions payload

`Positions` returns `PositionInfo[]`. Every field is present. `state` is flattened into the row; `settlement` is present only when `state` is `settled`.

Open put spread:

```json
{
  "type": "Positions",
  "data": {
    "request_id": "f1b3d5e7-9a2c-4d84-8e6a-4b6c8d0e2f59",
    "positions": [
      {
        "revision": { "slot": 312456800, "transaction_index": 14, "instruction_index": 2 },
        "pda": "PositionPdaBase58",
        "market": "MarketPdaBase58",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_decimals": 9,
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_decimals": 6,
        "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
        "side": "sell",
        "state": "open",
        "quantity": 5000000000,
        "price": 1200000000,
        "total_premium": 5970000,
        "created_at": 1710000012,
        "expiry_ts": 1710600000
      }
    ]
  }
}
```

Settled cash position (settlement price `133`), the same row after settlement:

```json
{
  "revision": { "slot": 312999000, "transaction_index": 3, "instruction_index": 1 },
  "pda": "PositionPdaBase58",
  "market": "MarketPdaBase58",
  "underlying_mint": "So11111111111111111111111111111111111111112",
  "underlying_decimals": 9,
  "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "quote_decimals": 6,
  "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
  "side": "sell",
  "state": "settled",
  "settlement": {
    "type": "cash",
    "settlement_price": 133000000000,
    "maker_payoff": 35000000,
    "taker_return": 15000000
  },
  "quantity": 5000000000,
  "price": 1200000000,
  "total_premium": 5970000,
  "created_at": 1710000012,
  "expiry_ts": 1710600000
}
```

A settled physical position carries `"settlement": { "type": "physical", "settlement_price": 150000000000, "is_otm": true }`.

Field semantics:
- `state`: `open`, `funded`, `liquidated` or `settled`. Only `settled` carries `settlement`.
- `settlement.type`: `physical` or `cash`. Cash amounts (`maker_payoff`, `taker_return`) are quote atoms. `settlement_price` is `1e9`-scaled.
- `price`: gross price per underlying unit, `1e9`-scaled.
- `total_premium`: net premium in quote atoms after the protocol fee. It is the authoritative money figure.
- `underlying_decimals`, `quote_decimals`: token decimals for converting `quantity` and `total_premium`.
- `revision`: the last applied lifecycle event in chain order. See below.
- All amounts are unsigned `u64`.

### PositionUpdated (server -> taker)

The server pushes `PositionUpdated` to the session of the position's taker (and maker) whenever the position changes. It needs no subscription.

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": { "slot": 312999000, "transaction_index": 3, "instruction_index": 1 },
      "pda": "PositionPdaBase58",
      "market": "MarketPdaBase58",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_decimals": 6,
      "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
      "side": "sell",
      "state": "settled",
      "settlement": {
        "type": "cash",
        "settlement_price": 133000000000,
        "maker_payoff": 35000000,
        "taker_return": 15000000
      },
      "quantity": 5000000000,
      "price": 1200000000,
      "total_premium": 5970000,
      "created_at": 1710000012,
      "expiry_ts": 1710600000
    },
    "update_type": "settled"
  }
}
```

`update_type` is `created`, `funded`, `liquidated` or `settled`. Spreads and cash puts only produce `created` and `settled`. The row is complete, not a delta.

Keep positions by `pda`. Apply a `PositionInfo` from any source (`GetPositions`, `PositionUpdated`) only if its `revision` is newer than the stored one. Compare `(slot, transaction_index, instruction_index)` lexicographically. Equal revisions mean equal state. Updates for one position coalesce to the newest revision, so the stream is not a log of every transition. After a reconnect, re-read with `GetPositions`.

### MyActiveRfqs payload

```json
{
  "type": "MyActiveRfqs",
  "data": {
    "request_id": "d2f4a6c8-7e9b-4b13-8d5f-0a2c4e6a8b37",
    "rfqs": [
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "market": "MarketPdaBase58",
        "terms": { "type": "cash_secured_put", "strike": 136000000000 },
        "side": "sell",
        "quantity": 20000000,
        "expires_at": 1710000030,
        "state": "active",
        "locked_order_id": null,
        "quotes_count": 3,
        "best_price": 50000000
      }
    ]
  }
}
```

`state` is `active`, `pending_signature` or `enqueued`. `locked_order_id` is the order ID when the RFQ is locked on an order (`pending_signature` or `enqueued`), otherwise `null`. `best_price` is `null` when no quote exists. For `covered_call` and `cash_secured_put` it is the fee-adjusted display price; for spreads and cash puts it is the gross price.

### Position lifecycle (taker perspective)

**Covered call and cash-secured put** (physical):

1. The position opens. The taker's collateral is locked and the net premium is paid to the taker.
2. The maker may deposit the settlement asset before market expiry (`funded`).
3. After expiry the market is finalized with an oracle price.
4. If the position is out of the money, the taker gets the collateral back. If it is in the money, the taker receives the maker's settlement asset.
5. If the maker did not fund an in-the-money position, normal settlement fails until a liquidator submits a liquidation transaction. The taker receives the settlement asset when that transaction lands.

**Call spread, put spread and cash put** (cash-settled): the position opens with the taker's collateral `C` locked and the net premium paid to the taker. After the market is finalized the position settles in the quote mint: the maker receives its payoff and the taker receives the rest of `C`. There is no funding step and no liquidation.

See [Protocol Flow](protocol-flow.md) for the settlement scenarios.

---

## Delivery and recovery

Delivery is best-effort, with loss surfaced as a disconnect. Everything a taker receives except `StatsUpdate` is critical: owner-direct pushes (`RfqCreated`, `QuoteReceived`, `QuotesUpdate`, `RfqClosed`, `PositionUpdated`, `TradeExecuted`, and the order lifecycle `OrderAccepted`, `SponsoredTxToSign`, `OrderSubmitted`, `OrderConfirmed`, `OrderFailed`) and subscribed broadcasts (`ChainEvent`, market events). Each is delivered reliably or, if it cannot be delivered under backpressure, dropped, after which the server closes the connection. A lost critical message therefore always surfaces as a disconnect. Only `StatsUpdate` may drop silently.

After `AuthSuccess`, re-read `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus` for unresolved orders. These reads have different authorities: live RFQs come from live server state; positions are a chain-derived database projection; the execution lookup combines pending obligations and execution evidence. They are not one atomic snapshot, and absence from a single response does not prove nonexecution.

A successful `ResumeAuth` completes the handoff before `AuthSuccess`: ownership transfers from the previous physical connection of the same auth session. A fresh credential for the same wallet does not perform that transfer.

| Recovered state | Client action |
|---|---|
| Same credential, same `rfq_id` and `locked_order_id`, `pending_signature`, signature not sent | Repeat the exact `AcceptQuote` to retrieve the existing signing payload. While building it, the server may return `OrderAccepted`. Keep the original signature deadline. |
| `enqueued`, or `SubmitSignedSponsoredTx` already sent | Reconcile execution with `GetOrderStatus`; do not resubmit or sign again automatically. |
| `unknown`, missing RFQ/position, timeout or failed lookup | Keep the outcome unresolved. Missing evidence does not close the obligation. |

Preserve the selected order identity and whether the signature was sent across reconnect. Ignore wallet results that belong to an older connection or selection. An uncorrelated `SignatureTimeout` does not identify the affected RFQ; use the matching `RfqAvailableAgain` or `RfqClosed`, or re-read its state.

There is no stream replay cursor. Reconcile entity versions on lifecycle pushes, and apply owner-scoped reads after reconnect. `OrderConfirmed` establishes execution; `RfqClosed` closes the auction, not every possible unresolved execution record.

---

## Discovery

Taker discovery requests. `request_id` is required on each.

```json
{ "type": "GetMarkets", "data": { "request_id": "uuid" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "uuid" } }
{ "type": "GetExpiries", "data": { "request_id": "uuid", "underlying_mint": { "type": "all" }, "quote_mint": { "type": "all" }, "is_put": { "type": "all" } } }
{ "type": "GetTokens", "data": { "request_id": "uuid" } }
{ "type": "GetMyActiveRfqs", "data": { "request_id": "uuid" } }
{ "type": "GetOrderStatus", "data": { "request_id": "uuid", "order_id": "0x...64chars" } }
{ "type": "GetSubscriptions", "data": { "request_id": "uuid" } }
{ "type": "GetIndicativePrices", "data": { "request_id": "uuid", "market": "MarketPdaBase58", "position_type": "covered_call" } }
{ "type": "GetTokenCaps", "data": { "request_id": "uuid", "include_markets": true } }
{ "type": "GetEarnSummary", "data": { "request_id": "uuid" } }
{ "type": "GetTokenMarketsInfo", "data": { "request_id": "uuid", "underlying_mint": "UnderlyingMintBase58" } }
```

`GetExpiries` filters take `{ "type": "all" }` or `{ "type": "only", "value": ... }` (a base58 mint for the mint filters, a boolean for `is_put`). `GetPositions` is described above.

Responses echo the same `request_id`: `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`, `Positions`, `MyActiveRfqs`, `OrderStatus`, `Subscriptions`, `IndicativePrices`, `TokenCaps`, `EarnSummary`, `TokenMarketsInfo`.

`GetMarkets`, `GetMarketDescriptors`, `GetExpiries` and `GetTokens` answer from the server's [reference snapshot](ws-common.md#reference-data): tradable markets only, meaning non-finalized, non-disabled and before the pre-expiry trading cutoff. Closed and finalized markets are on HTTP `GET /api/v1/markets?active=false`. Re-read on [`ReferenceChanged`](ws-common.md#referencechanged).

```json
{ "type": "Expiries", "data": { "request_id": "uuid", "expiries_ts": [1710600000, 1711200000] } }
```

### Discovery strict policy

- Every underlying of an active market needs a configured size rule. If one is missing, `GetMarketDescriptors` and `GetTokens` fail as a whole with `missing_size_rule_for_underlying_mint`.
- `GetMarketDescriptors` also fails as a whole (`MarketMetadataIncomplete`) when an active market has no oracle PDA. Partial discovery is not supported.

### MarketDescriptors payload

`MarketDescriptors` returns `markets: MarketDescriptorInfo[]`:

```typescript
type MarketDescriptorInfo = {
  products: Product[];             // as MarketInfo.products
  market: {
    chain_id: number;
    program_id: string;
    market_pda: string;
    underlying_mint: string;
    quote_mint: string;
    expiry_ts: number;
    is_put: boolean;
    collateral_mint: string;
    settlement_mint: string;
  };
  underlying_oracle_pda: string;
  quote_oracle_pda: string;
  underlying_decimals: number;
  quote_decimals: number;
  size_rule: {
    min_size: number;
    max_size: number;
    step: number;
  };
  underlying_symbol: string;
  quote_symbol: string;
};
```

`size_rule` semantics (per underlying mint, underlying atoms):
- `min_size <= quantity <= max_size`
- `(quantity - min_size) % step == 0`
- if the rule for the market's underlying mint is missing, the server rejects the RFQ with `missing_size_rule_for_underlying_mint`

`size_rule` constrains the underlying quantity on the wire for every product, including cash-secured puts, spreads and cash puts, not the quote deposit. Frontends that collect input in quote terms (for example USDC) must convert before sending `RfqRequest`. See [Quantity and collateral by product](ws-common.md#quantity-and-collateral-by-product).

`program_id` is the same on devnet and mainnet. `GET /api/v1/venue` returns the authoritative `program_id`.

### Tokens payload

```jsonc
{ "type": "Tokens", "data": { "request_id": "uuid", "tokens": [ /* TokenInfo */ ] } }
```

`tokens` lists each mint once; see [`TokenInfo`](ws-common.md#tokeninfo). `role` is `{ "type": "quote" }` or `{ "type": "underlying", "size_rule", "strike_tick", "quote_mints" }`. `strike_tick` is `1e9`-scaled, like strikes. The same objects are served by HTTP `GET /api/v1/tokens`.

---

## Dynamic subscription management

Modify subscriptions without replacing the full set. Both mint lists are required; `[]` selects every mint for that dimension.

```json
{ "type": "AddMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"], "quote_mints": [] } }
{ "type": "RemoveMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"], "quote_mints": [] } }
{ "type": "AddChannels", "data": { "request_id": "uuid", "channels": ["trades"] } }
{ "type": "RemoveChannels", "data": { "request_id": "uuid", "channels": ["stats"] } }
{ "type": "Unsubscribe", "data": { "request_id": "uuid", "channels": ["trades"] } }
```

The four `Add*`/`Remove*` messages respond with `SubscriptionUpdated`, which holds the subscription state after the operation. `Unsubscribe` responds with `UnsubscribeAck { request_id, unsubscribed }`. `GetSubscriptions` responds with `Subscriptions`, which has the same fields as `SubscriptionUpdated`.

```json
{
  "type": "SubscriptionUpdated",
  "data": {
    "request_id": "uuid",
    "channels": ["markets", "trades"],
    "underlying_mints": ["MintBase58"],
    "quote_mints": []
  }
}
```

---

## Invite gating (closed mainnet)

During closed mainnet, takers need an invite code to trade. The server sends `RequireInvite` after auth if the taker is not registered.

### RequireInvite (server -> taker)

```json
{ "type": "RequireInvite" }
```

Unit variant, no `data` field. Sent once after `AuthSuccess` if the taker has not redeemed an invite.

### RedeemInvite (taker -> server)

```json
{
  "type": "RedeemInvite",
  "data": {
    "request_id": "c8e2a4f6-3b5d-4a71-9c0e-5f7a9b1d3e26",
    "code": "abc123"
  }
}
```

The authenticated taker session proves wallet ownership; `RedeemInvite` carries no signature.

### InviteRedeemed (server -> taker)

```json
{
  "type": "InviteRedeemed",
  "data": {
    "request_id": "c8e2a4f6-3b5d-4a71-9c0e-5f7a9b1d3e26",
    "referral_code": "mycode"
  }
}
```

After redemption, the taker receives a personal `referral_code` it can share.

### ClaimReferralCode (taker -> server)

```json
{
  "type": "ClaimReferralCode",
  "data": {
    "request_id": "uuid",
    "code": "mypreferred"
  }
}
```

Claims a custom referral code and replaces the auto-assigned one. The reply is `ReferralCodeClaimed`:

```json
{
  "type": "ReferralCodeClaimed",
  "data": {
    "request_id": "uuid",
    "referral_code": "mypreferred"
  }
}
```

### GetMyReferralInfo (taker -> server)

```json
{ "type": "GetMyReferralInfo", "data": { "request_id": "uuid" } }
```

```json
{
  "type": "MyReferralInfo",
  "data": {
    "request_id": "uuid",
    "referral_code": "mycode",
    "status": "active",
    "total_invited": 3,
    "invited_this_period": 1,
    "max_invites_per_period": 5,
    "next_slot_frees_in_seconds": 86400
  }
}
```

### Invite error types

Invite errors are typed `ServerError` variants, delivered as `RequestError` when the request has a `request_id`.

**`InviteRequired`** is returned when an unregistered taker attempts a trading action.

**`Invite`** (`RedeemInvite` errors). The wire shape is `{ "type": "Invite", "data": "invalid_code" }`.

| Value | Meaning |
|-------|---------|
| `invalid_code` | Code does not exist |
| `code_exhausted` | Code has no remaining uses |
| `code_expired` | Code has expired |
| `code_disabled` | Code has been disabled |
| `code_owner_inactive` | Code owner is inactive |
| `code_owner_blacklisted` | Code owner is blacklisted |
| `already_registered` | Taker already registered |
| `internal_error` | Server error |

**`Claim`** (`ClaimReferralCode` errors):

| Value | Meaning |
|-------|---------|
| `not_registered` | Taker not registered (must redeem an invite first) |
| `invalid_format` | Code format invalid |
| `code_taken` | Code already claimed by another user |
| `reserved` | Code is reserved |
| `internal_error` | Server error |

---

## Token caps

Platform-level open interest, notional and spread-liability caps. Use them to show remaining capacity. No authentication required.

### GetTokenCaps (taker -> server)

```json
{ "type": "GetTokenCaps", "data": { "request_id": "uuid", "include_markets": true } }
```

`include_markets` is required. The server ignores it and always returns every configured cap.

### TokenCaps (server -> taker)

```json
{
  "type": "TokenCaps",
  "data": {
    "request_id": "uuid",
    "tokens": [
      {
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "symbol": "SOL",
        "current_oi": 500000000000,
        "max_oi": 1000000000000,
        "utilization": 0.5
      }
    ],
    "markets": [
      {
        "market_id": "MarketPdaBase58",
        "current_oi": 100000000000,
        "max_oi": 250000000000,
        "utilization": 0.4
      }
    ],
    "quotes": [
      {
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "current_notional": 2000000000,
        "max_notional": 10000000000,
        "utilization": 0.2
      }
    ],
    "quote_liabilities": [
      {
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "current_liability": 120000000,
        "policy": { "type": "limited", "max_liability": 1000000000, "headroom": 880000000 }
      }
    ]
  }
}
```

All four arrays are always present and may be empty. Amounts are `u64`: underlying atoms for OI, quote atoms for notional and liability. `quote_liabilities` reports the spread liability (collateral `C`) per quote mint. `policy` is a tagged union: `unconfigured` (counted, no enforced budget), `unlimited`, or `limited` with `max_liability` and `headroom`. See [Capacity limits](caps.md) for the semantics of each cap.

---

## TokenMarketsInfo

`GetTokenMarketsInfo { request_id, underlying_mint }` is available after `Welcome` without authentication. It returns one market-page snapshot:

```json
{
  "type": "TokenMarketsInfo",
  "data": {
    "request_id": "uuid",
    "underlying_symbol": "SOL",
    "underlying_decimals": 9,
    "quote_symbol": "USDC",
    "quote_decimals": 6,
    "size_rule": { "min_size": 100000000, "max_size": 10000000000, "step": 100000000 },
    "reference_price": 150000000000,
    "markets": [{
      "products": ["covered_call", "call_spread"],
      "market_pda": "MarketPdaBase58",
      "expiry_ts": 1710600000,
      "is_put": false,
      "indicatives": [{
        "position_type": "covered_call",
        "updated_at": 1710000000,
        "is_stale": false,
        "strikes": [{ "strike": 160000000000, "best_price": 50000000 }]
      }]
    }]
  }
}
```

`reference_price` is the backend-validated underlying/quote spot, `1e9`-scaled. `best_price` is `Price | null`: a non-binding indicative display premium per underlying unit, `1e9`-scaled, with the quote-mint fee adjustment applied when fee configuration is available. It is an estimate, not the exact on-chain net premium. Size-rule quantities are underlying atoms. Examples illustrate units, not current prices or limits.

The response does not echo `underlying_mint`; keep it with the request ID. Use spot and premiums from the same response for an APR estimate, and suppress the estimate when the indicative is stale or missing. `updated_at` and `is_stale` describe the maker indicative premiums, not the oracle spot. The response has no oracle publish timestamp, confidence or age.

The backend owns Pyth/Hermes access and price validation. A browser does not need a Pyth key. This endpoint supplies a current snapshot, not historical chart data or a binding execution quote.

**Failure behavior:** missing size rules, incomplete metadata or unavailable oracle prices can return an uncorrelated `Error` (including `OraclePriceNotReady`). When there are no active markets for the underlying, the handler sends no response. Apply a request deadline, show unavailable data, and discard stale responses by request ID. Do not read silence as an empty snapshot. This endpoint is an exception to correlated query-error handling.

## Indicative prices

Indicative prices exist for `covered_call` and `cash_secured_put` only. `GetIndicativePrices` with `call_spread`, `put_spread` or `cash_put` returns an uncorrelated `Error` with `spread_invalid_strikes`, because a spread price depends on both strikes the taker selects. A product that is not enabled for the pair returns `product_not_allowed`.

### GetIndicativePrices (taker -> server)

```json
{
  "type": "GetIndicativePrices",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "position_type": "covered_call"
  }
}
```

### IndicativePrices (server -> taker)

```json
{
  "type": "IndicativePrices",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "position_type": "covered_call",
    "updated_at": 1710000000,
    "is_stale": false,
    "strikes": [
      { "strike": 450000000000, "best_price": 5000000 },
      { "strike": 460000000000, "best_price": null }
    ]
  }
}
```

---

## Earn summary

Aggregated earn data per asset: APR range, capacity utilization, nearest market. No authentication required.

### GetEarnSummary (taker -> server)

```json
{ "type": "GetEarnSummary", "data": { "request_id": "uuid" } }
```

### EarnSummary (server -> taker)

```json
{
  "type": "EarnSummary",
  "data": {
    "request_id": "uuid",
    "assets": [
      {
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_symbol": "SOL",
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_symbol": "USDC",
        "position_type": "covered_call",
        "apr": { "type": "available", "min": 5.2, "max": 18.7 },
        "cap_filled_pct": 0.45,
        "cap_total": 100000000000,
        "cap_used": 45000000000,
        "strikes_count": 3,
        "nearest_market_pda": "MarketPdaBase58",
        "markets_count": 2,
        "nearest_expiry_ts": 1710600000
      }
    ],
    "computed_at": 1710000000
  }
}
```

Field semantics:
- `apr`: `{ "type": "unavailable" }` while no strike has a fresh indicative premium, otherwise `{ "type": "available", "min", "max" }`, an annualized percentage rate range across active strikes.
- `cap_filled_pct`: fraction of capacity used, not clamped to 1.0. It may exceed 1.0 after limits are tightened. See [caps](caps.md#monitoring-caps).
- `cap_total`, `cap_used`: underlying atoms.
- `nearest_market_pda`: market with the nearest expiry. Use it to navigate to the market page.
- `nearest_expiry_ts`: Unix seconds of the nearest market expiry.
- `computed_at`: Unix seconds when the summary was computed.

---

## Broadcast messages

Delivered when subscribed to the channel.

```json
{ "type": "TradeExecuted", "data": { "trade": { "id": "uuid", "market": "MarketPdaBase58", "terms": { "type": "covered_call", "strike": 136000000000 }, "side": "sell", "quantity": 5000000000, "price": 50000000, "taker": "TakerPubkeyBase58", "maker": "MakerOwnerPubkeyBase58", "tx_signature": "5eyk...base58sig", "executed_at": 1710000012 } } }
```

`TradeExecuted` is a best-effort live hint. Merge it with authoritative history by `trade.id`; do not derive counts from frame counts.

```json
{ "type": "ChainEvent", "data": { "event_type": "PositionSettled", "signature": "5eyk...base58sig", "instruction_index": 2, "slot": 312999000, "position": "PositionPdaBase58" } }
```

`event_type` is `PositionOpened`, `MarketCreated`, `MarketFinalized`, `MakerRegistered`, `PositionSettled` or `PositionLiquidated`. A chain event is identified by `(signature, instruction_index)`.

```json
{ "type": "MarketFinalized", "data": { "market_pda": "MarketPdaBase58", "settlement_price": 133000000000 } }
```

`MarketCreated` carries a [`MarketInfo`](ws-common.md#marketinfo) as `data`. `StatsUpdate` carries `{ "stats": GlobalStats }` with `usd` (`null` or `{ notional_24h, premium_24h, priced_trades_24h }`), `total_volume_24h`, `total_trades_24h`, `spread_trades_24h`, `total_price_24h`, `active_markets`, `active_makers` and `active_rfqs`.

---

## Authentication requirements

### Does not require auth

- `GetMarkets`
- `GetMarketDescriptors`
- `GetExpiries`
- `GetTokens`
- `GetTokenCaps`
- `GetIndicativePrices`
- `GetEarnSummary`
- `GetTokenMarketsInfo`
- `GetSubscriptions`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `ResumeAuth`, `StartAuth`, `AuthChallenge` (these perform auth)

### Requires auth

- `GetPositions`
- `GetMyActiveRfqs`
- `GetOrderStatus`
- `RfqRequest`, `AcceptQuote`, `CancelRfq`, `SubmitSignedSponsoredTx`
- `Subscribe`, `Unsubscribe`
- `RedeemInvite`, `ClaimReferralCode`, `GetMyReferralInfo`

### Requires auth + invite

- `RfqRequest` (the server returns `InviteRequired` if the taker has not redeemed an invite)

---

## Errors

Two error message types: `Error` (connection-level) and `RequestError` (request-correlated). See [WS common conventions](ws-common.md#error-format) for the envelope.

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "1e3a5c7e-9b0d-4f26-8a4c-6e8a0c2e4f93",
    "error": { "type": "Generic", "data": { "code": "invalid_quantity_size_rule", "message": "quantity violates min/max/step" } },
    "delivery": null
  }
}
```

`delivery` is `null` except for `SubmitSignedSponsoredTx` and `SwapSubmit` failures, where it is `not_forwarded` or `unknown`.

Parsing rule for taker integrations:

1. Handle `RequestError` for any request where you passed a `request_id`.
2. Handle `Error` for connection-level failures (auth, WebSocket errors, unknown request).
3. For both, switch on `ServerError.type`.
4. If `type == "Generic"`, switch on `data.code`.

`WsErrorCode` is a closed set. Every code is in the [WS error code catalog](ws-common.md#ws-error-codes).

Every typed `ServerError` variant and its `data` is in the [ServerError catalog](ws-common.md#servererror-catalog).

`Generic` codes a taker meets on `RfqRequest`: `invalid_timeout`, `taker_buy_unsupported`, `product_not_allowed`, `spread_invalid_strikes`, `spread_tick_violation`, `missing_tick_for_underlying_mint`, `invalid_strike_notional`, `invalid_quantity_size_rule`, `missing_size_rule_for_underlying_mint`, `insufficient_collateral`. The catalog lists the market-state and server-state codes.

### RateLimitReason (in typed `RateLimit` error)

| Value | Meaning |
|-------|---------|
| `too_many_active_rfqs_total` | Platform-wide RFQ limit reached |
| `too_many_active_rfqs_per_taker` | Per-taker RFQ limit reached |
| `too_many_failed_orders_per_rfq` | `AcceptQuote` refused: too many failed orders on this RFQ are still unresolved. Create a new RFQ |
| `too_many_quotes_per_rfq` | The RFQ holds the maximum number of quotes |
| `too_many_sessions_per_user` | Too many concurrent sessions |

---

## Related

- [Taker quickstart](../quickstart/taker-quickstart.md): narrative flow and raw sponsored-tx signing
- [Taker wire examples](../quickstart/taker-wire-examples.md): complete JSON session and branch scenarios
- [Taker TS SDK quickstart](../quickstart/web-client-ts-sdk.md)
- [Protocol flow](protocol-flow.md): trade lifecycle, economics, settlement, risk
- [WS common conventions](ws-common.md): units, envelopes, timeouts
- [Capacity limits](caps.md): OI, notional and liability caps
- [Sandbox / Devnet](sandbox.md): endpoints, faucets, program addresses
- [FAQ](faq.md)
