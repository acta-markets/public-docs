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

### Hello (first message)

Protocol constraints:

- Current server protocol: `protocol_version=1.0.0`
- Current server minimum supported version: `min_supported_version=1.0.0`
- `Hello` MUST be the first client message
- `Hello` timeout: `5000ms`
- Version compatibility check is semver-based: client `protocol_version >= min_supported_version`

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

### Welcome (server -> client)

```json
{
  "type": "Welcome",
  "data": {
    "protocol_version": "1.0.0",
    "server_version": "0.1.0",
    "min_supported_version": "1.0.0",
    "enabled_features": [],
    "server_time_unix_ms": 1710000000000,
    "min_quote_validity_seconds": 60,
    "settlement_buffer_seconds": 50
  }
}
```

`server_time_unix_ms` is required. Use it for clock skew estimation (see [WS common conventions](ws-common.md)).

`min_quote_validity_seconds` and `settlement_buffer_seconds` are the venue quote window, the same values as `GET /api/v1/venue`: `valid_until` must be at least `now + min_quote_validity_seconds`, and a quote stays executable only while `valid_until - settlement_buffer_seconds > now`. The Rust `QuoteBuilder::welcome(&welcome)` applies both.

If the client's `protocol_version` is incompatible, the server sends `VersionMismatch` instead of `Welcome` and closes the connection.

`features` is an opt-in list; the server echoes the subset it enabled in `Welcome.enabled_features`. A feature that is absent from `enabled_features` is not active, so a client that requires one must check the echo before quoting.

| Feature | Effect |
|---------|--------|
| `quote_expired` | The server emits explicit `QuoteExpired` events instead of silent expiry. |
| `cancel_on_disconnect` | On an actual disconnect (socket close or liveness timeout), the server removes active and retained non-winning quotes still owned by that physical session, including retained quotes in locked RFQs. Ownership takeover preserves quotes; late cleanup of the replaced connection cannot cancel the new owner's book. Quotes a taker has already accepted (`QuoteSelected`) stay locked and settle or expire on their own timers. Without this feature, resting quotes survive a disconnect and can be filled while the maker is offline. Session resume does not restore cancelled quotes; re-quote after reconnecting. |

### Subscribe

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "uuid",
    "channels": ["rfqs", "chain_events"],
    "underlying_mints": ["UnderlyingMintBase58"],
    "quote_mints": ["QuoteMintBase58"]
  }
}
```

`request_id` is required. The server always responds with `SubscribeAck` echoing the
`request_id` and the channels that were newly added.

`underlying_mints` and `quote_mints` filter market broadcasts. Omitting both preserves the current scope (unrestricted on a fresh session). Providing either replaces both: an omitted peer or an empty list means unrestricted for that dimension. To preserve one dimension while changing the other, send both lists.

### Connection policy defaults

- Auth deadline (after challenge issued): `15s`
- Idle timeout: `90s`
- Server WS ping interval: `30s`
- Max consecutive parse errors before close: `3`
- Inbound message size: `32 KiB`
- Message rate limit per connection: token bucket `30 msg/s` sustained, `60` burst; exceeding it closes the connection with `4003 rate_limited`. Quote-plane messages (`Quote`, `ReplaceQuote`, `BatchQuotes` within `max_batch_quotes`, `CancelQuote`, `CancelAllQuotes`) do not count toward it, so quoting and cancelling never close the socket; an oversized `BatchQuotes` does count.
- Quote rate limit per connection: token bucket `50 quote tokens/s` sustained, `100` burst — covers `Quote`/`ReplaceQuote`/`BatchQuotes`/`CancelQuote`/`CancelAllQuotes` (maker plane); `BatchQuotes` costs `max(1, quotes.length)` tokens. Exceeding it is a soft reject and the connection stays open: `Quote` / `ReplaceQuote` get `QuoteRejected` (per quote in `BatchQuotesAck`) with reason `rate_limited`; `CancelQuote` / `CancelAllQuotes` get a `RequestError` with `Generic` code `rate_limited`, and nothing is cancelled.
- Query rate limit per connection: token bucket `20 query tokens/s` sustained, `40` burst
- `BatchQuotes` hard max: `50` quote elements; cost is `max(1, quotes.length)` quote tokens

### Snapshot (server -> maker)

```json
{
  "type": "Snapshot",
  "data": {
    "markets": [
      {
        "pda": "MarketPdaBase58",
        "underlying": "UnderlyingMintBase58",
        "quote": "QuoteMintBase58",
        "expiry_ts": 1710600000,
        "is_put": false,
        "status": "active"
      }
    ]
  }
}
```

`Snapshot.markets` lists the active markets as compact [`MarketInfo`](ws-common.md#marketinfo) (`pda`, `underlying`, `quote`, `expiry_ts`, `is_put`, `status`). This is a different DTO from the `MarketDescriptor` in `RfqBroadcast` which has additional fields (`chain_id`, `program_id`, `market_pda`, `underlying_mint`, `quote_mint`, `collateral_mint`, `settlement_mint`). Use `GetMarketDescriptors` for the full market descriptor.

---

## Message index

### Client -> Server

- `Hello`, `AuthChallenge`, `Logout`
- `Subscribe`, `Unsubscribe`, `Ping`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `Quote`, `ReplaceQuote`, `BatchQuotes`, `CancelQuote`, `CancelAllQuotes`
- `IndicativePricesResponse`
- `GetOrderStatus`, `GetMyQuotes`, `GetMakerPositions`, `GetMarketsForMaker`, `GetMmSummary`, `GetMyTrades`, `GetTokenCaps`, `GetMyCaps`, `CheckQuote`, `GetSubscriptions`
- `GetActiveRfqs`, `GetMarkets`, `GetMarketDescriptors`, `GetExpiries`, `GetTokens`

### Server -> Maker (direct)

- `Welcome`, `VersionMismatch`, `LogoutSuccess`
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
| `RfqBroadcast` | `rfqs` channel, within the mint scope | no | sessions with `sign_maker_quotes`, no subscription |
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
`OrderConfirmed`. `PositionUpdated` and `MakerCapsUpdated` reach only the
owner's `/maker` session; a client that reads on `/maker/data` alone recovers
them with `GetMakerPositions`, `GetMyCaps` or `GetMmSummary`.

---

## Quote flow

### RfqBroadcast (server -> maker)

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "uuid",
    "market": {
      "chain_id": 0,
      "program_id": "ProgramIdBase58",
      "market_pda": "MarketPdaBase58",
      "underlying_mint": "UnderlyingMintBase58",
      "quote_mint": "QuoteMintBase58",
      "expiry_ts": 1710600000,
      "is_put": false,
      "collateral_mint": "CollateralMintBase58",
      "settlement_mint": "SettlementMintBase58"
    },
    "position_type": "covered_call",
    "strike": 160000000000,
    "quantity": 1000000000,
    "expires_at": 1710000050,
    "taker": "TakerPubkeyBase58",
    "order_options": [
      {
        "strike": 150000000000
      },
      {
        "strike": 160000000000
      }
    ],
    "sent_at_unix_ms": 1710000000123
  }
}
```

