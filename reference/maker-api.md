# Acta Maker API Reference

## Endpoint

Quote plane:

```
wss://devnet-api.acta.markets/maker
wss://beta-api.acta.markets/maker
```

Data plane:

```
wss://devnet-api.acta.markets/maker/data
wss://beta-api.acta.markets/maker/data
```

Use `/maker` for RFQ subscriptions, quote submission, quote replacement, and
quote cancellation. Use `/maker/data` for private maker reads and recovery
queries such as `GetMyQuotes`, `GetMakerPositions`, `GetMyTrades`, and
`GetMmSummary`. These reads are also accepted on `/maker`. Use a separate data
connection to keep read requests off the quote connection.

`GetMmSummary` is for dashboard bootstrap and recovery on `/maker/data`. Do not
call it from a timer loop. Use it after connect/auth, reconnect, explicit manual
refresh, detected drift, or post-transaction reconciliation when no owner-direct
push is expected.

## Wire rules

The wire is closed.

- Every field is present on every frame. A fact that can be absent is an explicit `null`. A missing key is a decode error.
- A shape that depends on state is a tagged union, for example `terms`, `state`, `scope`, `cursor`, `verdict` and `role`.
- Unknown message types, error codes, enum values and fields are rejected.
- Requests have no defaults. Filters, limits and cursors are required fields.

## Connection flow

```
Connect -> Hello -> Welcome -> AuthRequest -> AuthChallenge -> AuthSuccess -> Snapshot -> Subscribe -> ...
```

Makers are auto-challenged on a fresh connection. The server sends `AuthRequest`
immediately after `Welcome`; `StartAuth` is unnecessary and ignored on maker
endpoints. A reconnect may instead send `ResumeAuth` with the previous maker
session ID. Resume credentials are endpoint-role bound: a maker credential is
rejected on the taker endpoint.

`AuthSuccess.expires_at` is the Unix-second deadline for starting another
connection with that maker resume credential. An already-authenticated
connection remains live until disconnect, replacement, logout, blacklist, or
server shutdown.

### Data connection

`/maker/data` has the same handshake as `/maker`: `Hello`, `Welcome`, an automatic `AuthRequest`, `AuthChallenge`, `AuthSuccess`, `Snapshot`. Sign with `quote_signing` and send the `maker_owner` pubkey, as on `/maker`. Send the same `Hello`. `on_disconnect` is required, but a data session owns no quotes.

