# Acta Maker API Reference

## Endpoint

```
wss://devnet-api.acta.markets/maker
wss://beta-api.acta.markets/maker
```

## Connection flow

```
Connect -> Hello -> Welcome -> AuthRequest -> AuthChallenge -> AuthSuccess -> Snapshot -> Subscribe -> ...
```

Makers are auto-challenged on connect. The server sends `AuthRequest` immediately after `Welcome` without waiting for `StartAuth`. Makers do not use `StartAuth` or `ResumeAuth`.

`AuthSuccess.expires_at` is always `null` for makers (makers authenticate via challenge/signature on every connect and do not support session resume).

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
    "server_time_unix_ms": 1710000000000
  }
}
```

`server_time_unix_ms` is optional and may be omitted. When present, use it for clock skew estimation (see `ws-common.md`).

If the client's `protocol_version` is incompatible, the server sends `VersionMismatch` instead of `Welcome` and closes the connection.

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

`underlying_mints` (optional): filter broadcast events to markets with specific underlying mints. `quote_mints` (optional): filter by quote mint. If both are omitted, subscriptions apply to all markets.

### Connection policy defaults

- Auth deadline (after challenge issued): `15s`
- Idle timeout: `90s`
- Server WS ping interval: `30s`
- Max consecutive parse errors before close: `3`
- Message rate limit per connection: token bucket `30 msg/s` sustained, `60` burst

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
        "is_put": false
      }
    ]
  }
}
```

`Snapshot.markets` uses a compact `MarketInfo` shape (`pda`, `underlying`, `quote`, `expiry_ts`, `is_put`). This is a different DTO from the `MarketDescriptor` in `RfqBroadcast` which has additional fields (`chain_id`, `program_id`, `market_pda`, `underlying_mint`, `quote_mint`, `collateral_mint`, `settlement_mint`). Use `GetMarketDescriptors` for the full market descriptor.

---

## Message index

### Client -> Server

- `Hello`, `AuthChallenge`, `Logout`
- `Subscribe`, `Unsubscribe`, `Ping`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `Quote`, `ReplaceQuote`, `BatchQuotes`, `CancelQuote`, `CancelAllQuotes`
- `IndicativePricesResponse`
- `GetMyQuotes`, `GetMakerPositions`, `GetMarketsForMaker`, `GetMmSummary`, `GetMyTrades`, `GetTokenCaps`, `GetMyCaps`, `GetSubscriptions`
- `GetActiveRfqs`, `GetMarkets`, `GetMarketDescriptors`, `GetExpiries`, `GetTokens`

### Server -> Maker (direct)

- `Welcome`, `VersionMismatch`, `LogoutSuccess`
- `AuthRequest`, `AuthSuccess`, `AuthError`
- `Snapshot`
- `RfqBroadcast`, `RfqSkipped`
- `QuoteRefreshRequested`, `QuoteAcknowledged`, `QuoteBestStatus`, `QuoteOutbid`, `QuoteSelected`
- `QuoteRejected`, `QuoteFilled`, `QuoteCancelled`, `QuoteExpired`, `RfqAvailableAgain`, `RfqClosed`
- `CancelAllQuotesAck`, `BatchQuotesAck`
- `IndicativePricesRequest`
- `SubscriptionUpdated`
- `Error`, `RequestError`, `Pong`
- `SubscribeAck`, `UnsubscribeAck`

### Server -> Maker (responses / query results)

- `MyQuotes`, `MakerPositions`, `MakerMarkets`, `MmSummary`, `MyTrades`, `TokenCaps`, `MyCaps`, `Subscriptions`
- `ActiveRfqs`, `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`

### Server -> Maker (broadcast if subscribed)

- `TradeExecuted`, `StatsUpdate`, `PositionUpdated`, `ChainEvent`
- `MarketCreated`, `MarketFinalized`
- `QuotesUpdate`

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
    "order_options": [{ "strike": 150000000000 }, { "strike": 160000000000 }]
  }
}
```

`strike` is the primary strike from the RFQ request. `order_options` lists all strikes available for quoting (always includes the primary `strike`). When `order_options` is present, the maker must pick a strike from that set. When `order_options` is empty, only the top-level `strike` is valid. Submitting a strike outside this set results in `QuoteRejected { reason: "invalid_strike" }`.

### RfqSkipped (server -> maker)

Sent instead of `RfqBroadcast` when cap limits pre-filter the maker. `reason` is a cap error code, for example `token_oi_cap_exceeded` or `maker_insufficient_balance`.

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
    "valid_until": 1710000310,
    "nonce": 42,
    "order_id": "0x...64chars",
    "signature": "base58sig"
  }
}
```