`strike` is the primary strike from the RFQ request. `order_options` lists all strikes available for quoting (always includes the primary `strike`). When `order_options` is present, the maker must pick a strike from that set. When `order_options` is empty, only the top-level `strike` is valid. Submitting a strike outside this set results in `QuoteRejected { reason: "invalid_strike" }`.

### RfqSkipped (server -> maker)

Sent instead of `RfqBroadcast` when cap limits pre-filter the maker. `reason` is a cap error code — full catalog in [CapError](#caperror-in-typed-cap-error-and-rfqskippedreason).

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "uuid",
    "market_id": "MarketPdaBase58",
    "quantity": 1000000000,
    "reason": "token_oi_cap_exceeded"
  }
}
```

### Quote (maker -> server)

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "uuid",
    "strike": 160000000000,
    "price": 50000000,
    "valid_until": 1710000350,
    "nonce": 42,
    "order_id": "0x...64chars",
    "signature": "base58sig"
  }
}
```

Rules:
- `valid_until` MUST be >= `now + 100` seconds (server rejects shorter expiries with `quote_expiry_too_short`)
- `valid_until` MUST be <= the market's `expiry_ts` (later expiries are rejected with `market_expired`)
- The server applies a **settlement buffer** (`settlement_buffer_seconds` from `GET /api/v1/venue`, currently 50). Your quote is active for trading until `valid_until` minus that buffer. Set `valid_until` to at least `rfq.expires_at + min_quote_validity_seconds` (currently 60) so the quote stays tradable for the whole auction; the accepted floor is `now + min_quote_validity_seconds`.
- `order_id = sha256(preimage182)`
- maker signs only 32-byte `order_id`
- if `order_options` is present, strike must be from that set

Before building the preimage, derive the market PDA locally using Solana PDA seeds
`["market", underlying_mint[32], quote_mint[32], expiry_ts_u64_le[8], is_put_u8[1]]`
under the Acta program ID from your deployment configuration. Require chain ID
`0`, that configured program ID, and equality with `market_pda`. Also check that
`position_type` matches `is_put`: collateral/settlement are underlying/quote for
calls and quote/underlying for puts. Do not copy an endpoint-selected program ID
into the derivation. Rust `RfqBinding::from_broadcast` and TS
`buildSignedQuoteFromRfq` perform these checks before signing. Strategy mint
allowlists and pricing inputs remain your responsibility; PDA verification does
not certify an endpoint's prices or mint decimals.

Canonical `order_id` preimage layout (self-contained):
- preimage is exactly 182 bytes
- numeric fields use little-endian encoding
- hash is `sha256(preimage)` and serialized as 32-byte `order_id`

Preimage fields (offset, size):
- `0,4` -> `domain_tag` (`"ACTA"`)
- `4,8` -> `chain_id` (`u64`, current value `0` for Solana)
- `12,32` -> `program_id` (bytes)
- `44,1` -> `is_taker_buy` (`u8`, current flow uses `0`)
- `45,1` -> `position_type` (`u8`, `0=covered_call`, `1=cash_secured_put`)
- `46,32` -> `market` (bytes)
- `78,8` -> `strike` (`u64`)
- `86,8` -> `quantity` (`u64`)
- `94,8` -> `gross_price` (`u64`)
- `102,8` -> `valid_until` (`u64`, unix seconds)
- `110,32` -> `maker` (bytes)
- `142,32` -> `taker` (bytes)
- `174,8` -> `nonce` (`u64`)

### ReplaceQuote (maker -> server)

Atomic cancel-and-resubmit for a specific quote. Cancels the quote identified by `old_order_id` and submits a new quote in a single operation. The old quote is removed **only if** the new quote passes all validation (caps, signature, `valid_until`). If validation fails, the old quote remains active.

```json
{
  "type": "ReplaceQuote",
  "data": {
    "old_order_id": "0x...64chars",
    "rfq_id": "uuid",
    "strike": 160000000000,
    "price": 55000000,
    "valid_until": 1710000350,
    "nonce": 43,
    "order_id": "0x...new64chars",
    "signature": "base58sig"
  }
}
```

`old_order_id` identifies the specific quote being replaced. Server responds with `QuoteAcknowledged` (with `replaced_order_id`) on success, or `QuoteRejected` / `Error` on failure.

**Errors:**
- `QuoteRejected` with standard reasons (`invalid_strike`, `cap_exceeded`, etc.) — new quote failed validation; old quote remains active
- `Error { type: "QuoteLocked" }` — old quote is locked in `PendingSignature`/`Enqueued` (cannot replace)
- `Error { type: "QuoteNotFound" }` — `old_order_id` not found (race: already cancelled/expired)

### BatchQuotes (maker -> server)

Submit multiple quotes in one message. A batch must contain at most `ws.max_batch_quotes` quotes (default 50). The server charges one quote-rate-limit token per quote element; an empty batch costs one token. Within an accepted batch, each quote is validated like a standalone `Quote`. The server responds with one `BatchQuotesAck` containing `results[]` — one entry (`QuoteAcknowledged` or a `QuoteRejected` reason) per quote that reached quote-level validation. **Correlate results by `order_id`, not by position:** a quote rejected at pre-kernel validation (bad signature, expired, unregistered maker) is reported as an inline error and may be omitted from `results[]`. Partial success is allowed. For an implicit same-strike replacement, the aggregated `QuoteAcknowledged` leaves `replaced_order_id` null; the replaced order id is delivered on the asynchronous per-quote `QuoteAcknowledged` event.

```json
{
  "type": "BatchQuotes",
  "data": {
    "quotes": [
      {
        "rfq_id": "uuid-1",
        "strike": 150000000000,
        "price": 45000000,
        "valid_until": 1710000350,
        "nonce": 42,
        "order_id": "0x...64chars-a",
        "signature": "base58sig-a"
      },
      {
        "rfq_id": "uuid-2",
        "strike": 160000000000,
        "price": 50000000,
        "valid_until": 1710000350,
        "nonce": 43,
        "order_id": "0x...64chars-b",
        "signature": "base58sig-b"
      }
    ]
  }
}
```