After `Snapshot`, do not send `Subscribe`: it is rejected with `WrongEndpoint`. Send `GetMmSummary`. Owner pushes start after it is answered. A data session never replaces the `/maker` session; the limit is under [owner-direct delivery](#server---maker-owner-direct-no-subscription).

### Resume

`ResumeAuth` reconnects without a new signature, on `/maker` and on `/maker/data`. Send it after `Welcome` with `session_id` from the last `AuthSuccess`:

```json
{ "type": "ResumeAuth", "data": { "session_id": "3a5c7e9b-1d2f-4e60-8a4c-6b8d0f2a4c71" } }
```

The server has already sent `AuthRequest`. Keep that challenge, and do not answer it while the resume is pending.

- On success you get `AuthSuccess` with the same `session_id`, then `Snapshot`. The session's server-side subscriptions are restored, and the pending challenge no longer applies.
- On `AuthError` with reason `session_expired`, the connection stays open and the pending challenge is still valid. Sign it and send `AuthChallenge` before the auth deadline.
- Any other `AuthError` reason is in the [WS error code catalog](ws-common.md#ws-error-codes).

A failed resume counts as a failed auth attempt.

### Hello (first message)

Protocol constraints:

- `Hello` MUST be the first client message.
- `Hello` timeout: `5000ms`.
- `protocol_version` must equal the server's `1.0.0` exactly. There is no range check.
- All four fields are required. Extra fields are rejected.

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

`on_disconnect` sets what the server does with the quotes this connection owns when it ends:

| Value | Effect |
|-------|--------|
| `cancel_quotes` | On an actual disconnect (socket close or liveness timeout), the server removes active and retained non-winning quotes still owned by that physical session, including retained quotes in locked RFQs. Ownership takeover preserves quotes; late cleanup of the replaced connection cannot cancel the new owner's book. Quotes a taker has already accepted (`QuoteSelected`) stay locked and settle or expire on their own timers. Session resume does not restore cancelled quotes; re-quote after reconnecting. |
| `keep_quotes` | Resting quotes survive a disconnect and can be filled while the maker is offline. |

`QuoteExpired` is always sent. There is no feature negotiation.

A different `protocol_version` gets an `Error` frame and a WebSocket close with code `4006`:

```json
{
  "type": "Error",
  "data": {
    "type": "Generic",
    "data": {
      "code": "unsupported_protocol_version",
      "message": "Protocol 0.9.0 is not supported; expected 1.0.0"
    }
  }
}
```

### Welcome (server -> client)

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

`server_time_unix_ms` is required. Use it for clock skew estimation (see [WS common conventions](ws-common.md)).

`min_quote_validity_seconds` and `settlement_buffer_seconds` are the venue quote window, the same values as `GET /api/v1/venue`: `valid_until` must be at least `now + min_quote_validity_seconds`, and a quote stays executable only while `valid_until - settlement_buffer_seconds > now`. The Rust `QuoteBuilder::welcome(&welcome)` applies both.

### Subscribe

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

All fields are required. The server answers with `SubscribeAck` echoing `request_id` and the channels that were newly added.

- `products` is a nonempty list of unique product names: `covered_call`, `cash_secured_put`, `call_spread`, `put_spread`, `cash_put`. An empty list is rejected with `invalid_products`. RFQ fan-out follows this selection. Position, history and recovery reads include every product regardless of it.
- `underlying_mints` and `quote_mints` filter market broadcasts. `[]` means every mint.
- Each `Subscribe` replaces the product selection and both mint lists. Channels are additive.

### Connection policy defaults

- Auth deadline (after challenge issued): `15s`
- Idle timeout: `90s`
- Server WS ping interval: `30s`
- Max consecutive parse errors before close: `3`
- Inbound message size: `32 KiB`
- Rate limits: [Rate limits and admission](ws-common.md#rate-limits-and-admission).

### Snapshot (server -> maker)

```json
{
  "type": "Snapshot",
  "data": {
    "markets": [
      {
        "pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
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

`Snapshot.markets` lists the active markets as compact [`MarketInfo`](ws-common.md#marketinfo). `settlement_price` is `null` until the market settles. `products` is described under [Market products](#market-products). This is a different DTO from the `MarketDescriptor` in `RfqBroadcast`, which has additional fields (`chain_id`, `program_id`, `market_pda`, `underlying_mint`, `quote_mint`, `collateral_mint`, `settlement_mint`). Use `GetMarketDescriptors` for the full market descriptor.

---

## Message index

### Client -> Server

- `Hello`, `AuthChallenge`, `ResumeAuth`, `Logout`
- `Subscribe`, `Unsubscribe`, `Ping`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `Quote`, `ReplaceQuote`, `BatchQuotes`, `CancelQuote`, `CancelAllQuotes`
- `IndicativePricesResponse`
- `GetOrderStatus`, `GetMyQuotes`, `GetMakerPositions`, `GetMarketsForMaker`, `GetMmSummary`, `GetMyTrades`, `GetTokenCaps`, `GetMyCaps`, `CheckQuote`, `GetSubscriptions`
- `GetActiveRfqs`, `GetMarkets`, `GetMarketDescriptors`, `GetExpiries`, `GetTokens`

### Server -> Maker (direct)

- `Welcome`, `LogoutSuccess`
- `AuthRequest`, `AuthSuccess`, `AuthError`
- `Snapshot`, `ReferenceChanged`
- `RfqBroadcast`, `RfqSkipped`
- `QuoteRefreshRequested`, `QuoteAcknowledged`, `QuoteBestStatus`, `QuoteOutbid`, `QuoteSelected`
- `QuoteRejected`, `QuoteFilled`, `QuoteCancelled`, `QuoteExpired`, `RfqAvailableAgain`, `RfqClosed`
- `CancelQuoteAck`, `CancelAllQuotesAck`, `BatchQuotesAck`
- `IndicativePricesRequest`
- `SubscriptionUpdated`
- `Error`, `RequestError`, `Pong`
- `SubscribeAck`, `UnsubscribeAck`

### Server -> Maker (responses / query results)

- `OrderStatus`, `MyQuotes`, `MakerPositions`, `MakerMarkets`, `MmSummary`, `MyTrades`, `TokenCaps`, `MyCaps`, `QuoteCheck`, `Subscriptions`
- `ActiveRfqs`, `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`

### Server -> Maker (owner-direct, no subscription)

- `PositionUpdated`, `MakerCapsUpdated`, `TradeExecuted`

Delivered to the owner's quote session and ready `/maker/data` sessions. Data
sessions become eligible after a successful `GetMmSummary` response; authentication
alone does not enable pushes. No subscription is needed. Up to four authenticated
data sessions per owner are allowed; the fifth receives `AuthError` with
`reason = "session_limit_exceeded"`. Disconnect or logout releases its slot.
A full data queue closes only that session (`4002 slow_consumer`). Reconcile on
restart/reconnect: these pushes are not replayed.

### Server -> Maker (broadcast if subscribed)

- `StatsUpdate`, `ChainEvent`
- `MarketCreated`, `MarketFinalized`
- `VaultCatalogChanged`, `VaultProjectionAdvanced` (`vaults` channel; see [ws-common](ws-common.md#vaultcatalogchanged))

### Delivery by endpoint

Pushes reach a session in one of three ways: a channel the session subscribed
to (`Subscribe` is accepted on `/maker` and `/vault`, not on `/maker/data`),
the session that sent the quote, or the maker owner's `/maker` and ready
`/maker/data` sessions regardless of subscriptions. Query responses (`MyQuotes`, `MakerPositions`,
`MmSummary`, ...) return on the connection that asked.

| Message | `/maker` | `/maker/data` | `/vault` |
|---|---|---|---|
| `RfqBroadcast` | `rfqs` channel, within the product and mint scope | no | sessions with `sign_maker_quotes`, no subscription |
| `RfqSkipped` | `rfqs` channel, makers the caps prefilter skipped | no | sessions with `sign_maker_quotes` the prefilter skipped |
| `IndicativePricesRequest` | `rfqs` channel | no | no |
| `QuoteAcknowledged`, `QuoteRejected`, `QuoteBestStatus`, `QuoteOutbid`, `QuoteSelected`, `QuoteRefreshRequested`, `QuoteExpired`, `QuoteCancelled` | the session that sent the quote | no | the session that sent the quote |
| `QuoteFilled` | the session that sent the winning quote | no | the session that sent the winning quote |
| `RfqClosed`, `RfqAvailableAgain` | every session that quoted the RFQ | no | every session that quoted the RFQ, and the taker session |
| `OrderAccepted`, `SponsoredTxToSign`, `OrderSubmitted`, `OrderConfirmed`, `OrderFailed` | never: these go to the taker | no | only as the taker of its own RFQ |
| `TradeExecuted` | owner's session, no subscription | owner's ready sessions, no subscription | no |
| `PositionUpdated`, `MakerCapsUpdated` | owner's session, no subscription | owner's ready sessions, no subscription | no |
| `ReferenceChanged` | every session | every ready session | every session |
| `MarketCreated`, `MarketFinalized` | `markets` channel | no | no |
| `StatsUpdate` / `ChainEvent` | `stats` / `chain_events` channel | no | no |
| `VaultCatalogChanged` | `vaults` channel | no | no |
| `VaultProjectionAdvanced` | `vaults` channel | no | `vaults` channel, own vault only |

A maker learns of its fill from `QuoteFilled` (and `TradeExecuted`), never from
`OrderConfirmed`. `PositionUpdated` and `MakerCapsUpdated` reach the owner's
`/maker` session and its ready `/maker/data` sessions; a client with neither
recovers them with `GetMakerPositions`, `GetMyCaps` or `GetMmSummary`.

---

## Products and terms

Every RFQ, quote, position and trade carries two required fields: `terms` and `side`.

`terms` is a tagged union. Each variant carries exactly its own fields.

| `terms.type` | Fields | Settlement |
|---|---|---|
| `covered_call` | `strike` | physical |
| `cash_secured_put` | `strike` | physical |
| `call_spread` | `lower_strike`, `upper_strike` | cash, in the quote mint |
| `put_spread` | `lower_strike`, `upper_strike` | cash, in the quote mint |
| `cash_put` | `strike` | cash, in the quote mint |

```json
{ "type": "covered_call", "strike": 160000000000 }
{ "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 }
{ "type": "cash_put", "strike": 150000000000 }
```

- Strikes are integers scaled by 1e9 per underlying token, independent of mint decimals. `strike_tick` from `GetTokens` is on the same scale.
- `side` describes the taker: `sell` or `buy`. Only `sell` is executable. A `buy` quote is rejected with `taker_buy_unsupported`.
- `cash_put` is a put with strike K that settles in cash. On chain it executes as a put spread with legs `(0, K)`. It is signed with the put-spread product byte, lower strike `0` and upper strike K.
- Public `call_spread` and `put_spread` terms require `0 < lower_strike < upper_strike`. Both strikes are chosen by the taker. For puts the short strike is the upper strike.
- A put with a zero lower leg is a `cash_put`. A `put_spread` with `lower_strike` `0` is rejected with `spread_invalid_strikes`. On chain a position with legs `(0, K)` reads back as `cash_put`.
- Both strikes of a spread, and the strike of a `cash_put`, must be multiples of the underlying mint's `strike_tick`. A mint without a tick is rejected with `missing_tick_for_underlying_mint`.
- `order_options` lists complete alternative `terms`, never independent legs. A spread RFQ lists exactly its requested pair.

### Market products

Each market row (`Snapshot`, `Markets`, `MarketDescriptors`, `MakerMarkets`, `MarketCreated`, `GET /api/v1/markets`) carries a nonempty `products` list. It holds only the products of the pair's policy that fit the market's side:

| Market | Possible `products` |
|---|---|
| `is_put: false` | `covered_call`, `call_spread` |
| `is_put: true` | `cash_secured_put`, `put_spread`, `cash_put` |

A market with no allowed product is not listed. An RFQ for a product the pair does not allow is rejected with `product_not_allowed`.

### Cash settlement and fees

Spreads and `cash_put` pay out in the quote mint. Their lifecycle is `open` then `settled`. They have no funded state, no liquidation and no early close, so a position of one of these products never produces `PositionUpdated` with `update_type` `funded` or `liquidated`.

With `ud` underlying decimals, `qd` quote decimals and `width = upper_strike - lower_strike` (`width = strike` for `cash_put`):

- Collateral: `C = ceil(width * quantity * 10^qd / (1e9 * 10^ud))`, in quote atoms.
- Gross premium: `gross = floor(price * quantity * 10^qd / (1e9 * 10^ud))`, in quote atoms.
- Admission requires `quantity > 0`, `0 < price < width` and `0 < gross < C`.
- Premium fee: `gross - quote(price - floor(price * protocol_fee_bps_premium / 10000))`, where `quote(x)` is the same scaled conversion with floor.
- Volume fee: `floor(exposure * protocol_fee_bps_volume / 10000)`. `exposure` is `quote(width)` for spreads and `cash_put`, and `quote(strike)` for covered calls and cash-secured puts.
- Protocol fee: `min(premium fee, volume fee)`. Net premium: `gross - protocol fee`.
- `protocol_fee_bps_premium` and `protocol_fee_bps_volume` come from `GET /api/v1/venue`.

Worked example. A call spread 100 to 120 on 1 SOL (`quantity` 1000000000, `ud` 9, `qd` 6) at `price` 5000000000, with fees of 50 and 10 bps:

| Value | Atoms |
|---|---|
| `C` | 20000000 |
| `gross` | 5000000 |
| premium fee | 25000 |
| volume fee | 20000 |
| protocol fee | 20000 |
| net premium | 4980000 |

A cash-settled position settles at settlement price S:

- Call spread payoff to the maker: `ceil(quantity * min(max(S - lower, 0), width))`, scaled to quote atoms.
- Put spread payoff: `ceil(quantity * min(max(upper - S, 0), width))`, scaled to quote atoms.
- Cash put payoff: `ceil(quantity * max(K - S, 0))`, scaled to quote atoms and capped at `C`.
- The taker receives the remaining collateral.

The Rust SDK computes these with `SpreadTerms::premium` (spreads and cash puts) and `physical_premium` (covered calls and cash-secured puts). Both return the contract's rounding.

---

## Quote flow

### RfqBroadcast (server -> maker)

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "market": {
      "chain_id": 0,
      "program_id": "33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U",
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 1710600000,
      "is_put": false,
      "collateral_mint": "So11111111111111111111111111111111111111112",
      "settlement_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
    },
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "quantity": 1000000000,
    "expires_at": 1710000050,
    "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "order_options": [
      { "type": "covered_call", "strike": 150000000000 },
      { "type": "covered_call", "strike": 160000000000 }
    ],
    "sent_at_unix_ms": 1710000000123
  }
}
```

`terms` is the RFQ's primary terms. `order_options` is a list of complete alternative terms and may be empty.

- Covered call and cash-secured put: the quote's strike must be the strike of `terms` or of an `order_options` entry. Any other strike is rejected with `invalid_strike`.
- Spreads and cash puts: the quote's `terms` must equal the RFQ's `terms` exactly. Any difference is rejected with `quote_identity_mismatch`.
- The quote's product and `side` must match the RFQ. Otherwise the quote is rejected with `quote_identity_mismatch`.

### RfqSkipped (server -> maker)

Sent instead of `RfqBroadcast` when cap limits pre-filter the maker. `cap_detail` is the failing cap; the full catalog is in [CapError](#caperror).

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "market_id": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "quantity": 1000000000,
    "cap_detail": {
      "token_oi_cap_exceeded": {
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "current": 1000000000000,
        "limit": 1000000000000
      }
    }
  }
}
```

The filter runs before broadcast and knows no premium, so `maker_insufficient_balance.required` is always `0` here.

### Quote (maker -> server)

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
    "signature": "<base58 ed25519 signature of order_id>"
  }
}
```

A spread or cash-put quote uses the same fields with the matching `terms`:

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
    "signature": "<base58 ed25519 signature of order_id>"
  }
}
```

`price` is the gross premium per underlying unit, scaled by 1e9.

Rules:
- `valid_until` MUST be >= `now + min_quote_validity_seconds` (from `Welcome` or `GET /api/v1/venue`). A shorter expiry is rejected with `quote_expiry_too_short`.
- `valid_until` MUST be <= the market's `expiry_ts`. A later expiry is rejected with `market_expired`.
- The server applies a **settlement buffer** (`settlement_buffer_seconds`). Your quote is active for trading until `valid_until` minus that buffer. Set `valid_until` to at least `rfq.expires_at + min_quote_validity_seconds` so the quote stays tradable for the whole auction.
- `order_id = sha256(preimage)`. The preimage is 182 bytes for covered calls and cash-secured puts and 190 bytes for spreads and cash puts. See [Order id preimage](#order-id-preimage).
- The maker signs only the 32-byte `order_id`.
- `terms` and `side` must match the RFQ as described under [RfqBroadcast](#rfqbroadcast-server---maker).

### Order id preimage

Before building the preimage, derive the market PDA locally using Solana PDA seeds
`["market", underlying_mint[32], quote_mint[32], expiry_ts_u64_le[8], is_put_u8[1]]`
under the Acta program ID from your deployment configuration. The program ID is the same on devnet and mainnet. `GET /api/v1/venue` returns it as `program_id`. Require chain ID
`0`, that configured program ID, and equality with `market_pda`. Also check that
the product fits `is_put`: puts are `cash_secured_put`, `put_spread` and `cash_put`; calls are `covered_call` and `call_spread`. Check the mints on the RFQ's `market`: collateral/settlement are underlying/quote for covered
calls, quote/underlying for cash-secured puts, and quote/quote for spreads and cash puts. These pairs apply to `RfqBroadcast`, whose `market` is built for the RFQ's product. Rows of `MarketDescriptors` are market-level: they always show the physical pair for the market's side, so on a spread or cash-put RFQ they differ from the broadcast. Do not copy an endpoint-selected program ID
into the derivation. Rust `RfqBinding::from_broadcast` and TS
`buildSignedQuoteFromRfq` perform these checks before signing. Strategy mint
allowlists and pricing inputs remain your responsibility; PDA verification does
not certify an endpoint's prices or mint decimals.

All numeric fields use little-endian encoding. The domain tag is `ACTA` for both layouts.

Physical layout (covered call, cash-secured put), 182 bytes (offset, size):
- `0,4` -> `domain_tag` (`"ACTA"`)
- `4,8` -> `chain_id` (`u64`, `0` for Solana)
- `12,32` -> `program_id`
- `44,1` -> `is_taker_buy` (`u8`: `0` for side `sell`, `1` for `buy`)
- `45,1` -> `position_type` (`u8`: `0` covered_call, `1` cash_secured_put)
- `46,32` -> `market`
- `78,8` -> `strike` (`u64`)
- `86,8` -> `quantity` (`u64`)
- `94,8` -> `gross_price` (`u64`)
- `102,8` -> `valid_until` (`u64`, unix seconds)
- `110,32` -> `maker` (the `maker_owner` pubkey)
- `142,32` -> `taker`
- `174,8` -> `nonce` (`u64`)

Spread layout (call spread, put spread, cash put), 190 bytes. Offsets 0 to 181 are as above, with these differences:
- `45,1` -> `position_type` (`u8`: `2` call_spread, `3` put_spread)
- `78,8` -> `lower_strike` (`u64`)
- `182,8` -> `upper_strike` (`u64`)

A `cash_put` with strike K uses `position_type` `3`, `lower_strike` `0` and `upper_strike` K.

`gross_price` is the same value as `price` in the `Quote` message.

`maker` is the `maker_owner` pubkey. It is not the maker PDA (`AuthSuccess.maker_pda`) and not the `quote_signing` key. For a vault operator's quote it is the vault PDA. `taker` is `RfqBroadcast.taker`.

#### Test vectors

Test your builder against these values before quoting. All three use `chain_id` 0, `program_id` 32 bytes of `0x04`, `market` 32 bytes of `0x01`, `maker` 32 bytes of `0x02`, `taker` 32 bytes of `0x03`, side `sell`, `quantity` 1000000, `gross_price` 1000000000, `valid_until` 123 and `nonce` 999. `order_id` here is plain hex; the wire adds `0x`.

Covered call, 182 bytes: `position_type` 0, `strike` 100000000000.

```text
preimage: 41435441000000000000000004040404040404040404040404040404040404040404040404040404040404040000010101010101010101010101010101010101010101010101010101010101010100e876481700000040420f000000000000ca9a3b000000007b0000000000000002020202020202020202020202020202020202020202020202020202020202020303030303030303030303030303030303030303030303030303030303030303e703000000000000
order_id: 292033726f26752392a0c18ccdc1e09016c0601e3d74b42fc0d5ef98e0190eb6
```

Call spread, 190 bytes: `position_type` 2, `lower_strike` 100000000000, `upper_strike` 120000000000.

```text
preimage: 41435441000000000000000004040404040404040404040404040404040404040404040404040404040404040002010101010101010101010101010101010101010101010101010101010101010100e876481700000040420f000000000000ca9a3b000000007b0000000000000002020202020202020202020202020202020202020202020202020202020202020303030303030303030303030303030303030303030303030303030303030303e70300000000000000b08ef01b000000
order_id: 37d008c2a5ca5ea9dbf727994e4fdb394cc52bd53b1ed91e74a03bfbdd672c3f
```

Cash put, 190 bytes: `position_type` 3, `lower_strike` 0, `upper_strike` 120000000000 (`terms.strike` 120000000000).

```text
preimage: 414354410000000000000000040404040404040404040404040404040404040404040404040404040404040400030101010101010101010101010101010101010101010101010101010101010101000000000000000040420f000000000000ca9a3b000000007b0000000000000002020202020202020202020202020202020202020202020202020202020202020303030303030303030303030303030303030303030303030303030303030303e70300000000000000b08ef01b000000
order_id: 03b50d62e08b00556ccb458945ebfa2570fdd65b50e9753e911584fb4951c52c
```

### ReplaceQuote (maker -> server)

Atomic cancel-and-resubmit for a specific quote. Cancels the quote identified by `old_order_id` and submits a new quote in a single operation. The old quote is removed **only if** the new quote passes all validation (caps, signature, `valid_until`). If validation fails, the old quote remains active.

```json
{
  "type": "ReplaceQuote",
  "data": {
    "old_order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "price": 5500000000,
    "valid_until": 1710000120,
    "nonce": 43,
    "order_id": "0xb2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3",
    "signature": "<base58 ed25519 signature of order_id>"
  }
}
```

`old_order_id` identifies the specific quote being replaced. Server responds with `QuoteAcknowledged` (with `replaced_order_id`) on success, or `QuoteRejected` / `Error` on failure.

**Errors:**
- `QuoteRejected` with standard reasons (`invalid_strike`, `cap_exceeded`, etc.) — new quote failed validation; old quote remains active
- `Error { type: "QuoteLocked", data: { reason } }` — the old quote cannot be replaced: `reason` is `rfq_locked` while the taker's signature is pending and `order_submitted` once the signed transaction is queued
- `Error { type: "QuoteNotFound" }` — `old_order_id` not found (race: already cancelled/expired)

### BatchQuotes (maker -> server)

Submit multiple quotes in one message. A batch must contain at most `max_batch_quotes` quotes (default 50). The server charges one quote-rate-limit token per quote element; an empty batch costs one token. Within an accepted batch, each quote is validated like a standalone `Quote`. The server responds with one `BatchQuotesAck` containing `results[]` — one entry (`acknowledged` or `rejected`) per quote that reached quote-level validation. **Correlate results by `order_id`, not by position:** a quote rejected at initial validation (bad signature, expired, unregistered maker) is reported as an inline error and may be omitted from `results[]`. Partial success is allowed. For an implicit same-terms replacement, the aggregated `QuoteAcknowledged` leaves `replaced_order_id` null; the replaced order id is delivered on the asynchronous per-quote `QuoteAcknowledged` event.

```json
{
  "type": "BatchQuotes",
  "data": {
    "quotes": [
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "terms": { "type": "covered_call", "strike": 150000000000 },
        "side": "sell",
        "price": 4500000000,
        "valid_until": 1710000120,
        "nonce": 44,
        "order_id": "0xaaaa2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "<base58 ed25519 signature of order_id>"
      },
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "terms": { "type": "covered_call", "strike": 160000000000 },
        "side": "sell",
        "price": 5000000000,
        "valid_until": 1710000120,
        "nonce": 45,
        "order_id": "0xbbbb2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "<base58 ed25519 signature of order_id>"
      }
    ]
  }
}
```

Each entry in `quotes` has the same schema as a standalone `Quote` message. Quotes are processed sequentially — a rejection of one quote does not block subsequent quotes.

Each `results` entry is `{ "status": "acknowledged", "data": <QuoteAcknowledged> }` or `{ "status": "rejected", "data": <QuoteRejected> }`:

```json
{
  "type": "BatchQuotesAck",
  "data": {
    "results": [
      {
        "status": "acknowledged",
        "data": {
          "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
          "order_id": "0xaaaa2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
          "replaced_order_id": null
        }
      },
      {
        "status": "rejected",
        "data": {
          "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
          "order_id": "0xbbbb2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
          "reason": "invalid_strike",
          "message": "strike is not offered by the RFQ"
        }
      }
    ]
  }
}
```

### QuoteRejected (server -> maker)

`QuoteRejected` is sent for quote-specific validation failures and carries a typed `reason` for programmatic handling. `message` is a required string with detail for logs.

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "reason": "invalid_strike",
    "message": "strike is not offered by the RFQ"
  }
}
```