Rules:
- `valid_until` MUST be >= `now + 310` seconds (server rejects shorter expiries with `quote_expiry_too_short`)
- The server applies a **300-second settlement buffer**. Your quote is considered active for trading until `valid_until - 300` seconds. Set `valid_until` at least 310 seconds from now to allow a minimum 10-second trading window.
- `order_id = sha256(preimage182)`
- maker signs only 32-byte `order_id`
- if `order_options` is present, strike must be from that set

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
    "valid_until": 1710000310,
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

Submit multiple quotes in one message. Each quote is validated like a standalone `Quote`. The server responds with one `BatchQuotesAck` containing `results[]`, one entry per input quote in the same order. Each result is either `QuoteAcknowledged` or an error object with a `QuoteRejected` reason. Partial success is allowed.

```json
{
  "type": "BatchQuotes",
  "data": {
    "quotes": [
      {
        "rfq_id": "uuid-1",
        "strike": 150000000000,
        "price": 45000000,
        "valid_until": 1710000310,
        "nonce": 42,
        "order_id": "0x...64chars-a",
        "signature": "base58sig-a"
      },
      {
        "rfq_id": "uuid-2",
        "strike": 160000000000,
        "price": 50000000,
        "valid_until": 1710000310,
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

Sent when a quote fails server-side validation. Replaces generic `Error` for quote-specific rejections, providing a typed `reason` for programmatic handling.

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "uuid",
    "order_id": "0x...64chars",
    "reason": "cap_exceeded",
    "message": "optional detail"
  }
}
```

`message` is optional and may be omitted.

### QuoteRejectReason

| Value | Meaning |
|-------|---------|
| `invalid_strike` | Strike not in `order_options` set |
| `market_expired` | Target market has expired |
| `quote_expiry_too_short` | `valid_until` below minimum |
| `invalid_signature` | Maker signature verification failed |
| `maker_not_registered` | Maker not found in on-chain registry |
| `order_id_mismatch` | `order_id` does not match `sha256(preimage)` |
| `cap_exceeded` | Platform or maker cap limit exceeded (see `message` for detail) |
| `rfq_not_found` | RFQ does not exist |
| `rfq_not_active` | RFQ is no longer accepting quotes |
| `duplicate_order_id` | `order_id` already submitted |

### QuoteRefreshRequested (server -> maker)

```json
{
  "type": "QuoteRefreshRequested",
  "data": {
    "rfq_id": "uuid",
    "strike": 160000000000,
    "min_valid_until": 1710000035,
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

`QuoteFilled` is the fill-details event for the winning maker.
`RfqClosed` is the terminal RFQ event.

Current runtime behavior for successful fills:
- winner maker receives `QuoteFilled` and then `RfqClosed` (in that order for maker session)
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

`reason` values: `requested` (maker requested), `risk_check` (server-side risk), `rfq_accepted` (RFQ accepted another quote).

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

### QuotesUpdate (server -> maker)

```json
{
  "type": "QuotesUpdate",
  "data": {
    "rfq_id": "uuid",
    "quotes": [
      {
        "rfq_id": "uuid",
        "strike": 160000000000,
        "maker": "MakerOwnerPubkeyBase58",
        "price": 50000000,
        "valid_until": 1710000310,
        "nonce": 42,
        "order_id": "0x...64chars"
      }
    ]
  }
}
```

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

Cancels **all** of this maker's active quotes on the given RFQ (across all strikes). There is no per-order_id cancel — use `CancelQuote` per rfq_id. Server responds with `QuoteCancelled` (containing all removed `order_ids`) on success, or `Error` (e.g. `QuoteNotFound`, `QuoteLocked`).

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

`market` is optional. If provided, cancels only quotes on that market. If omitted, cancels all active quotes.

Server responds with `CancelAllQuotesAck` immediately, followed by individual `QuoteCancelled` messages as the kernel processes each cancellation.

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

`CancelAllQuotesAck` is sent immediately as acknowledgment that the bulk cancellation was dispatched. `cancelled_count` and `cancelled_order_ids` reflect quotes confirmed cancelled at ack time (may be `0` / empty). Individual `QuoteCancelled` messages follow asynchronously with actual order IDs as the kernel processes each cancellation.

Recommendation: after `CancelAllQuotes`, call `GetMyQuotes` after 100–500ms to confirm all quotes are cancelled, in case some `QuoteCancelled` messages were lost in best-effort delivery.

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
    "underlying_mints": ["MintBase58"],
    "quote_mints": null
  }
}
```

`underlying_mints` and `quote_mints` are `null` when unfiltered (all mints).

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

All discovery requests require `request_id` (UUID). The server echoes `request_id` in the corresponding response.