Each entry in `quotes` has the same schema as a standalone `Quote` message. Quotes are processed sequentially — a rejection of one quote does not block subsequent quotes.

### QuoteRejected (server -> maker)

`QuoteRejected` is sent for quote-specific validation failures and carries a typed `reason` for programmatic handling.

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "reason": "invalid_strike",
    "message": "optional detail"
  }
}
```

`message` is optional and may be omitted. A cap rejection carries the failing
dimension inside the reason instead of a bare string, so it cannot arrive
without its detail:

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
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
| `unauthorized` | Session is not an authenticated maker, or a vault maker that may not trade or hold this position (see `message`) |
| `invalid_strike` | Strike not in `order_options` set |
| `market_expired` | Target market has expired |
| `quote_expiry_too_short` | `valid_until` below minimum |
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
    "rfq_id": "uuid",
    "strike": 160000000000,
    "min_valid_until": 1710000350,
    "reason": "expiring_soon"
  }
}
```

Identified by `(rfq_id, strike)`, not `order_id`. If quoting multiple strikes per RFQ, maintain a local `(rfq_id, strike) -> order_id` mapping to know which quote to refresh.

`min_valid_until` includes the settlement buffer — the refreshed quote must have `valid_until >= min_valid_until` to satisfy both the buffer and the refresh margin.

### QuoteSelected (server -> maker)

```json
{
  "type": "QuoteSelected",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "taker": "TakerPubkeyBase58",
    "price": 50000000,
    "quantity": 1000000000,
    "strike": 160000000000,
    "signature_deadline": 1710000040
  }
}
```

### QuoteFilled (server -> maker)

```json
{
  "type": "QuoteFilled",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "market_pda": "MarketPdaBase58",
    "expiry_ts": 1710489600,
    "position_type": "covered_call",
    "taker": "TakerPubkeyBase58",
    "price": 50000000,
    "quantity": 1000000000,
    "strike": 160000000000,
    "position_pda": "PositionPdaBase58",
    "tx_signature": "5eyk...base58sig",
    "filled_at": 1710000042
  }
}
```