A cap rejection carries the failing
dimension inside the reason instead of a bare string, so it cannot arrive
without its detail:

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "reason": { "cap_exceeded": { "maker_position_cap_exceeded": { "current": 3, "limit": 3 } } },
    "message": "Maker position cap exceeded (3/3)"
  }
}
```

The value under `cap_exceeded` is a `CapError`: a bare string for
`caps_unavailable`, otherwise an object keyed by the cap code (see
[caps.md](caps.md) for the codes and amounts).

### QuoteRejectReason

| Value | Meaning |
|-------|---------|
| `product_not_allowed` | The pair's product policy does not allow the quoted product |
| `unauthorized` | Session is not an authenticated maker, or a vault maker that may not trade or hold this position (see `message`) |
| `invalid_strike` | Covered-call or cash-secured-put strike is not offered by the RFQ |
| `spread_invalid_strikes` | Zero, reversed or otherwise invalid spread strikes |
| `spread_tick_violation` | A strike is not a multiple of the underlying mint's strike tick |
| `spread_premium_invalid` | `price >= width`, zero gross premium, `gross >= C`, or the amounts are not representable |
| `taker_buy_unsupported` | `side` is `buy` |
| `quote_identity_mismatch` | Product, `side` or spread `terms` differ from the RFQ |
| `market_expired` | `valid_until` is after the market's expiry, or the market has expired |
| `quote_expiry_too_short` | `valid_until` below `now + min_quote_validity_seconds` |
| `invalid_signature` | Maker signature verification failed |
| `maker_not_registered` | Maker not found in on-chain registry |
| `order_id_mismatch` | `order_id` does not match `sha256(preimage)` |
| `{ "cap_exceeded": CapError }` | Platform or maker cap limit exceeded; the failing dimension with `current`/`limit` is the payload |
| `rfq_not_found` | RFQ does not exist |
| `rfq_not_active` | RFQ is no longer accepting quotes |
| `duplicate_order_id` | `order_id` already submitted |
| `rate_limited` | Quote rate limit exceeded; retry after backing off |
| `trading_paused` | Trading is paused venue-wide; retry after it resumes |
| `internal_error` | Server-side failure; the quote was not accepted, retry is safe |

### QuoteRefreshRequested (server -> maker)

```json
{
  "type": "QuoteRefreshRequested",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "min_valid_until": 1710000121,
    "reason": "expiring_soon"
  }
}
```

The refresh timer fires at `valid_until - settlement_buffer_seconds - quote_refresh_lead_seconds`. The quote then stays in the book but cannot be accepted until you send a fresh one. `min_valid_until` is the fire time plus the buffer, the lead and one second.

Identified by `(rfq_id, terms)`, not `order_id`. If quoting several terms per RFQ, keep a local `(rfq_id, terms) -> order_id` mapping to know which quote to refresh.

`reason` is `expired` or `expiring_soon`. `min_valid_until` includes the settlement buffer; the refreshed quote must have `valid_until >= min_valid_until`.

### QuoteSelected (server -> maker)

```json
{
  "type": "QuoteSelected",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "price": 5000000000,
    "quantity": 1000000000,
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "signature_deadline": 1710000040
  }
}
```

### QuoteFilled (server -> maker)

```json
{
  "type": "QuoteFilled",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "expiry_ts": 1710600000,
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "price": 5000000000,
    "quantity": 1000000000,
    "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
    "tx_signature": "<base58 transaction signature>",
    "filled_at": 1710000042
  }
}
```

`QuoteFilled` is the fill-details event for the winning maker. `market_pda`,
`expiry_ts` (market expiry, Unix seconds; `null` if the server no longer knows the market, so resolve it from `market_pda`), `terms` and `side` identify the
market and product without the RFQ, so a fill can be attributed after a restart. Mint
decimals for the market come from `GetMarketDescriptors`.

`QuoteFilled` is the fill fact. It carries `price` and `quantity`, and it is sent before any chain event reaches the server, so it has no fee, net premium or collateral fields.

The authoritative net premium is `total_premium` on the `PositionUpdated` row, applied by `revision`. `MakerPositions` and `MmSummary` return the same row.

To have the amounts at once, compute them locally with the rates from `GET /api/v1/venue`: the Rust SDK's `physical_premium` for covered calls and cash-secured puts, and `SpreadTerms::premium` for spreads and cash puts. The result is an estimate if the venue rates changed.

`quantity` is in underlying atoms and `price` is 1e9-scaled per underlying token, so the gross
premium (before protocol fees) in quote atoms is `floor(price * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`
(the contract's rounding; the Rust SDK's `total_premium_quote_atoms` and the TS
SDK's `totalPremiumQuoteAtoms` compute it).

`RfqClosed` is the terminal RFQ event.

Current runtime behavior for successful fills:
- winner maker receives `RfqClosed` and then `QuoteFilled`; accept fill details after the RFQ has closed
- taker receives `OrderConfirmed` and then `RfqClosed`
- close RFQ state only on `RfqClosed`; process `QuoteFilled` as fill-details information

### QuoteAcknowledged (server -> maker)

```json
{
  "type": "QuoteAcknowledged",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "replaced_order_id": null
  }
}
```

`replaced_order_id` is the displaced order when the new quote replaced a prior quote from the same maker on the same `(rfq, terms)`. It is `null` otherwise.

### QuoteBestStatus (server -> maker)

```json
{
  "type": "QuoteBestStatus",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "is_best": true,
    "current_best_price": 5000000000
  }
}
```

`is_best`: whether the maker's quote is currently the highest premium.
`current_best_price`: the best price on the RFQ (may be another maker's price). `null` when there is none.

### QuoteOutbid (server -> maker)

```json
{
  "type": "QuoteOutbid",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "your_price": 5000000000,
    "current_best_price": 5500000000
  }
}
```

`your_price`: the maker's current quote price.
`current_best_price`: the new best price that outbid the maker. `null` when there is none.

### QuoteExpired (server -> maker)

Always sent when a quote expires.

```json
{
  "type": "QuoteExpired",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "reason": "expired"
  }
}
```

`reason` is `expired` or `expiring_soon`.

### QuoteCancelled (server -> maker)

```json
{
  "type": "QuoteCancelled",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_ids": ["0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"],
    "reason": "requested",
    "cancelled_at": 1710000025
  }
}
```

`reason` values: `requested` (maker requested), `risk_check` (server-side risk), `rfq_accepted` (RFQ accepted another quote), `maker_disconnected` (the session said `on_disconnect: "cancel_quotes"` and disconnected; sent to the session that is gone, so a reconnecting maker sees the cancellation through `GetMyQuotes`).

### RfqClosed (server -> maker)

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 3,
    "reason": "filled",
    "your_quote": {
      "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
      "status": "outbid",
      "price": 5000000000
    },
    "winner": {
      "maker": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
      "price": 5500000000,
      "tx_signature": "<base58 transaction signature>"
    },
    "closed_at": 1710000042
  }
}
```