```json
{ "type": "GetMyQuotes", "data": { "request_id": "uuid", "active_only": true } }
{ "type": "GetMakerPositions", "data": { "request_id": "uuid" } }
{ "type": "GetMarketsForMaker", "data": { "request_id": "uuid" } }
{ "type": "GetMmSummary", "data": { "request_id": "uuid" } }
{ "type": "GetSubscriptions", "data": { "request_id": "uuid" } }
{ "type": "GetActiveRfqs", "data": { "request_id": "uuid" } }
{ "type": "GetMyTrades", "data": { "request_id": "uuid" } }
{ "type": "GetTokenCaps", "data": { "request_id": "uuid" } }
{ "type": "GetMyCaps", "data": { "request_id": "uuid" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "uuid", "active_only": true } }
{ "type": "GetExpiries", "data": { "request_id": "uuid" } }
{ "type": "GetTokens", "data": { "request_id": "uuid", "active_only": true } }
```

`active_only` behavior:
- default is `true` when omitted (wire default for `GetMyQuotes`, `GetMarketDescriptors`, `GetTokens`)
- `GetMyQuotes` with `active_only=true` returns live kernel quotes
- `GetMyQuotes` with `active_only=false` also appends historical quotes from DB; `limit` is optional, defaults to 200, and is capped at 1000
- for `GetMarketDescriptors`/`GetTokens`, `active_only=true` means non-finalized markets with `expiry_ts > now`; `active_only=false` returns all markets/tokens

Optional filters for `GetMakerPositions`:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "underlying_mint": "MintBase58",
    "status": ["open"],
    "min_expiry_ts": 1710000000
  }
}
```

All filter fields are optional. If omitted, all positions are returned.

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
        "is_put": false
      }
    ]
  }
}
```

Same compact `MarketInfo` shape as `Snapshot.markets`.

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

### MarketDescriptorInfo (from `MarketDescriptors`)

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
    ]
  }
}
```

`status` values: `none`, `open`, `funded`, `liquidated`, `settled` (see [PositionStatus](#positionstatus)).

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
    "quotes": [
      {
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
        "valid_until": 1710000310,
        "status": "best",
        "created_at": 1710000010
      }
    ]
  }
}
```

`status` values: `pending`, `best`, `outbid`, `filled`, `expired`.

`selected` is present only for historical quotes (`active_only=false`) — `true` if the taker selected this quote, `false` otherwise. Omitted from the wire for live quotes.

Pre-resolved token metadata (`underlying_*`, `quote_*`) is included so the maker can render quotes without a separate `GetTokens` lookup.

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
        "quote_symbol": "USDC",
        "stats": null
      }
    ]
  }
}
```

`stats` is present when requested via `include_stats: true` in `GetMarketsForMaker`.

### MmSummary payload

`MmSummary` is the one-shot MM dashboard bootstrap response. It bundles caps, positions, active quotes, markets, and token metadata in a single payload so a freshly-connected market maker can render its UI without firing a fan-out of discovery requests.

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
    "computed_at": 1710000042
  }
}
```