`QuoteFilled` is the fill-details event for the winning maker. `market_pda`,
`expiry_ts` (market expiry, Unix seconds; `null` if the server no longer knows the market, so resolve it from `market_pda`) and `position_type` identify the
market without the RFQ, so a fill can be attributed after a restart; mint
decimals for the market come from `GetMarketDescriptors`. `quantity` is in
underlying atoms and `price` is 1e9-scaled per underlying token, so the gross
premium (before protocol fees) in quote atoms is `floor(price * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`
(the contract's rounding; the Rust SDK's `total_premium_quote_atoms` and the TS
SDK's `totalPremiumQuoteAtoms` compute it).
`RfqClosed` is the terminal RFQ event.

Current runtime behavior for successful fills:
- winner maker receives `RfqClosed` and then `QuoteFilled`; accept fill details after the RFQ has closed
- taker receives `OrderConfirmed` and then `RfqClosed`
- close RFQ state only on `RfqClosed`; process `QuoteFilled` as fill-details information
- `RfqClosed.your_quote` / `RfqClosed.winner` are optional fields and may be omitted

### QuoteAcknowledged (server -> maker)

```json
{
  "type": "QuoteAcknowledged",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "replaced_order_id": "0x...64chars"
  }
}
```

`replaced_order_id` is present when the new quote replaced a prior quote from the same maker on the same (rfq, strike). Omitted when `null`.

### QuoteBestStatus (server -> maker)

```json
{
  "type": "QuoteBestStatus",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "is_best": true,
    "current_best_price": 50000000
  }
}
```

`is_best`: whether the maker's quote is currently the highest premium.
`current_best_price`: the best price on the RFQ (may be another maker's price). Omitted when `null`.

### QuoteOutbid (server -> maker)

```json
{
  "type": "QuoteOutbid",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "your_price": 50000000,
    "current_best_price": 55000000
  }
}
```

`your_price`: the maker's current quote price.
`current_best_price`: the new best price that outbid the maker. Omitted when `null`.

### QuoteExpired (server -> maker)

Feature-gated: only sent if the client includes `"quote_expired"` in `Hello.features` and the server echoes it in `Welcome.enabled_features`.

```json
{
  "type": "QuoteExpired",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "reason": "valid_until_passed"
  }
}
```

### QuoteCancelled (server -> maker)

```json
{
  "type": "QuoteCancelled",
  "data": {
    "rfq_id": "uuid",
    "order_ids": ["0x...64chars"],
    "reason": "requested",
    "cancelled_at": 1710000025
  }
}
```

`reason` values: `requested` (maker requested), `risk_check` (server-side risk), `rfq_accepted` (RFQ accepted another quote), `maker_disconnected` (the session enabled `cancel_on_disconnect` and disconnected; sent to the session that is gone, so a reconnecting maker sees the cancellation through `GetMyQuotes`).

### RfqClosed (server -> maker)

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "uuid",
    "rfq_version": 3,
    "reason": "filled",
    "your_quote": {
      "order_id": "0x...64chars",
      "status": "outbid",
      "price": 50000000
    },
    "winner": {
      "maker": "WinnerMakerPubkeyBase58",
      "price": 55000000,
      "tx_signature": "5eyk...base58sig"
    },
    "closed_at": 1710000042
  }
}
```

`reason` values: `expired`, `taker_cancelled`, `filled`, `market_expired`, `ladder_timeout`.

`your_quote`: present when the maker had an active quote on this RFQ. `your_quote.status` values: `expired`, `outbid`, `cancelled`, `filled`.

`winner`: present when the RFQ was filled. `winner.tx_signature` may be `null` if the transaction has not confirmed yet.

Both `your_quote` and `winner` are optional and may be omitted.

### RfqAvailableAgain (server -> maker)

```json
{
  "type": "RfqAvailableAgain",
  "data": {
    "rfq_id": "uuid",
    "rfq_version": 2,
    "reason": "signature_timeout",
    "available_again_at": 1710000045
  }
}
```

Sent when a previously locked RFQ becomes available for new quotes (e.g. taker failed to sign in time).

`reason` values: `signature_timeout`, `tx_failed`, `tx_build_failed`.

### VersionMismatch (server -> maker)

Sent instead of `Welcome` when the client's `protocol_version` is not compatible.

```json
{
  "type": "VersionMismatch",
  "data": {
    "requested_version": "2.0.0",
    "server_version": "1.0.0",
    "min_supported_version": "1.0.0",
    "message": "Client version 2.0.0 is not supported"
  }
}
```

Connection is closed after this message.

---

## Quote management

### CancelQuote (maker -> server)

```json
{
  "type": "CancelQuote",
  "data": {
    "rfq_id": "uuid",
    "request_id": "uuid"
  }
}
```

Cancels **all** of this connection's active quotes on the given RFQ (across all strikes). There is no per-order_id cancel — use `CancelQuote` per rfq_id. The server responds once the cancellation is applied with `CancelQuoteAck { request_id, rfq_id, cancelled_order_ids }`, or a correlated `RequestError` (for example `RfqNotFound`, `RfqNotActive`, `QuoteLocked`). `QuoteCancelled` is a lifecycle notification, not the command receipt.

### CancelQuoteAck (server -> maker)

```json
{
  "type": "CancelQuoteAck",
  "data": {
    "request_id": "uuid",
    "rfq_id": "uuid",
    "cancelled_order_ids": ["0x...64chars"]
  }
}
```

On an Active RFQ an empty list is an idempotent no-op: the connection has no quote to remove there. `PendingSignature` and `Enqueued` reject single-RFQ cancellation as a whole with `QuoteLocked`, even for a non-winning maker.

### CancelAllQuotes (maker -> server)

```json
{
  "type": "CancelAllQuotes",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58"
  }
}
```

`market` is optional. If provided, it filters cancellation to that market; otherwise all markets are considered. CancelAll removes this connection's active and retained non-winning quotes, including those inside locked RFQs. Removed retained quotes cannot return on rollback. Selected/awaiting-signature and executing orders are preserved. Cancel-on-disconnect applies the same cleanup once the server processes the disconnect. Neither operation is an account-wide persistent halt.

The server responds with `CancelAllQuotesAck` once the cancellation is applied. An invalid `market` returns `RequestError` with `InvalidMarket`; it does not broaden cancellation to all markets.

### CancelAllQuotesAck (server -> maker)

```json
{
  "type": "CancelAllQuotesAck",
  "data": {
    "request_id": "uuid",
    "cancelled_count": 0,
    "cancelled_order_ids": []
  }
}
```

`cancelled_count` equals `cancelled_order_ids.length`; both report the orders actually removed. An empty list means this command removed no quotes; it does not establish that no selected or executing obligations remain. Lifecycle `QuoteCancelled` messages are separate from the receipt and must not complete another request's await.

Quote, batch, replacement and cancellation commands from one connection retain FIFO order from the SDK writer through the server's processing. Commands before a cancellation may execute first; commands sent afterward may create new quotes. Cancellation is not a persistent account halt.

If the connection drops before the ACK, retain an unknown outcome and reconcile instead of automatically replaying the command.

### Dynamic subscription management

Modify subscriptions incrementally:

```json
{ "type": "AddMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"], "quote_mints": ["MintBase58"] } }
{ "type": "RemoveMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"] } }
{ "type": "AddChannels", "data": { "request_id": "uuid", "channels": ["trades"] } }
{ "type": "RemoveChannels", "data": { "request_id": "uuid", "channels": ["stats"] } }
```

All four respond with `SubscriptionUpdated` containing the subscription state:

```json
{
  "type": "SubscriptionUpdated",
  "data": {
    "request_id": "uuid",
    "channels": ["rfqs", "trades"],
    "underlying_mints": ["MintBase58"]
  }
}
```

In subscription responses, an omitted mint list means that dimension is unfiltered. In requests, omission/null preserves the current filter; an explicit empty list clears it.

### Unsubscribe (maker -> server)

```json
{
  "type": "Unsubscribe",
  "data": {
    "request_id": "uuid",
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
{ "type": "GetMyQuotes", "data": { "request_id": "uuid", "scope": "live" } }
{ "type": "GetOrderStatus", "data": { "request_id": "uuid", "order_id": "0x...64chars" } }
{ "type": "GetMakerPositions", "data": { "request_id": "uuid" } }
{ "type": "GetMarketsForMaker", "data": { "request_id": "uuid" } }
{ "type": "GetMmSummary", "data": { "request_id": "uuid" } }
{ "type": "GetSubscriptions", "data": { "request_id": "uuid" } }
{ "type": "GetActiveRfqs", "data": { "request_id": "uuid" } }
{ "type": "GetMyTrades", "data": { "request_id": "uuid" } }
{ "type": "GetTokenCaps", "data": { "request_id": "uuid" } }
{ "type": "GetMyCaps", "data": { "request_id": "uuid" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "uuid" } }
{ "type": "GetExpiries", "data": { "request_id": "uuid" } }
{ "type": "GetTokens", "data": { "request_id": "uuid" } }
```

Request notes:
- `GetMarkets`, `GetMarketDescriptors`, `GetExpiries` and `GetTokens` answer from the [reference snapshot](ws-common.md#reference-data): **tradable** markets only — non-finalized, non-disabled, and before the pre-expiry trading cutoff. Re-read on [`ReferenceChanged`](ws-common.md#referencechanged).
- `GetMyQuotes` requires `scope: "live"` or `scope: "history"`. Live is the complete, unpaged Core set of this owner's unfinished quotes across previous connections, including retained and selected quotes. A missing/incomplete source returns an error, not a successful partial set.
- History contains only persisted historical rows. `limit` defaults to 200 and is capped at 1000. Pass `cursor` (last row's `created_at`, Unix seconds) together with `cursor_id` (hex `order_id`) — both or neither. Stop when `has_more` is false. History is not a substitute for Live or execution lookup.
- `GetMarketsForMaker` uses the wider active set (non-finalized, non-disabled, `expiry_ts > now`), so it still lists a market during its final pre-expiry no-trade window.

Optional filters for `GetMakerPositions`:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "underlying_mint": "MintBase58",
    "status": ["open"],
    "min_expiry_ts": 1710000000,
    "limit": 100
  }
}
```

All filter fields are optional. If omitted, all filters match. `limit` is optional,
defaults to `100`, and is clamped to `[1, 500]`. The `MakerPositions` response sets
`has_more: true` when the result was truncated at `limit`; page with the keyset cursor: pass `cursor` (the `created_at` of the last row, unix seconds) together with `cursor_id` (its `pda`) — both or neither. Ordering is second-granular with `pda` as the tie-break; stop when `has_more` is `false`.
Avoid using this request as a high-frequency refresh path.

Optional filters for `GetMarketsForMaker`:

```json
{
  "type": "GetMarketsForMaker",
  "data": {
    "request_id": "uuid",
    "underlying_mints": ["MintBase58"],
    "quote_mints": ["MintBase58"],
    "min_expiry_ts": 1710000000,
    "max_expiry_ts": 1711000000,
    "is_put": false,
    "include_stats": true
  }
}
```

All filter fields are optional. `include_stats` defaults to `false`.
The current wire request has no result cap. Use filters for dashboard views and
avoid polling this request.

### Markets payload

```json
{
  "type": "Markets",
  "data": {
    "request_id": "uuid",
    "markets": [
      {
        "pda": "MarketPdaBase58",
        "underlying": "UnderlyingMintBase58",
        "quote": "QuoteMintBase58",
        "expiry_ts": 1710600000,
        "is_put": false,
        "status": "active"
      }
    ]
  }
}
```

Same compact [`MarketInfo`](ws-common.md#marketinfo) shape as `Snapshot.markets`; active markets only.

### Expiries payload

```json
{
  "type": "Expiries",
  "data": {
    "request_id": "uuid",
    "expiries_ts": [1710600000, 1711200000]
  }
}
```

List of all distinct market expiry timestamps (Unix seconds).

### Tokens payload

```jsonc
{ "type": "Tokens", "data": { "request_id": "uuid", "tokens": [ /* TokenInfo */ ] } }
```

Each mint once; see [`TokenInfo`](ws-common.md#tokeninfo). Underlyings carry `size_rule`,
`strike_tick` and `quote_mints`.

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
    "request_id": "uuid",
    "positions": [
      {
        "pda": "PositionPdaBase58",
        "market": "MarketPdaBase58",
        "underlying_mint": "UnderlyingMintBase58",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "QuoteMintBase58",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "position_type": "covered_call",
        "status": "open",
        "strike": 160000000000,
        "quantity": 1000000000,
        "price": 50000000,
        "total_premium": 123456789,
        "created_at": 1710000042,
        "expiry_ts": 1710600000
      }
    ],
    "has_more": false
  }
}
```

`has_more`: `true` when the result was truncated at `limit` (default 100, max 500). To page with the keyset cursor: pass `cursor` (the `created_at` of the last row, unix seconds) together with `cursor_id` (its `pda`) — both or neither. Ordering is second-granular with `pda` as the tie-break; stop when `has_more` is `false`.

`status` values: `none`, `open`, `funded`, `liquidated`, `settled` (see [PositionStatus](#positionstatus)).

`reconciliation_status` is normally omitted. During lifecycle repair the server
may expose `orphan_pending` or `orphaned_closed` while retaining public
`status = "settled"`.

`is_otm` is `null` for open/funded positions. Set to `true` (out-of-the-money) or `false` (in-the-money) after settlement or liquidation. Omitted from the wire when `null`.

`settlement_price` is the on-chain settlement price (1e9 scale) for `settled`/`liquidated` positions. Omitted from the wire when `null`.

Field semantics:
- `underlying_symbol` / `quote_symbol` / `underlying_decimals` / `quote_decimals`: pre-resolved token metadata so the maker does not need a separate `GetTokens` lookup
- `price`: gross premium per 1 underlying unit (1e9 scale)
- `total_premium`: net premium amount from on-chain position state (quote atomic units)
- all amount fields are unsigned (`u64`); timestamps remain Unix seconds

### MyQuotes payload

```json
{
  "type": "MyQuotes",
  "data": {
    "request_id": "uuid",
    "has_more": false,
    "quotes": [{
      "rfq_id": "uuid",
      "order_id": "0x...64chars",
      "market": "MarketPdaBase58",
      "underlying_mint": "UnderlyingMintBase58",
      "underlying_symbol": "SOL",
      "underlying_decimals": 9,
      "quote_mint": "QuoteMintBase58",
      "quote_symbol": "USDC",
      "quote_decimals": 6,
      "strike": 160000000000,
      "price": 50000000,
      "quantity": 1000000000,
      "valid_until": 1710000350,
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

Historical `status` is one of `submitted`, `filled`, `cancelled`, `replaced`, `expired`, `lost`. There is no top-level `status` or `selected`. `has_more` is required; it is always false for Live.

```json
{ "type": "GetMyQuotes", "data": { "request_id": "uuid", "scope": "history", "limit": 200 } }
```

A History row uses the same identity/amount fields with, for example, `"state": { "type": "historical", "status": "filled" }`. Live and History never mix in one response.

### GetOrderStatus / OrderStatus

```json
{ "type": "GetOrderStatus", "data": { "request_id": "uuid", "order_id": "0x...64chars" } }
{ "type": "OrderStatus", "data": { "request_id": "uuid", "order_id": "0x...64chars", "state": { "type": "pending" } } }
```

Available to the authenticated owner on `/maker`, `/maker/data`, `/taker`, and to the operator on `/vault`. Makers should use `/maker/data` to keep execution lookups off the quote connection. `state` is `pending`, `confirmed` (with `position_pda`), or `unknown`, encoded as a tagged object. See the shared [execution lookup contract](taker-api.md#orderstatus-server---taker).

A lost ACK leaves the command outcome unknown. Keep unresolved orders in strategy state across reconnect; neither an empty Live response, a missing position, a timeout nor `unknown` proves nonexecution. Execution lookup failures return `RequestError` with the request ID.

### MakerMarkets payload

```json
{
  "type": "MakerMarkets",
  "data": {
    "request_id": "uuid",
    "markets": [
      {
        "market_pda": "MarketPdaBase58",
        "underlying_mint": "UnderlyingMintBase58",
        "quote_mint": "QuoteMintBase58",
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

`stats` is present when requested via `include_stats: true` in `GetMarketsForMaker`.

### MmSummary payload

`MmSummary` is the one-shot MM dashboard bootstrap response. It bundles caps, positions, active quotes, markets, and token metadata in a single payload so a freshly-connected market maker can render its UI without firing a fan-out of discovery requests.

Send `GetMmSummary` on `/maker/data`. Valid triggers are initial connect/auth,
reconnect, manual refresh, detected drift, and post-transaction reconciliation
when no owner-direct push is expected. Do not poll it. The current server charges
it through the query bucket as one query message; operators should budget it as a
heavy query, and the next backend policy target is `10` query tokens.

```json
{
  "type": "MmSummary",
  "data": {
    "request_id": "uuid",
    "maker_pda": "MakerPdaBase58",
    "caps": {
      "request_id": "uuid",
      "positions": { "current": 5, "limit": 100 },
      "notional": [
        {
          "underlying_mint": "UnderlyingMintBase58",
          "symbol": "SOL",
          "current": 50000000000,
          "limit": 200000000000
        }
      ],
      "balances": [
        {
          "mint": "QuoteMintBase58",
          "symbol": "USDC",
          "decimals": 6,
          "deposited": 1000000000,
          "committed": 500000000,
          "available": 500000000
        }
      ]
    },
    "positions": [],
    "active_quotes": [],
    "markets": [],
    "tokens": [],
    "computed_at": 1710000042,
    "positions_has_more": false
  }
}
```

Field semantics:
- `maker_pda`: on-chain maker account PDA (base58)
- `caps`: same shape as the [`MyCaps`](#mycaps-payload) response (echoes `request_id` from `GetMmSummary`)
- `positions`: `MakerPositionInfo[]` — see [MakerPositions payload](#makerpositions-payload)
- `active_quotes`: `MakerQuoteInfo[]` — see [MyQuotes payload](#myquotes-payload)
- `markets`: `MakerMarketInfo[]` — see [MakerMarkets payload](#makermarkets-payload)
- `tokens`: `TokenInfo[]` — see [`TokenInfo`](ws-common.md#tokeninfo)
- `computed_at`: server-side snapshot timestamp (Unix seconds)
- `positions_has_more`: `true` when the embedded `positions` were capped at 500. Retrieve the rest via `GetMakerPositions` — page with the keyset cursor: pass `cursor` (the `created_at` of the last row, unix seconds) together with `cursor_id` (its `pda`) — both or neither. Ordering is second-granular with `pda` as the tie-break; stop when `has_more` is `false`. The rest of the snapshot is complete.

All balance values are in their respective token atomic units (use `decimals` from `caps.balances` to format).

### ActiveRfqs payload

```json
{
  "type": "ActiveRfqs",
  "data": {
    "request_id": "uuid",
    "rfqs": [
      {
        "rfq_id": "uuid",
        "market": "MarketPdaBase58",
        "taker": "TakerPubkeyBase58",
        "position_type": "covered_call",
        "strike": 160000000000,
        "quantity": 1000000000,
        "expires_at": 1710000050,
        "quotes_count": 3,
        "best_price": 55000000,
        "order_options": [{ "strike": 150000000000 }, { "strike": 160000000000 }]
      }
    ]
  }
}
```

`best_price` may be `null` if no quotes have been submitted yet.

### GetMyTrades (maker -> server)

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "uuid",
    "limit": 50,
    "cursor": 1710000042,
    "cursor_id": "uuid",
    "market": "MarketPdaBase58"
  }
}
```

All fields except `request_id` are optional.

- `limit`: max number of trades to return (server-defined default, usually 50).
- `cursor` / `cursor_id`: keyset pagination cursor. Use `confirmed_at` and `id` from the last trade in the previous page. The cursor is exclusive (returns trades older than the cursor).
- `market`: filter by market PDA.

### MyTrades payload

```json
{
  "type": "MyTrades",
  "data": {
    "request_id": "uuid",
    "trades": [
      {
        "id": "uuid",
        "rfq_id": "uuid",
        "market_pda": "MarketPdaBase58",
        "underlying_mint": "UnderlyingMintBase58",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "QuoteMintBase58",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "position_type": "covered_call",
        "taker": "TakerPubkeyBase58",
        "strike": 160000000000,
        "quantity": 1000000000,
        "price": 50000000,
        "tx_signature": "5eyk...base58sig",
        "position_pda": "PositionPdaBase58",
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
- `has_more`: `true` if there are more trades beyond this page; use the last trade's `confirmed_at` and `id` as `cursor` and `cursor_id` for the next page

`MyTrades` returns **confirmed trades only** — in-flight orders are visible via `GetOrderStatus` and the order push flow, not here. The cursor is therefore stable.

### TokenCaps payload

Platform-level OI and notional caps. Full schema (fields, unlimited semantics, `include_markets` behavior): [caps.md](caps.md#platform-caps).

### MyCaps payload

Per-maker position count, notional, and balance caps. Full schema (fields, `decimals` rendering, unlimited semantics): [caps.md](caps.md#maker-caps).

### CheckQuote / QuoteCheck

Dry run of the cap checks a `Quote` of `quantity` at `price` on `market_id` (the market PDA) would face. Allowed on `/maker`, `/maker/data` and `/vault`. Nothing is reserved: a passing check is advice, not a hold.

```json
{
  "type": "CheckQuote",
  "data": {
    "request_id": "uuid",
    "market_id": "MarketPdaBase58",
    "quantity": 1000000000,
    "price": 50000000
  }
}
```

```json
{
  "type": "QuoteCheck",
  "data": {
    "request_id": "uuid",
    "admissible": false,
    "cap_detail": { "maker_position_cap_exceeded": { "current": 10, "limit": 10 } }
  }
}
```

`quantity` is in underlying base units; `price` is the gross price per underlying unit scaled by 1e9, as on `Quote`. `cap_detail` is omitted when `admissible` is `true`; otherwise it is the [CapError](caps.md#caperror-variants) a `QuoteRejected` would carry. `"cap_detail": "caps_unavailable"` means retry, not a cap. An unknown `market_id` returns an uncorrelated `Error` with `Generic` code `market_not_found`.

### Subscriptions payload

```json
{
  "type": "Subscriptions",
  "data": {
    "request_id": "uuid",
    "channels": ["rfqs", "chain_events"]
  }
}
```

Subscription responses omit mint lists when unfiltered; filtered lists contain base58 mint addresses.

---

## Broadcast events

Received when subscribed to the corresponding channel.

### TradeExecuted (channel: `trades`)

```json
{
  "type": "TradeExecuted",
  "data": {
    "trade": {
      "id": "uuid",
      "market": "MarketPdaBase58",
      "position_type": "covered_call",
      "strike": 160000000000,
      "quantity": 1000000000,
      "price": 50000000,
      "taker": "TakerPubkeyBase58",
      "maker": "MakerPubkeyBase58",
      "tx_signature": "5eyk...base58sig",
      "executed_at": 1710000042
    }
  }
}
```

### StatsUpdate (channel: `stats`)

```json
{
  "type": "StatsUpdate",
  "data": {
    "stats": {
      "usd": { "notional_24h": "1500.00", "premium_24h": "49.00", "priced_trades_24h": 149 },
      "total_volume_24h": 1000000,
      "total_trades_24h": 150,
      "total_price_24h": 500000,
      "active_markets": 12,
      "active_makers": 5,
      "active_rfqs": 3
    }
  }
}
```

`stats.usd` contains exact decimal USD strings: underlying notional and gross
premium. Only confirmed orders with a captured oracle valuation contribute;
`priced_trades_24h` exposes coverage against `total_trades_24h`. Older servers omit
this object. Legacy numeric volume/price aggregates are not dollars and can retain
the last representable value after overflow; prefer the USD object.

### PositionUpdated (owner-only push)

`PositionUpdated` is delivered **directly to the session of the maker that owns the position**. It is never broadcast and needs no subscription (there is no `positions` channel). After a fill, positions move through `open` -> `funded` -> `settled` or `liquidated`. See [Protocol Flow](protocol-flow.md).

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "pda": "PositionPdaBase58",
      "market": "MarketPdaBase58",
      "underlying_mint": "UnderlyingMintBase58",
      "quote_mint": "QuoteMintBase58",
      "position_type": "covered_call",
      "status": "funded",
      "strike": 160000000000,
      "quantity": 1000000000,
      "price": 50000000,
      "total_premium": 123456789,
      "created_at": 1710000042,
      "expiry_ts": 1710600000
    },
    "update_type": "funded",
    "caps_snapshot": {
      "positions": { "current": 6, "limit": 100 },
      "notional": [
        {
          "underlying_mint": "UnderlyingMintBase58",
          "symbol": "SOL",
          "current": 51000000000,
          "limit": 200000000000
        }
      ],
      "balances": [
        {
          "mint": "QuoteMintBase58",
          "symbol": "USDC",
          "decimals": 6,
          "deposited": 1000000000,
          "committed": 550000000,
          "available": 450000000
        }
      ]
    }
  }
}
```

The type admits `created`, `funded`, `liquidated`, `settled`; the server emits this push for funding (`funded`) only. Do not rely on it for fills or settlement notifications: reconcile via `GetMyCaps` / position queries or the relevant `ChainEvent`.

`caps_snapshot` is the maker's caps **after** this update was applied. It has the same shape as the `caps` portion of [`MmSummary`](#mmsummary-payload), without the inner `request_id`. Use it to keep local caps in sync without calling `GetMyCaps` after every fill.


### MarketCreated (channel: `markets`)

```json
{
  "type": "MarketCreated",
  "data": {
    "pda": "MarketPdaBase58",
    "underlying": "UnderlyingMintBase58",
    "quote": "QuoteMintBase58",
    "expiry_ts": 1710600000,
    "is_put": false
  }
}
```

### MarketFinalized (channel: `markets`)

```json
{
  "type": "MarketFinalized",
  "data": {
    "market_pda": "MarketPdaBase58",
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
    "signature": "5eyk...base58sig",
    "slot": 250000000,
    "market": "MarketPdaBase58",
    "maker": "MakerPubkeyBase58",
    "taker": "TakerPubkeyBase58",
    "position_type": "covered_call",
    "strike": 160000000000,
    "quantity": 1000000000,
    "price": 50000000,
    "order_id": "0x...64chars",
    "instruction_index": 0
  }
}
```

Variants:

| `event_type` | Additional fields |
|--------|-------------------|
| `PositionOpened` | `market`, `maker`, `taker`, `position_type`, `strike`, `quantity`, `price`, `order_id` |
| `MarketCreated` | `market`, `underlying_mint`, `quote_mint`, `expiry_ts`, `is_put` |
| `MarketFinalized` | `market`, `settlement_price` |
| `MakerRegistered` | `owner`, `maker_pda`, `quote_signing` |
| `PositionSettled` | `position` |
| `PositionLiquidated` | `position` |

All variants require `signature` (tx signature, base58), `instruction_index` (instruction coordinate) and `slot` (Solana slot number). Deduplicate by `(signature, instruction_index)`; one transaction may contain multiple events.

---

## Delivery and recovery

When your outbound queue is full, a *critical* message is dropped and the server **closes the connection** (`4002 slow_consumer`), so its loss always surfaces as a disconnect. Best-effort broadcasts are dropped silently. For `/maker` the critical set is `RfqBroadcast`, the quote and order lifecycle, `QuoteRefreshRequested`, `MarketCreated`, `MarketFinalized`, `VaultCatalogChanged`, `VaultProjectionAdvanced`, and every message addressed to your session alone (responses, `PositionUpdated`, `MakerCapsUpdated`). `ChainEvent`, `StatsUpdate`, `RfqSkipped` and `IndicativePricesRequest` broadcasts are best-effort. Owner-direct `TradeExecuted` uses addressed delivery and closes the affected session on queue overflow. The full table is in [WebSocket common](ws-common.md#delivery-criticality).

On every (re)connect read `GetActiveRfqs` and complete `GetMyQuotes { scope: Live }` on `/maker`, and `GetMmSummary` on `/maker/data`. The managed Rust SDK sends exactly these reads per endpoint before `Ready`; querying `GetOrderStatus` for orders you still consider unresolved is up to you. These observations span Core live state and persistent projections; they are not an atomic snapshot. `ActiveRfqs` supplies each RFQ's taker and market PDA for the quote preimage.

The wire has no replay cursor or stream sequence number. Apply entity versions on a live connection and re-read state after disconnect; do not infer execution from absent events. SDK receiver sequence gaps are local delivery gaps, distinct from the wire.

`TradeExecuted` is a UI hint, not authoritative. The DB-backed reads — `GetMmSummary`, `GetMakerPositions`, `GetMyTrades` — are the recovery authority: on any doubt, re-read rather than trusting the last live push. A fill's `PositionUpdated` can occasionally be deferred to a later snapshot rather than pushed, so also reconcile via `GetMmSummary` after your own transactions and when you detect drift, not on a timer.

If `MmSummary.positions_has_more` (or a `MakerPositions` response's `has_more`) is `true`, the position list was truncated at 500 — retrieve the rest via `GetMakerPositions`: page with the keyset cursor: pass `cursor` (the `created_at` of the last row, unix seconds) together with `cursor_id` (its `pda`) — both or neither. Ordering is second-granular with `pda` as the tie-break; stop when `has_more` is `false`.

---

## Indicative pricing workflow

### Request from taker-side cache (server -> maker)

```json
{
  "type": "IndicativePricesRequest",
  "data": {
    "request_id": "uuid",
    "market": {
      "chain_id": 0,
      "program_id": "ProgramIdBase58",
      "market_pda": "MarketPdaBase58",
      "underlying_mint": "UnderlyingMintBase58",
      "quote_mint": "QuoteMintBase58",
      "expiry_ts": 1710600000,
      "is_put": false,
      "collateral_mint": "CollateralMintBase58",
      "settlement_mint": "SettlementMintBase58"
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
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "position_type": "covered_call",
    "prices": [{ "strike": 150000000000, "price": 42000000 }]
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
5. Keep fallback handling for unknown `generic.code`.

Important typed `ServerError` variants for makers (PascalCase on the wire):
- `RfqNotFound`, `RfqNotActive`
- `QuoteNotFound`, `QuoteExpired`, `QuoteLocked`
- `InvalidStrike`, `InvalidValidUntil`, `OrderIdMismatch`, `QuoteExpiryTooShort`
- `SignatureTimeout`
- `OracleNotReady`, `OraclePriceNotReady`, `OraclePriceStale`
- `InvalidPositionType`, `InvalidMarket`
- `MarketMetadataIncomplete`, `TokenMetadataIncomplete`
- `Cap` (position, notional, or balance cap exceeded)
- `RateLimit` (data is a plain string reason code)
- `DbDisabled` (DB unavailable for query)
- `KernelNotAvailable`
- `ServerShuttingDown`
- `Unauthenticated`, `Unauthorized`

Quote-specific validation failures are sent as `QuoteRejected` with a typed `reason`. See the `QuoteRejected` section above for its variants.

`Generic` codes are listed, with their frames, in the [WS error code catalog](ws-common.md#ws-error-codes); match on `code`.

Operational handling recommendations:
- for `rate_limited`: retry with exponential backoff + jitter. Quote/query-bucket rejects are soft (connection stays open); a message-rate-bucket breach closes the connection — see Operational defaults.
- for `session_replaced`: another connection replaced this session — stop; do not auto-reconnect in a loop.
- for `parse_error`: treat as payload bug and stop retries until fixed.
- for `too_many_parse_errors`: reconnect only after fixing payload/codec mismatch.

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
| `maker_disconnected` | Session had `cancel_on_disconnect` enabled and disconnected |

### RfqAvailableAgainReason

| Value | Meaning |
|-------|---------|
| `signature_timeout` | Taker did not sign in time |
| `tx_failed` | On-chain transaction failed |
| `tx_build_failed` | Transaction could not be built |

When `reason = "tx_build_failed"`, a corresponding `Error` or `RequestError` is also sent to
the taker with `generic.code = "tx_build_failed"` and a `message` field
describing what went wrong.

### PositionStatus

Discriminant values for `MakerPositionInfo.status`:

| Value | Meaning |
|-------|---------|
| `none` | Reserved/uninitialized; should not appear on the wire for live positions. |
| `open` | Position created on-chain but maker has not yet escrowed settlement asset. |
| `funded` | Maker called `DepositFundsToPosition`. Settlement asset is escrowed. |
| `liquidated` | Unfunded ITM position seized by a third-party liquidator. |
| `settled` | Position settled at expiry; assets distributed per ITM/OTM outcome. |

Position status is a lowercase snake_case string on the wire.

### PositionUpdateType (in `PositionUpdated`)

| Value | Meaning |
|-------|---------|
| `created` | Position opened on-chain. Taker collateral locked, premium paid to taker. |
| `funded` | Maker called `DepositFundsToPosition`. Settlement asset escrowed. |
| `liquidated` | Unfunded ITM position liquidated by a third party. |
| `settled` | Position settled at expiry. Assets distributed per ITM/OTM outcome. |

State transitions: `created` -> `funded` -> `settled` or `liquidated`.
An unfunded position (`created`) can also be `settled` if OTM, or `liquidated` if ITM.

See [Protocol Flow](protocol-flow.md) for all four scenarios.

### RateLimitReason (in typed `rate_limit` error)

| Value | Meaning |
|-------|---------|
| `too_many_active_rfqs_total` | Platform-wide RFQ limit reached |
| `too_many_active_rfqs_per_taker` | Per-taker active RFQ limit reached |
| `too_many_quotes_per_rfq` | Max quotes per RFQ reached |
| `too_many_failed_orders_per_rfq` | `AcceptQuote` refused: too many failed orders on this RFQ are still unresolved. Create a new RFQ |
| `too_many_sessions_per_user` | Too many concurrent sessions |

### CapError (in typed `cap` error and `RfqSkipped.reason`)

Full variant catalog (`token_oi_cap_exceeded`, `market_oi_cap_exceeded`, `maker_position_cap_exceeded`, `maker_notional_cap_exceeded`, `maker_insufficient_balance`, `quote_notional_cap_exceeded`, `maker_quote_notional_cap_exceeded`) with fields and rejection semantics: [caps.md](caps.md#caperror-variants).

---

## Quirks and constraints

Details that are easy to miss in the schemas.

### Multiple quotes per RFQ

When `RfqBroadcast.order_options` lists multiple strikes, you can quote each strike independently — each quote needs a distinct `order_id` and `nonce`. Submitting a new quote on the same `(rfq_id, strike)` pair replaces your prior quote on that pair; `QuoteAcknowledged.replaced_order_id` tells you which one was displaced. Across all makers, `max_quotes_per_rfq` is 50.

### `order_options` validation

If `order_options` is set, the quote's `strike` must be one of the listed strikes. Anything else returns `invalid_strike`.

### Subscribe / Unsubscribe

Both require a `request_id`. The ack carries the *diff* of channels added or removed by this call, not the full subscription. For the full set, query `GetSubscriptions`.

### Sessions and disconnects

One active quote connection per maker pubkey. A new quote connection replaces the previous quote connection; the separate data connection is for reads. On replacement, the displaced session gets `Error { generic.code = "session_replaced" }` and is closed.

The client requests `cancel_on_disconnect` in `Hello.features`; the server enables it by including it in `Welcome.enabled_features`. The managed quote SDK requests it by default. Selected and executing quotes remain obligations. Reconcile `GetMyQuotes { scope: "live" }` and execution status after reconnect.

A resumed maker auth session restores server-side routing and mint scope. Omitted or null mint fields leave the saved scope unchanged; explicit empty lists clear mint filters. After fresh auth, establish subscriptions again. The managed SDK restores its own desired target after either path, sending both mint lists explicitly, including empty lists.

### Max message size

32 KB inbound. A larger frame closes the connection without an error frame.

### Nonce

`nonce` is a `u64` baked into the order-id preimage. The server doesn't enforce uniqueness on the nonce itself — it's validated only as part of `sha256(preimage)` matching the submitted `order_id`.

### `order_id` hex

Accepted with or without `0x` prefix; case-insensitive. Must decode to exactly 32 bytes.

### Message ordering after auth

After `AuthSuccess`, the server sends `Snapshot` before any broadcast events. You can safely initialise state from `Snapshot` before processing push messages.

### `is_taker_buy`

In the order-id preimage, `is_taker_buy` is always `0`. Reserved for future use.

### `gross_price` in preimage

The `gross_price` at preimage offset 94 is the same value as `price` in the `Quote` message — gross premium per one underlying unit, 1e9 scale. `total_premium` in position data comes from on-chain state and is the net amount, not derivable from the wire `price` alone. The gross/net split is documented under the fee model in [WS common conventions](ws-common.md).

---

## Related

- [Maker quickstart](../quickstart/maker-quickstart.md)
- [Maker wire examples](../quickstart/maker-wire-examples.md)
- [WS common conventions](ws-common.md)