`reason` values: `expired`, `taker_cancelled`, `filled`, `market_expired`, `ladder_timeout`.

`your_quote` is this session's quote on the RFQ, or `null` when it has none. `your_quote.status` values: `expired`, `outbid`, `cancelled`, `filled`. A session with several quotes gets one of them; a session that won gets the winning quote.

`winner` is set on `filled` and `null` otherwise.

- On `filled`, every recipient gets `winner`, including `winner.tx_signature`. The winning session gets `your_quote.status` `filled`, a session with another live quote gets `outbid`, and a session whose quotes are already gone gets `null`.
- On any other reason, a session with a live quote gets it with status `expired` (reason `expired`) or `cancelled`. A session whose quotes are already gone gets `null`.

### RfqAvailableAgain (server -> maker)

```json
{
  "type": "RfqAvailableAgain",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "signature_timeout",
    "available_again_at": 1710000045
  }
}
```

Sent when a previously locked RFQ becomes available for new quotes (e.g. taker failed to sign in time).

`reason` values: `signature_timeout`, `tx_failed`, `tx_build_failed`.

---

## Quote management

### CancelQuote (maker -> server)

```json
{
  "type": "CancelQuote",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  }
}
```

Cancels **all** of this connection's active quotes on the given RFQ (across all terms). There is no per-order_id cancel — use `CancelQuote` per rfq_id. The server responds once the cancellation is applied with `CancelQuoteAck { request_id, rfq_id, cancelled_order_ids }`, or a correlated `RequestError` (for example `RfqNotFound`, `RfqNotActive`, `QuoteLocked`). `QuoteCancelled` is a lifecycle notification, not the command receipt.

### CancelQuoteAck (server -> maker)

```json
{
  "type": "CancelQuoteAck",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "cancelled_order_ids": ["0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"]
  }
}
```

On an Active RFQ an empty list is an idempotent no-op: the connection has no quote to remove there. While the taker's signature is pending or the signed transaction is queued, single-RFQ cancellation is rejected as a whole with `QuoteLocked`, even for a non-winning maker.

### CancelAllQuotes (maker -> server)

```json
{
  "type": "CancelAllQuotes",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "market": { "type": "only", "value": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi" }
  }
}
```

`market` is a `QueryFilter`. `{ "type": "all" }` considers all markets; `only` restricts cancellation to that market. CancelAll removes this connection's active and retained non-winning quotes, including those inside locked RFQs. Removed retained quotes cannot return on rollback. Selected/awaiting-signature and executing orders are preserved. Cancel-on-disconnect (`on_disconnect: "cancel_quotes"`) applies the same cleanup once the server processes the disconnect. Neither operation is an account-wide persistent halt.

The server responds with `CancelAllQuotesAck` once the cancellation is applied. An invalid `market` returns `RequestError` with `InvalidMarket`; it does not broaden cancellation to all markets.

### CancelAllQuotesAck (server -> maker)

```json
{
  "type": "CancelAllQuotesAck",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "cancelled_count": 0,
    "cancelled_order_ids": []
  }
}
```

`cancelled_count` equals `cancelled_order_ids.length`; both report the orders actually removed. An empty list means this command removed no quotes; it does not establish that no selected or executing obligations remain. Lifecycle `QuoteCancelled` messages are separate from the receipt and must not complete another request's await.

Quote, batch, replacement and cancellation commands from one connection retain FIFO order from the SDK writer through the server's processing. Commands before a cancellation may execute first; commands sent afterward may create new quotes. Cancellation is not a persistent account halt.

If the connection drops before the ACK, retain an unknown outcome and reconcile instead of automatically replaying the command.

### Dynamic subscription management

Modify subscriptions incrementally. Every field is required.

```json
{ "type": "AddMints", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000101", "underlying_mints": ["So11111111111111111111111111111111111111112"], "quote_mints": [] } }
{ "type": "RemoveMints", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000102", "underlying_mints": ["So11111111111111111111111111111111111111112"], "quote_mints": [] } }
{ "type": "AddChannels", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000103", "channels": ["trades"] } }
{ "type": "RemoveChannels", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000104", "channels": ["stats"] } }
```

In `AddMints` and `RemoveMints`, `[]` means no change for that list. All four respond with `SubscriptionUpdated` containing the full subscription state:

```json
{
  "type": "SubscriptionUpdated",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000101",
    "channels": ["rfqs", "trades"],
    "underlying_mints": ["So11111111111111111111111111111111111111112"],
    "quote_mints": []
  }
}
```

In responses, an empty mint list means that dimension is unfiltered.