Field semantics:
- `maker_pda`: on-chain maker account PDA (base58)
- `caps`: same shape as the [`MyCaps`](#mycaps-payload) response (echoes `request_id` from `GetMmSummary`)
- `positions`: `MakerPositionInfo[]` — see [MakerPositions payload](#makerpositions-payload)
- `active_quotes`: `MakerQuoteInfo[]` — see [MyQuotes payload](#myquotes-payload)
- `markets`: `MakerMarketInfo[]` — see [MakerMarkets payload](#makermarkets-payload)
- `tokens`: `TokenInfo[]` — see [Tokens payload](#tokens-payload)
- `computed_at`: server-side snapshot timestamp (Unix seconds)

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
- `tx_signature` and `position_pda` may be `null` if the transaction has not confirmed yet
- `has_more`: `true` if there are more trades beyond this page; use the last trade's `confirmed_at` and `id` as `cursor` and `cursor_id` for the next page

> **Cursor stability:** Trades with `tx_signature: null` (unconfirmed) may move or disappear from subsequent pages as they confirm on-chain. When paginating, be aware that results may shift between requests. For consistent snapshots, filter to confirmed trades or re-fetch from the beginning after gaps.

### TokenCaps payload

Platform-level OI and notional caps. Full schema (fields, unlimited semantics, `include_markets` behavior): [caps.md](caps.md#platform-caps).

### MyCaps payload

Per-maker position count, notional, and balance caps. Full schema (fields, `decimals` rendering, unlimited semantics): [caps.md](caps.md#maker-caps).

### Subscriptions payload

```json
{
  "type": "Subscriptions",
  "data": {
    "request_id": "uuid",
    "channels": ["rfqs", "chain_events"],
    "markets": null
  }
}
```

`markets` is `null` when subscribed to all markets, or a list of market PDAs when filtered.

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

### PositionUpdated (owner-only push)

`PositionUpdated` is delivered **directly to the session of the maker that owns the position**. It is not broadcast to public channel subscribers, even though `positions` appears in the channel list. After a fill, positions move through `open` -> `funded` -> `settled` or `liquidated`. See [Protocol Flow](protocol-flow.md).

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
      "status": "open",
      "strike": 160000000000,
      "quantity": 1000000000,
      "price": 50000000,
      "total_premium": 123456789,
      "created_at": 1710000042,
      "expiry_ts": 1710600000,
      "is_otm": null
    },
    "update_type": "created",
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

`update_type` values: `created`, `funded`, `liquidated`, `settled`.

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
    "order_id": "0x...64chars"
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

All variants include `signature` (tx signature, base58) and `slot` (Solana slot number).

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
See `ws-common.md` "Error format" for the envelope.

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

Note: Some quote-specific validation errors (e.g. `InvalidStrike`, `OrderIdMismatch`, `CapExceeded`) are now returned as `QuoteRejected` messages rather than `Error`. See the `QuoteRejected` section above for the typed reason enum.

Common `Generic` variant `code` values in maker/runtime flows:
- `parse_error`, `too_many_parse_errors`, `message_too_large`
- `hello_required`, `hello_timeout`, `hello_already_sent`
- `session_replaced`, `session_expired`, `already_authenticated`
- `internal_error`

Operational handling recommendations:
- for `rate_limited`: retry with exponential backoff + jitter
- for `parse_error`: treat as payload bug and stop retries until fixed
- for `too_many_parse_errors`: reconnect only after fixing payload/codec mismatch

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

### RfqAvailableAgainReason

| Value | Meaning |
|-------|---------|
| `signature_timeout` | Taker did not sign in time |
| `tx_failed` | On-chain transaction failed |
| `tx_build_failed` | Transaction could not be built |

When `reason = "tx_build_failed"`, a corresponding `Error` or `RequestError` is also sent to
the taker with `generic.code = "tx_build_failed"` and a `message` field
describing what went wrong.

### QuoteStatus (in `MyQuotes` response)

| Value | Meaning |
|-------|---------|
| `pending` | Quote submitted, awaiting acknowledgment |
| `best` | Currently the best quote |
| `outbid` | Outbid by another maker |
| `filled` | Quote was filled |
| `expired` | Quote expired |

### PositionStatus

Discriminant values for `MakerPositionInfo.status`:

| Value | Meaning |
|-------|---------|
| `none` | Reserved/uninitialized; should not appear on the wire for live positions. |
| `open` | Position created on-chain but maker has not yet escrowed settlement asset. |
| `funded` | Maker called `DepositFundsToPosition`. Settlement asset is escrowed. |
| `liquidated` | Unfunded ITM position seized by a third-party liquidator. |
| `settled` | Position settled at expiry; assets distributed per ITM/OTM outcome. |

Wire form is lowercase snake_case. The legacy numeric encoding (4|5|6 -> `settled`) has been removed; clients must accept the string form.

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

One active WebSocket per maker pubkey. A second connection atomically replaces the first; the displaced session gets `Error { generic.code = "session_replaced" }` and is closed.

Quotes survive disconnects — they live until `valid_until` or `RfqClosed`. Subscriptions don't survive — re-`Subscribe` after every auth. Use `GetMyQuotes` after reconnect to reconcile live state.

### Max message size

32 KB inbound. Larger messages are rejected with `generic.code = "message_too_large"`.

### Nonce

`nonce` is a `u64` baked into the order-id preimage. The server doesn't enforce uniqueness on the nonce itself — it's validated only as part of `sha256(preimage)` matching the submitted `order_id`.

### `order_id` hex

Accepted with or without `0x` prefix; case-insensitive. Must decode to exactly 32 bytes.

### Message ordering after auth

After `AuthSuccess`, the server sends `Snapshot` before any broadcast events. You can safely initialise state from `Snapshot` before processing push messages.

### `is_taker_buy`

In the order-id preimage, `is_taker_buy` is always `0`. Reserved for future use.

### `gross_price` in preimage

The `gross_price` at preimage offset 94 is the same value as `price` in the `Quote` message — gross premium per one underlying unit, 1e9 scale. `total_premium` in position data comes from on-chain state and is the net amount, not derivable from the wire `price` alone. The gross/net split is documented under the fee model in `ws-common.md`.

---

## Related

- Maker quickstart: `../quickstart/maker-quickstart.md`
- Maker wire examples: `../quickstart/maker-wire-examples.md`
- Shared WS conventions: `ws-common.md`