### Unsubscribe (maker -> server)

```json
{
  "type": "Unsubscribe",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000105",
    "channels": ["trades"]
  }
}
```

Server responds with `UnsubscribeAck` echoing `request_id` and the channels that were removed. Use `GetSubscriptions` to verify the subscription state.

### Ping (maker -> server)

```json
{ "type": "Ping" }
```

Unit variant, no `data` field. Server responds with `Pong`.

---

## Discovery

### Discovery requests

All discovery requests require `request_id` (UUID). The server echoes `request_id` in the corresponding response. Maker-private and recovery reads should be sent on `/maker/data`; `GetSubscriptions` stays on `/maker` because it describes the live quote-plane subscription state.

```json
{ "type": "GetMyQuotes", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000001", "scope": { "type": "live" } } }
{ "type": "GetOrderStatus", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000002", "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a" } }
{ "type": "GetMmSummary", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000003" } }
{ "type": "GetSubscriptions", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000004" } }
{ "type": "GetActiveRfqs", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000005" } }
{ "type": "GetTokenCaps", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000006", "include_markets": true } }
{ "type": "GetMyCaps", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000007" } }
{ "type": "GetMarkets", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000008" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000009" } }
{ "type": "GetTokens", "data": { "request_id": "a1b2c3d4-0000-4000-8000-00000000000a" } }
{ "type": "GetExpiries", "data": { "request_id": "a1b2c3d4-0000-4000-8000-00000000000b", "underlying_mint": { "type": "all" }, "quote_mint": { "type": "all" }, "is_put": { "type": "all" } } }
```

Request notes:
- `GetMarkets`, `GetMarketDescriptors`, `GetExpiries` and `GetTokens` answer from the [reference snapshot](ws-common.md#reference-data): **tradable** markets only — non-finalized, non-disabled, and before the pre-expiry trading cutoff. Re-read on [`ReferenceChanged`](ws-common.md#referencechanged).
- `GetMyQuotes` takes a tagged `scope`: `{ "type": "live" }` or `{ "type": "history", "limit": N, "cursor": PageCursor }`. Live is the complete, unpaged set of this owner's unfinished quotes across previous connections, including retained and selected quotes. A missing/incomplete source returns an error, not a successful partial set. History contains only persisted historical rows and is not a substitute for Live or execution lookup.
- `GetMarketsForMaker` uses the wider active set (non-finalized, non-disabled, `expiry_ts > now`), so it still lists a market during its final pre-expiry no-trade window.

### Filters and pages

Single-value filters, time bounds and page cursors are tagged unions. List filters are required lists where `[]` means every value.

```json
{ "type": "all" }
{ "type": "only", "value": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi" }
{ "type": "unbounded" }
{ "type": "bounded", "value": 1710000000 }
{ "type": "start" }
{ "type": "after", "at": 1710000042, "id": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP" }
```

- `QueryFilter` is `all` or `only { value }`. It is used for `market`, `underlying_mint`, `quote_mint` and `is_put`.
- `TimeBound` is `unbounded` or `bounded { value }`, in Unix seconds. It is used for `min_expiry_ts` and `max_expiry_ts`.
- `PageCursor` is `start` or `after { at, id }`. `at` is the ordering timestamp of the last row seen, in Unix seconds, and `id` is that row's id.
- `limit` is required on every paged request. A value outside the range is rejected with `invalid_limit`. The server never defaults or clamps it.

| Request | Limit range | Cursor `at` | Cursor `id` |
|---|---|---|---|
| `GetMakerPositions` | `1..=500` | `created_at` | position `pda` (base58) |
| `GetMyQuotes` with `scope: history` | `1..=1000` | `created_at` | `order_id` |
| `GetMyTrades` | `1..=200` | `confirmed_at` | trade `id` (UUID) |

Pages run newest first, ordered by (created second, id) descending. A response sets `has_more: true` when rows remain past the page. To read the next page, send the same request with `cursor` set to `after` the last row of the previous page. Stop when `has_more` is `false`. `MakerPositions.unrenderable` rows count as rows of the page: when the last row of a page is unrenderable, its `created_at` and `pda` are the cursor.

Do not use paged reads as a polling path.

### GetMakerPositions

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000011",
    "market": { "type": "all" },
    "underlying_mint": { "type": "only", "value": "So11111111111111111111111111111111111111112" },
    "status": ["open", "funded"],
    "min_expiry_ts": { "type": "bounded", "value": 1710000000 },
    "limit": 100,
    "cursor": { "type": "start" }
  }
}
```

`status` is a list of `open`, `funded`, `liquidated`, `settled`; `[]` matches every status. `min_expiry_ts` keeps positions whose market expiry is at or after the bound. The `GetMakerPositions` response sets `has_more` when the result was truncated at `limit`.

### GetMarketsForMaker

```json
{
  "type": "GetMarketsForMaker",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000012",
    "underlying_mints": ["So11111111111111111111111111111111111111112"],
    "quote_mints": [],
    "min_expiry_ts": { "type": "bounded", "value": 1710000000 },
    "max_expiry_ts": { "type": "unbounded" },
    "is_put": { "type": "only", "value": false }
  }
}
```

The request has no result cap. Use filters for dashboard views and avoid polling it.

### Markets payload

```json
{
  "type": "Markets",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000008",
    "markets": [
      {
        "pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
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

Same compact [`MarketInfo`](ws-common.md#marketinfo) shape as `Snapshot.markets`; active markets only. `status` is `active`, `closed` or `finalized`. `settlement_price` is `null` until the market settles.

### Expiries payload

```json
{
  "type": "Expiries",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-00000000000b",
    "expiries_ts": [1710600000, 1711200000]
  }
}
```

List of distinct market expiry timestamps (Unix seconds) matching the request filters.

### Tokens payload

```json
{
  "type": "Tokens",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-00000000000a",
    "tokens": [
      {
        "mint": "So11111111111111111111111111111111111111112",
        "symbol": "SOL",
        "decimals": 9,
        "oracle_feed_ids": ["<oracle feed id>"],
        "role": {
          "type": "underlying",
          "size_rule": { "min_size": 100000000, "max_size": 1000000000000, "step": 100000000 },
          "strike_tick": 500000000,
          "quote_mints": ["EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"]
        }
      },
      {
        "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "decimals": 6,
        "oracle_feed_ids": ["<oracle feed id>"],
        "role": { "type": "quote" }
      }
    ]
  }
}
```

Each mint once; see [`TokenInfo`](ws-common.md#tokeninfo). `role` is `quote` or `underlying`. Underlyings carry `size_rule`, `strike_tick` (1e9-scaled, like strikes) and `quote_mints`.

### MarketDescriptorInfo (from `MarketDescriptors`)

Token decimals live here and nowhere in `RfqBroadcast`: `RfqBroadcast.quantity`,
`QuoteFilled.quantity` and `size_rule` are underlying atoms. Cache
`underlying_decimals` / `quote_decimals` by `market.market_pda` (the Rust SDK's
`MarketDecimals` does this from `MarketDescriptors`).

```typescript
type MarketDescriptor = {
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

type MarketDescriptorInfo = {
  products: string[];
  market: MarketDescriptor;
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

### MakerPositions payload

`MakerPositions` returns `MakerPositionInfo[]`:

```json
{
  "type": "MakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000011",
    "positions": [
      {
        "revision": { "slot": 250000000, "transaction_index": 12, "instruction_index": 1 },
        "pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
        "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "terms": { "type": "covered_call", "strike": 160000000000 },
        "side": "sell",
        "state": "funded",
        "quantity": 1000000000,
        "price": 5000000000,
        "total_premium": 4975000,
        "created_at": 1710000042,
        "expiry_ts": 1710600000
      },
      {
        "revision": { "slot": 250900000, "transaction_index": 3, "instruction_index": 0 },
        "pda": "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
        "market": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "terms": { "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 },
        "side": "sell",
        "state": "settled",
        "settlement": {
          "type": "cash",
          "settlement_price": 110000000000,
          "maker_payoff": 10000000,
          "taker_return": 10000000
        },
        "quantity": 1000000000,
        "price": 5000000000,
        "total_premium": 4980000,
        "created_at": 1709900000,
        "expiry_ts": 1710000000
      }
    ],
    "unrenderable": [],
    "has_more": false
  }
}
```

Field semantics:
- `state` is a tag on the row. `open`, `funded` and `liquidated` rows carry no settlement fields. A `settled` row carries `settlement`, a tagged union:
  - `{ "type": "physical", "settlement_price", "is_otm" }` for covered calls and cash-secured puts. `is_otm` is derived from the market settlement price and the strike. A settlement price equal to the strike is out of the money.
  - `{ "type": "cash", "settlement_price", "maker_payoff", "taker_return" }` for spreads and cash puts, with both amounts in quote atoms. `taker_return` includes any donated collateral.
- `settlement_price` is 1e9-scaled.
- `revision` is `{ slot, transaction_index, instruction_index }` of the last applied lifecycle event. See [Position revisions](#position-revisions).
- `total_premium` is the net premium in quote atoms after the protocol fee. It is the authoritative money figure for the position. The wire `price` alone does not give it.
- `price` is the gross premium per 1 underlying unit (1e9 scale).
- `underlying_symbol` / `quote_symbol` / `underlying_decimals` / `quote_decimals` are pre-resolved token metadata so the maker does not need a separate `GetTokens` lookup.
- `unrenderable` lists stored positions whose market projection is incomplete: `{ pda, market, created_at, missing }`, where `missing` names the fields the projection could not supply. They count toward `has_more`. Retrying does not repair them.
- all amount fields are unsigned (`u64`); timestamps remain Unix seconds.

#### Position revisions

Every full position row (`MakerPositionInfo`, and `PositionInfo` in `PositionUpdated`) carries `revision`. Compare `(slot, transaction_index, instruction_index)` lexicographically. Apply a row only when its revision is greater than the stored revision for the same `pda`. Rows with equal revisions describe the same state on every read path (`PositionUpdated`, `GetMakerPositions`, `MmSummary`), regardless of filters.

`PositionUpdated` is coalesced per `pda` to the newest revision. It is not a log with one message per transition.

#### Position state by product

| Product | States it can reach |
|---|---|
| `covered_call`, `cash_secured_put` | `open`, `funded`, `liquidated`, `settled` (physical) |
| `call_spread`, `put_spread`, `cash_put` | `open`, `settled` (cash) |

### MyQuotes payload

```json
{
  "type": "MyQuotes",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000001",
    "has_more": false,
    "quotes": [{
      "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
      "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
      "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_symbol": "SOL",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_symbol": "USDC",
      "quote_decimals": 6,
      "terms": { "type": "covered_call", "strike": 160000000000 },
      "side": "sell",
      "price": 5000000000,
      "quantity": 1000000000,
      "valid_until": 1710000120,
      "state": { "type": "active", "rank": "best", "rfq_version": 1 },
      "created_at": 1710000010
    }]
  }
}
```

| `state.type` | Fields | Meaning |
|---|---|---|
| `active` | `rank` (`best` or `outbid`), `rfq_version` | In the active book. |
| `retained` | `rfq_version` | Non-selectable quote: a locked non-winner or a frozen refresh quote. It may return on rollback/requote unless removed. |
| `awaiting_signature` | `rfq_version` | Selected quote awaiting the taker signature. |
| `executing` | `rfq_version` | Selected quote whose execution is unresolved. |
| `historical` | `status` | History only; persisted projection, not live authority. |

Historical `status` is one of `submitted`, `filled`, `cancelled`, `replaced`, `expired`, `lost`. There is no top-level `status` or `selected`. `has_more` is always false for Live.

```json
{
  "type": "GetMyQuotes",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000013",
    "scope": { "type": "history", "limit": 200, "cursor": { "type": "start" } }
  }
}
```

A History row uses the same identity/amount fields with, for example, `"state": { "type": "historical", "status": "filled" }`. Live and History never mix in one response.

### GetOrderStatus / OrderStatus

```json
{ "type": "GetOrderStatus", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000002", "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a" } }
{ "type": "OrderStatus", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000002", "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a", "state": { "type": "pending" } } }
```

Available to the authenticated owner on `/maker`, `/maker/data`, `/taker`, and to the operator on `/vault`. Makers should use `/maker/data` to keep execution lookups off the quote connection. `state` is `pending`, `confirmed` (with `position_pda`), or `unknown`, encoded as a tagged object. See the shared [execution lookup contract](taker-api.md#orderstatus-server---taker).

A lost ACK leaves the command outcome unknown. Keep unresolved orders in strategy state across reconnect; neither an empty Live response, a missing position, a timeout nor `unknown` proves nonexecution. Execution lookup failures return `RequestError` with the request ID.

### MakerMarkets payload

```json
{
  "type": "MakerMarkets",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000012",
    "markets": [
      {
        "products": ["covered_call", "call_spread"],
        "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "expiry_ts": 1710600000,
        "is_put": false,
        "is_finalized": false,
        "underlying_symbol": "SOL",
        "quote_symbol": "USDC"
      }
    ]
  }
}
```

### MmSummary payload

`MmSummary` is the one-shot MM dashboard bootstrap response. It bundles caps, positions, active quotes, markets, and token metadata in a single payload so a freshly-connected market maker can render its UI without firing a fan-out of discovery requests.

Send `GetMmSummary` on `/maker/data`. Valid triggers are initial connect/auth,
reconnect, manual refresh, detected drift, and post-transaction reconciliation
when no owner-direct push is expected. Do not poll it. It costs one query token, but it is a
heavy read.

```json
{
  "type": "MmSummary",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000003",
    "maker_pda": "HYe9CryCnQ3ZviCwoG8X7REFoSBK7HCjrhWWMpNtTah1",
    "caps": {
      "request_id": "a1b2c3d4-0000-4000-8000-000000000003",
      "positions": { "current": 5, "limit": 100 },
      "notional": [
        {
          "underlying_mint": "So11111111111111111111111111111111111111112",
          "symbol": "SOL",
          "current": 50000000000,
          "limit": 200000000000
        }
      ],
      "balances": [
        {
          "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
          "symbol": "USDC",
          "decimals": 6,
          "deposited": 1000000000,
          "committed": 500000000,
          "available": 500000000
        }
      ],
      "quote_liabilities": [
        {
          "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
          "symbol": "USDC",
          "current_liability": 20000000,
          "policy": { "type": "limited", "max_liability": 100000000, "headroom": 80000000 }
        }
      ]
    },
    "positions": [],
    "active_quotes": [],
    "markets": [],
    "tokens": [],
    "computed_at": 1710000042,
    "unrenderable_positions": [],
    "positions_has_more": false
  }
}
```

Field semantics:
- `maker_pda`: on-chain maker account PDA (base58)
- `caps`: same shape as the [`MyCaps`](#mycaps-payload) response (echoes `request_id` from `GetMmSummary`)
- `positions`: `MakerPositionInfo[]` — see [MakerPositions payload](#makerpositions-payload). The summary holds the newest 500 positions across all statuses, ordered by (created second, `pda`) descending.
- `active_quotes`: `MakerQuoteInfo[]` — see [MyQuotes payload](#myquotes-payload)
- `markets`: `MakerMarketInfo[]` — see [MakerMarkets payload](#makermarkets-payload)
- `tokens`: `TokenInfo[]` — see [`TokenInfo`](ws-common.md#tokeninfo)
- `computed_at`: server-side snapshot timestamp (Unix seconds)
- `unrenderable_positions`: positions the summary holds but cannot describe; same shape as `MakerPositions.unrenderable`
- `positions_has_more`: `true` when the embedded `positions` were capped at 500. Retrieve the rest by paging `GetMakerPositions` as described under [Filters and pages](#filters-and-pages). The rest of the snapshot is complete.

All balance values are in their respective token atomic units (use `decimals` from `caps.balances` to format).

### ActiveRfqs payload

```json
{
  "type": "ActiveRfqs",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000005",
    "rfqs": [
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
        "terms": { "type": "covered_call", "strike": 160000000000 },
        "side": "sell",
        "quantity": 1000000000,
        "expires_at": 1710000050,
        "quotes_count": 3,
        "best_price": 5500000000,
        "order_options": [
          { "type": "covered_call", "strike": 150000000000 },
          { "type": "covered_call", "strike": 160000000000 }
        ]
      }
    ]
  }
}
```

`best_price` is `null` if no quotes have been submitted yet.

### GetMyTrades (maker -> server)

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000014",
    "limit": 50,
    "cursor": { "type": "after", "at": 1710000042, "id": "7c2e9a41-6b3d-4f58-9e1a-0d4b6f8a2c35" },
    "market": { "type": "only", "value": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi" }
  }
}
```

All four fields are required.

- `limit`: page size in `1..=200`.
- `cursor`: `start`, or `after` with the `confirmed_at` and `id` of the last trade of the previous page. The cursor is exclusive: the next page holds trades older than the cursor.
- `market`: a `QueryFilter` on the market PDA.

### MyTrades payload

```json
{
  "type": "MyTrades",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000014",
    "trades": [
      {
        "id": "7c2e9a41-6b3d-4f58-9e1a-0d4b6f8a2c35",
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "terms": { "type": "covered_call", "strike": 160000000000 },
        "side": "sell",
        "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
        "quantity": 1000000000,
        "price": 5000000000,
        "tx_signature": "<base58 transaction signature>",
        "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
        "confirmed_at": 1710000042
      }
    ],
    "has_more": true
  }
}
```

Field semantics:
- `price`: gross premium per 1 underlying unit (1e9 scale)
- `confirmed_at`: Unix timestamp seconds when the trade was confirmed on-chain
- `tx_signature` is always present; `position_pda` may be `null`
- `has_more`: `true` if there are more trades beyond this page; use the last trade's `confirmed_at` and `id` as the `after` cursor for the next page

`MyTrades` returns **confirmed trades only** — in-flight orders are visible via `GetOrderStatus` and the order push flow, not here. The cursor is therefore stable.

### TokenCaps payload

Platform-level OI and notional caps. Full schema (fields, unlimited semantics, `include_markets` behavior): [caps.md](caps.md#platform-caps).

### MyCaps payload

Per-maker position count, notional, balance and quote-liability caps. Full schema (fields, `decimals` rendering, unlimited semantics): [caps.md](caps.md#maker-caps).

### CheckQuote / QuoteCheck

Dry run of the cap checks a `Quote` of `quantity` at `price` on `market_id` (the market PDA) would face. Allowed on `/maker`, `/maker/data` and `/vault`. Nothing is reserved: a passing check is advice, not a hold. It checks spread collateral `C` against the quote-liability caps as well as premium capacity.

```json
{
  "type": "CheckQuote",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000015",
    "market_id": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "terms": { "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 },
    "side": "sell",
    "quantity": 1000000000,
    "price": 5000000000
  }
}
```

```json
{
  "type": "QuoteCheck",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000015",
    "verdict": { "type": "rejected", "cap_detail": { "maker_position_cap_exceeded": { "current": 10, "limit": 10 } } }
  }
}
```

`verdict` is `{ "type": "admissible" }` or `{ "type": "rejected", "cap_detail": CapError }`. `quantity` is in underlying base units; `price` is the gross price per underlying unit scaled by 1e9, as on `Quote`. `cap_detail` is the [CapError](caps.md#caperror-variants) a `QuoteRejected` would carry. `"cap_detail": "caps_unavailable"` means retry, not a cap. An unknown `market_id` returns an uncorrelated `Error` with `Generic` code `market_not_found`.

### Subscriptions payload

```json
{
  "type": "Subscriptions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000004",
    "channels": ["rfqs", "chain_events"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

An empty mint list means that dimension is unfiltered; filtered lists contain base58 mint addresses. The response does not include `products`.

---

## Broadcast events

Received when subscribed to the corresponding channel.

### TradeExecuted (channel: `trades`)

```json
{
  "type": "TradeExecuted",
  "data": {
    "trade": {
      "id": "7c2e9a41-6b3d-4f58-9e1a-0d4b6f8a2c35",
      "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "terms": { "type": "covered_call", "strike": 160000000000 },
      "side": "sell",
      "quantity": 1000000000,
      "price": 5000000000,
      "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
      "maker": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
      "tx_signature": "<base58 transaction signature>",
      "executed_at": 1710000042
    }
  }
}
```

`TradeExecuted` is a live hint: it can be lost, delayed or repeated. Merge it with `GetMyTrades` by `trade.id`. Do not derive correctness from event counts.

### StatsUpdate (channel: `stats`)

```json
{
  "type": "StatsUpdate",
  "data": {
    "stats": {
      "usd": { "notional_24h": "1500.00", "premium_24h": "49.00", "priced_trades_24h": 149 },
      "total_volume_24h": 1000000,
      "total_trades_24h": 150,
      "spread_trades_24h": 12,
      "total_price_24h": 500000,
      "active_markets": 12,
      "active_makers": 5,
      "active_rfqs": 3
    }
  }
}
```

`stats.usd` contains exact decimal USD strings: underlying notional and gross
premium. It is `null` when no USD valuation is available. Only confirmed orders with a captured oracle valuation contribute;
`priced_trades_24h` exposes coverage against `total_trades_24h`. Volume and price aggregates for covered calls and cash-secured puts exclude spreads; spread trades are counted in `spread_trades_24h`. The numeric volume/price aggregates are not dollars and can retain
the last representable value after overflow; prefer the USD object.

### PositionUpdated (owner-only push)

`PositionUpdated` is delivered **directly to the session of the maker that owns the position**. It is never broadcast and needs no subscription (there is no `positions` channel). It carries the full position row and the update type. See [Protocol Flow](protocol-flow.md).

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": { "slot": 250000000, "transaction_index": 12, "instruction_index": 1 },
      "pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
      "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_decimals": 6,
      "terms": { "type": "covered_call", "strike": 160000000000 },
      "side": "sell",
      "state": "funded",
      "quantity": 1000000000,
      "price": 5000000000,
      "total_premium": 4975000,
      "created_at": 1710000042,
      "expiry_ts": 1710600000
    },
    "update_type": "funded"
  }
}
```

`update_type` follows the row's state: `open` gives `created`, and `funded`, `liquidated` and `settled` map to the same names. Spreads and cash puts only produce `created` and `settled`. Apply the row only if its `revision` is newer than the stored one for the same `pda`. The row has no symbol fields. The push has no caps data; caps arrive in `MakerCapsUpdated`.

### MakerCapsUpdated (owner-only push)

Sent to the owner's sessions when the maker's capacity changes without a position event: a settled or liquidated position released capacity, or an administrator changed a limit.

```json
{
  "type": "MakerCapsUpdated",
  "data": {
    "caps_snapshot": {
      "positions": { "current": 6, "limit": 100 },
      "notional": [
        {
          "underlying_mint": "So11111111111111111111111111111111111111112",
          "symbol": "SOL",
          "current": 51000000000,
          "limit": 200000000000
        }
      ],
      "balances": [
        {
          "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
          "symbol": "USDC",
          "decimals": 6,
          "deposited": 1000000000,
          "committed": 550000000,
          "available": 450000000
        }
      ],
      "quote_liabilities": []
    },
    "reason": "lifecycle"
  }
}
```

`reason` is `lifecycle` or `policy`. `caps_snapshot` has the shape of the `caps` portion of [`MmSummary`](#mmsummary-payload), without `request_id`. Use it to keep local caps in sync without calling `GetMyCaps`.

### MarketCreated (channel: `markets`)

```json
{
  "type": "MarketCreated",
  "data": {
    "pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "underlying": "So11111111111111111111111111111111111111112",
    "quote": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "expiry_ts": 1710600000,
    "is_put": false,
    "products": ["covered_call", "call_spread"],
    "status": "active",
    "settlement_price": null
  }
}
```

### MarketFinalized (channel: `markets`)

```json
{
  "type": "MarketFinalized",
  "data": {
    "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "settlement_price": 160000000000
  }
}
```

### ChainEvent (channel: `chain_events`)

`ChainEvent` is an internally tagged enum. The `event_type` field inside `data` indicates the variant. Event-specific fields are flat alongside `event_type` (not nested in a sub-`data` object).

```json
{
  "type": "ChainEvent",
  "data": {
    "event_type": "PositionOpened",
    "signature": "<base58 transaction signature>",
    "instruction_index": 0,
    "slot": 250000000,
    "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "maker": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
    "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "terms": { "type": "covered_call", "strike": 160000000000 },
    "side": "sell",
    "quantity": 1000000000,
    "price": 5000000000,
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"
  }
}
```

Variants:

| `event_type` | Additional fields |
|--------|-------------------|
| `PositionOpened` | `market`, `maker`, `taker`, `terms`, `side`, `quantity`, `price`, `order_id` |
| `MarketCreated` | `market`, `underlying_mint`, `quote_mint`, `expiry_ts`, `is_put` |
| `MarketFinalized` | `market`, `settlement_price` |
| `MakerRegistered` | `owner`, `maker_pda`, `quote_signing` |
| `PositionSettled` | `position` |
| `PositionLiquidated` | `position` |

All variants require `signature` (tx signature, base58), `instruction_index` (instruction coordinate) and `slot` (Solana slot number). Deduplicate by `(signature, instruction_index)`; one transaction may contain multiple events.

---

## Delivery and recovery

When your outbound queue is full, a *critical* message is dropped and the server **closes the connection** (`4002 slow_consumer`), so its loss always surfaces as a disconnect. Best-effort broadcasts are dropped silently. For `/maker` the critical set is `RfqBroadcast`, the quote lifecycle, `QuoteRefreshRequested`, `MarketCreated`, `MarketFinalized`, `VaultCatalogChanged`, `VaultProjectionAdvanced`, and every message addressed to your session alone (responses, `PositionUpdated`, `MakerCapsUpdated`). `ChainEvent`, `StatsUpdate`, `RfqSkipped` and `IndicativePricesRequest` broadcasts are best-effort. Owner-direct `TradeExecuted` uses addressed delivery and closes the affected session on queue overflow. The full table is in [WebSocket common](ws-common.md#delivery-criticality).

The wire has no replay cursor or stream sequence number. The server does not replay events missed during a disconnect. Do not infer execution from absent events. SDK receiver sequence gaps are local delivery gaps, distinct from the wire.

### Recovery after reconnect

After every reconnect, authenticate (`ResumeAuth` or the full challenge) and run these steps.

Quote plane, `/maker`. The managed Rust SDK does this before it reports `Ready`:

1. Send `GetSubscriptions`.
2. If the server holds channels the client does not want, send `Unsubscribe` for them. Wait for `UnsubscribeAck`.
3. Send `Subscribe` with the desired products, channels and both mint lists (empty lists included). Wait for `SubscribeAck`. This replaces any mint scope that a resumed session restored.
4. Send `GetActiveRfqs` and `GetMyQuotes { scope: live }`. `ActiveRfqs` supplies each RFQ's taker and market PDA for the quote preimage. A `MyQuotes` response with `has_more: true` is a failed read, and the SDK retries the session.

Data plane, `/maker/data`:

5. Send `GetMmSummary`. Owner pushes start after it is answered.

These reads span live server state and the database. They are not an atomic snapshot. Then, in the strategy:

6. Rebuild the position book. Apply `MmSummary.positions` by revision. If `positions_has_more` is `true`, page `GetMakerPositions` with `status: []` until `has_more` is `false`. To read fewer rows, set `min_expiry_ts` to `bounded` at the earlier of two times: the earliest expiry among local rows that are not `settled` or `liquidated`, and the time of the last row or fill you applied. A position opened later has a later expiry. Keep a row only when its revision is newer.
7. Book money from position rows. `total_premium` is the net premium the taker received. The maker's cost is the gross premium, [`scaled_quote_amount(price, quantity)`](protocol-flow.md#units), and the fee is that cost minus `total_premium`. `QuoteFilled` has no fee or net field.
8. Reconcile unfinished orders. For each local order with no terminal outcome that is not in the Live set, send `GetOrderStatus` and wait for its response before sending the next. Stop at the first `RequestError`. Fix the cause and restart the loop from that order. `pending` and `unknown` keep the order as an open obligation. `confirmed` gives the `position_pda`; book it from the position row.
9. Drop signed quotes from before the disconnect. Sign new ones with a fresh `valid_until`, `nonce` and `order_id`.

An empty Live set, a timeout and `unknown` do not prove that an order did not execute.

`TradeExecuted` is a live hint, not authoritative. The DB-backed reads (`GetMmSummary`, `GetMakerPositions`, `GetMyTrades`) are the recovery authority. On any doubt, re-read rather than trusting the last live push. Also reconcile via `GetMmSummary` after your own transactions and when you detect drift, not on a timer.

---

## Indicative pricing workflow

Indicative pricing covers covered calls and cash-secured puts only.

### Request from taker-side cache (server -> maker)

```json
{
  "type": "IndicativePricesRequest",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "market": {
      "chain_id": 0,
      "program_id": "33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U",
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 1710600000,
      "is_put": false,
      "collateral_mint": "So11111111111111111111111111111111111111112",
      "settlement_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
    },
    "position_type": "covered_call",
    "strikes": [150000000000, 160000000000]
  }
}
```

### Response (maker -> server)

```json
{
  "type": "IndicativePricesResponse",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "position_type": "covered_call",
    "prices": [{ "strike": 150000000000, "price": 4200000000 }]
  }
}
```

---

## Errors

Two error message types: `Error` (connection-level) and `RequestError` (request-correlated).
See [WS common conventions](ws-common.md) "Error format" for the envelope.

Parsing rule for maker integrations:

1. Handle `RequestError` where you pass `request_id` (e.g. query requests, `Subscribe`).
2. Handle `Error` for connection-level failures (auth, WS errors, unknown request).
3. For both, switch on `ServerError.type`.
4. If `type == "Generic"`, switch on `data.code`.
5. The set of error codes is closed. A frame whose code is not in the catalog does not match the 1.0.0 schema; treat it as a version mismatch.

Every typed `ServerError` variant and its `data` is in the [ServerError catalog](ws-common.md#servererror-catalog).

Quote-specific validation failures are sent as `QuoteRejected` with a typed `reason`. See the `QuoteRejected` section above for its variants.

`Generic` codes are listed, with their frames, in the [WS error code catalog](ws-common.md#ws-error-codes); match on `code`. The codes introduced by products and queries are:

| Code | Meaning |
|---|---|
| `unsupported_protocol_version` | `Hello.protocol_version` is not `1.0.0`; the server closes with `4006` |
| `invalid_products` | `Subscribe.products` is empty |
| `invalid_limit` | A page `limit` is outside the range of its request |
| `product_not_allowed` | The pair's policy does not allow the RFQ's product |
| `taker_buy_unsupported` | `side` is `buy` |
| `spread_invalid_strikes` | Invalid spread strikes |
| `spread_tick_violation` | A strike is off the underlying mint's strike tick |
| `spread_premium_invalid` | Price or gross premium outside `0 < price < width`, `0 < gross < C` |
| `quote_identity_mismatch` | Quote product, side or spread terms differ from the RFQ |
| `missing_tick_for_underlying_mint` | The underlying mint has no strike tick |

Operational handling recommendations:
- for `rate_limited`: retry with exponential backoff + jitter. Quote/query-bucket rejects are soft (connection stays open); a message-rate-bucket breach closes the connection — see Operational defaults.
- for `session_replaced`: another connection replaced this session — stop; do not auto-reconnect in a loop.
- for `parse_error`: treat as payload bug and stop retries until fixed.
- for `too_many_parse_errors`: reconnect only after fixing payload/codec mismatch.
- for `unsupported_protocol_version`: upgrade the client. Retrying does not help.

---

## Enums reference

### RfqCloseReason

| Value | Meaning |
|-------|---------|
| `expired` | RFQ expired without fill |
| `taker_cancelled` | Taker cancelled the RFQ |
| `filled` | RFQ was filled on-chain |
| `market_expired` | The underlying market expired |
| `ladder_timeout` | Ladder execution timeout |

### QuoteFinalStatus (in `RfqClosed.your_quote.status`)

| Value | Meaning |
|-------|---------|
| `expired` | Quote expired before selection |
| `outbid` | Another maker won |
| `cancelled` | Quote was cancelled |
| `filled` | This quote was selected and filled |

### QuoteCancelReason (in `QuoteCancelled.reason`)

| Value | Meaning |
|-------|---------|
| `requested` | Maker requested cancellation |
| `risk_check` | Server-side risk check triggered |
| `rfq_accepted` | RFQ accepted another quote |
| `maker_disconnected` | Session had `on_disconnect: "cancel_quotes"` and disconnected |

### QuoteExpiryReason (in `QuoteRefreshRequested.reason` and `QuoteExpired.reason`)

| Value | Meaning |
|-------|---------|
| `expired` | The quote's validity ended |
| `expiring_soon` | The quote reached its refresh point before the settlement-buffer cutoff |

### RfqAvailableAgainReason

| Value | Meaning |
|-------|---------|
| `signature_timeout` | Taker did not sign in time |
| `tx_failed` | On-chain transaction failed |
| `tx_build_failed` | Transaction could not be built |

When `reason = "tx_build_failed"`, a corresponding `Error` or `RequestError` is also sent to
the taker with `generic.code = "tx_build_failed"` and a `message` field
describing what went wrong.

### OnDisconnect (in `Hello`)

| Value | Meaning |
|-------|---------|
| `cancel_quotes` | The server removes the connection's active and retained non-winning quotes on disconnect |
| `keep_quotes` | Resting quotes stay live after disconnect |

### TakerSide (`side`)

| Value | Meaning |
|-------|---------|
| `sell` | The taker is the option writer. The only executable side. |
| `buy` | Rejected with `taker_buy_unsupported`. |

### PositionStatus (in `GetMakerPositions.status`)

| Value | Meaning |
|-------|---------|
| `open` | Position created on-chain but maker has not yet escrowed settlement asset. |
| `funded` | Maker called `DepositFundsToPosition`. Settlement asset is escrowed. |
| `liquidated` | Unfunded ITM position seized by a third-party liquidator. |
| `settled` | Position settled at expiry; assets distributed per ITM/OTM outcome, or paid in cash. |

Position status is a lowercase snake_case string on the wire. `funded` and `liquidated` apply to covered calls and cash-secured puts only.

### PositionState (on position rows)

| `state` | Extra fields |
|-------|---------|
| `open` | none |
| `funded` | none |
| `liquidated` | none |
| `settled` | `settlement`: `physical { settlement_price, is_otm }` or `cash { settlement_price, maker_payoff, taker_return }` |

### PositionUpdateType (in `PositionUpdated`)

| Value | Row state |
|-------|---------|
| `created` | `open`. Position opened on-chain. |
| `funded` | `funded`. Maker called `DepositFundsToPosition`. |
| `liquidated` | `liquidated`. Unfunded ITM position liquidated by a third party. |
| `settled` | `settled`. Position settled at expiry. |

Physical transitions: `created` -> `funded` -> `settled`. An unfunded position (`created`) goes to `settled` if OTM and to `liquidated` if ITM. A funded position cannot be liquidated. Cash products go `created` -> `settled`.

See [Protocol Flow](protocol-flow.md) for the physical scenarios.

### RateLimitReason (in typed `rate_limit` error)

| Value | Meaning |
|-------|---------|
| `too_many_active_rfqs_total` | Platform-wide RFQ limit reached |
| `too_many_active_rfqs_per_taker` | Per-taker active RFQ limit reached |
| `too_many_quotes_per_rfq` | Max quotes per RFQ reached |
| `too_many_failed_orders_per_rfq` | `AcceptQuote` refused: too many failed orders on this RFQ are still unresolved. Create a new RFQ |
| `too_many_sessions_per_user` | Too many concurrent sessions |

### CapError

Used in the typed `cap` error, `QuoteRejected.reason`, `RfqSkipped.cap_detail` and `QuoteCheck.verdict`.

Full variant catalog (`token_oi_cap_exceeded`, `market_oi_cap_exceeded`, `maker_position_cap_exceeded`, `maker_notional_cap_exceeded`, `maker_insufficient_balance`, `quote_notional_cap_exceeded`, `maker_quote_notional_cap_exceeded`, `quote_liability_cap_exceeded`, `maker_quote_liability_cap_exceeded`, `caps_unavailable`) with fields and rejection semantics: [caps.md](caps.md#caperror-variants).

---

## Quirks and constraints

Details that are easy to miss in the schemas.

### Multiple quotes per RFQ

When `RfqBroadcast.order_options` lists several strikes of a covered call or cash-secured put, you can quote each strike independently — each quote needs a distinct `order_id` and `nonce`. Submitting a new quote on the same `(rfq_id, terms)` pair replaces your prior quote on that pair; `QuoteAcknowledged.replaced_order_id` tells you which one was displaced. Across all makers, `max_quotes_per_rfq` from `GET /api/v1/venue` bounds the quotes per RFQ.

### `order_options` validation

For a covered call or cash-secured put, the quote's strike must be one the RFQ offers. Anything else returns `invalid_strike`. A spread or cash-put quote must carry the RFQ's `terms` exactly, or it returns `quote_identity_mismatch`.

### Subscribe / Unsubscribe

Both require a `request_id`. The ack carries the *diff* of channels added or removed by this call, not the full subscription. For the full set, query `GetSubscriptions`.

### Sessions and disconnects

One active quote connection per maker pubkey. A new quote connection replaces the previous quote connection; the separate data connection is for reads. On replacement, the displaced session gets `Error { generic.code = "session_replaced" }` and is closed.

The client chooses cancel-on-disconnect with `Hello.on_disconnect`. The managed quote SDK examples use `cancel_quotes`. Selected and executing quotes remain obligations. Reconcile `GetMyQuotes { scope: live }` and execution status after reconnect.

A resumed maker auth session restores server-side routing and mint scope. After fresh auth, establish subscriptions again. The managed SDK restores its own desired target after either path by sending `Subscribe` with both mint lists, including empty lists.

### Max message size

32 KB inbound. A larger frame closes the connection without an error frame.

### Nonce

`nonce` is a `u64` baked into the order-id preimage. The server doesn't enforce uniqueness on the nonce itself — it's validated only as part of `sha256(preimage)` matching the submitted `order_id`.

### `order_id` hex

Accepted with or without `0x` prefix; case-insensitive. Must decode to exactly 32 bytes.

### Message ordering after auth

After `AuthSuccess`, the server sends `Snapshot` before any broadcast events. You can safely initialise state from `Snapshot` before processing push messages.

### `is_taker_buy`

In the order-id preimage, `is_taker_buy` is `0` for side `sell`. Side `buy` sets it to `1` and is rejected with `taker_buy_unsupported`.

### `gross_price` in preimage

The `gross_price` at preimage offset 94 is the same value as `price` in the `Quote` message — gross premium per one underlying unit, 1e9 scale. `total_premium` in position data is the net amount after the protocol fee, not derivable from the wire `price` alone. The fee formula is under [Cash settlement and fees](#cash-settlement-and-fees).

---

## Related

- [Maker quickstart](../quickstart/maker-quickstart.md)
- [Maker wire examples](../quickstart/maker-wire-examples.md)
- [WS common conventions](ws-common.md)
