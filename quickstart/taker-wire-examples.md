# Acta Taker Wire Examples

Payloads illustrate wire shapes. Replace abbreviated IDs, addresses, signatures and past timestamps with real values. Request, session and RFQ IDs must be UUIDs, and order IDs must decode to 32 bytes. These examples are not transactions to send unchanged. The 5 SOL size may exceed a deployment's `max_size`; read `size_rule` first.

Concrete JSON for a full taker session and the common branch scenarios. Message shapes and field semantics are in [`../reference/taker-api.md`](../reference/taker-api.md). The narrative walkthrough is in [`taker-quickstart.md`](taker-quickstart.md).

Pubkeys and mints are base58. `order_id` is 64 hex characters, with an optional `0x`. Amounts are `u64`: prices and strikes are `1e9`-scaled, `quantity` is in underlying atoms, premiums and collateral are quote atoms. Timestamps are Unix seconds unless the field name ends in `_ms`. Placeholders such as `MarketPdaBase58` stand in for real base58 values.

Every frame carries all its fields. Absent facts are `null`, and state-dependent shapes are tagged unions.

## Complete Session (happy path, covered call)

### 1) Hello (client -> server)

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

Any `protocol_version` other than `1.0.0` gets an `Error` with `unsupported_protocol_version`, then close code `4006`.

### 2) Welcome (server -> client)

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

### 3) StartAuth, AuthRequest, AuthChallenge, AuthSuccess

Taker auth is lazy; you may defer it until the first authenticated action. Start the fresh-sign flow with your wallet pubkey:

```json
{ "type": "StartAuth", "data": { "pubkey": "TakerWalletPubkeyBase58" } }
```

The server replies with a challenge. The taker challenge includes a `Wallet:` line (the maker challenge does not):

```json
{
  "type": "AuthRequest",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: TakerWalletPubkeyBase58\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2024-03-09T12:00:00Z\n"
  }
}
```

Validate the complete [canonical challenge](../reference/ws-common.md#what-to-sign), including the expected wallet, before signing. Sign its original UTF-8 bytes, base58-encode the 64-byte signature and echo it back:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: TakerWalletPubkeyBase58\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2024-03-09T12:00:00Z\n",
    "signature": "3q7uQqYc3...base58sig",
    "pubkey": "TakerWalletPubkeyBase58"
  }
}
```

The signature is verified directly against `pubkey`. On success:

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

For takers `expires_at` is a number and `maker_pda` is `null`. Persist `session_id` and `expires_at` for `ResumeAuth`.

### 4) Snapshot (server -> client)

Sent automatically after `AuthSuccess`. `markets` lists the active markets in the `MarketInfo` shape. `products` lists the pair's products that fit the market's side. A put market lists a subset of `cash_secured_put`, `put_spread` and `cash_put`; a call market lists a subset of `covered_call` and `call_spread`. `settlement_price` is `null` until the market settles. Fetch full descriptors (step 5) before creating an RFQ.

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

### 5) GetMarketDescriptors, MarketDescriptors

Full descriptors carry `size_rule`, decimals and oracle PDAs, which you need to validate `quantity` and render prices. Call this before `RfqRequest`.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "2c7e4a91-5d3b-4f60-9a8e-1b4c6d2f7e05" } }
```

```json
{
  "type": "MarketDescriptors",
  "data": {
    "request_id": "2c7e4a91-5d3b-4f60-9a8e-1b4c6d2f7e05",
    "markets": [
      {
        "products": ["covered_call", "call_spread"],
        "market": {
          "chain_id": 0,
          "program_id": "33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U",
          "market_pda": "MarketPdaBase58",
          "underlying_mint": "So11111111111111111111111111111111111111112",
          "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
          "expiry_ts": 1710600000,
          "is_put": false,
          "collateral_mint": "So11111111111111111111111111111111111111112",
          "settlement_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
        },
        "underlying_oracle_pda": "UnderlyingOraclePdaBase58",
        "quote_oracle_pda": "QuoteOraclePdaBase58",
        "underlying_decimals": 9,
        "quote_decimals": 6,
        "size_rule": { "min_size": 1000000000, "max_size": 100000000000, "step": 1000000000 },
        "underlying_symbol": "SOL",
        "quote_symbol": "USDC"
      }
    ]
  }
}
```

`program_id` is the same on devnet and mainnet. `GET /api/v1/venue` returns the authoritative `program_id`.

### 6) Subscribe, SubscribeAck

Every field is required. `underlying_mints: []` and `quote_mints: []` mean every mint. `products` is a non-empty list of unique product names; the server does not use it to filter what a taker receives. A taker tracks position outcomes through `PositionUpdated`, which arrives on the session without a subscription. Subscribe to `chain_events` for chain-level settlement events, and optionally to `trades`, `stats` and `markets`. `rfqs` is a maker channel: a taker's own RFQ, quote and order events arrive on the session.

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

```json
{
  "type": "SubscribeAck",
  "data": { "request_id": "4e9a2b17-8c6d-4f31-a5e0-7d3b1c9f2a64", "subscribed": ["chain_events", "trades"] }
}
```

### 7) RfqRequest, CommandAccepted, RfqCreated

`terms` names the product and its strikes. `side` is `sell`; `buy` is rejected with `taker_buy_unsupported`. `quantity` is underlying atoms for every product. Here: 5 SOL, covered call, strike 136.0.

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "market": "MarketPdaBase58",
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "quantity": 5000000000,
    "timeout_seconds": 30
  }
}
```

```json
{ "type": "CommandAccepted", "data": { "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42" } }
```

```json
{
  "type": "RfqCreated",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 1,
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "expires_at": 1710000030,
    "created_at": 1710000000,
    "order_options": [
      { "type": "covered_call", "strike": 136000000000 }
    ]
  }
}
```

The receipt and `RfqCreated` can arrive in either order. `order_options` is the list of `terms` makers may quote. For a covered call or cash-secured put it is a strike ladder of the same product; here it holds one strike.

### 8) QuoteReceived (streamed, 0..N)

Each quote is firm and hash-bound through `order_id`. `net_price` is a display estimate and is `null` for spreads and cash puts. Two makers respond:

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "maker": "MakerAOwnerPubkeyBase58",
    "price": 50000000,
    "valid_until": 1710000330,
    "nonce": 42,
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111",
    "net_price": 49750000
  }
}
```

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "maker": "MakerBOwnerPubkeyBase58",
    "price": 51000000,
    "valid_until": 1710000335,
    "nonce": 7,
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "net_price": 50745000
  }
}
```

You pick the winner by `order_id` (here maker B, the higher premium). Show `net_price`; use `price` and `order_id` for the accept.

### 9) AcceptQuote, CommandAccepted, SponsoredTxToSign

```json
{
  "type": "AcceptQuote",
  "data": {
    "request_id": "0c9e1d52-6c0b-4f7e-b1a9-7d8f3c2e5a10",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "maker": "MakerBOwnerPubkeyBase58",
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222"
  }
}
```

```json
{ "type": "CommandAccepted", "data": { "request_id": "0c9e1d52-6c0b-4f7e-b1a9-7d8f3c2e5a10" } }
```

```json
{
  "type": "SponsoredTxToSign",
  "data": {
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "tx_base64": "AQABAg... (base64 v0 VersionedTransaction) ...",
    "last_valid_block_height": 312456789,
    "signature_deadline": 1710000030
  }
}
```

The receipt and the lifecycle push (`SponsoredTxToSign` here, `OrderAccepted` in step 10) can arrive in either order. Verify the transaction against the accepted quote, sign it and return it before `signature_deadline`. The keeper is the fee payer in signature slot 0. The taker fills signature slot 1. See [Sponsored transaction: raw signing](taker-quickstart.md#sponsored-transaction-raw-signing) for the byte-level layout.

### 10) SubmitSignedSponsoredTx, CommandAccepted, OrderAccepted, OrderSubmitted, OrderConfirmed

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": {
    "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f",
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "tx_base64": "AQABAg... (same tx, taker slot now filled) ..."
  }
}
```

```json
{ "type": "CommandAccepted", "data": { "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f" } }
```

```json
{ "type": "OrderAccepted", "data": { "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222", "order_version": 1 } }
```

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "tx_signature": "5eyk...base58sig",
    "order_version": 2
  }
}
```

```json
{
  "type": "OrderConfirmed",
  "data": {
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "position_pda": "PositionPdaBase58",
    "order_version": 4
  }
}
```

`tx_signature` arrives on `OrderSubmitted` and is not repeated on `OrderConfirmed`.

### 11) RfqClosed (server -> client)

Terminal RFQ event; it follows `OrderConfirmed` on a fill. Drop per-RFQ state here. `your_quote` and `winner` are always present and `null` when they do not apply.

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "filled",
    "your_quote": null,
    "winner": {
      "maker": "MakerBOwnerPubkeyBase58",
      "price": 51000000,
      "tx_signature": "5eyk...base58sig"
    },
    "closed_at": 1710000012
  }
}
```

### 12) PositionUpdated (server -> client)

After the position is projected, the session receives the full row. No subscription is needed. Apply a row only if its `revision` is newer than the stored one for that `pda`. `total_premium` is the net premium in quote atoms after the protocol fee: `5 * 0.051 USDC = 255000`, minus a 0.5% fee, gives `253725`.

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": { "slot": 312456800, "transaction_index": 14, "instruction_index": 2 },
      "pda": "PositionPdaBase58",
      "market": "MarketPdaBase58",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_decimals": 6,
      "terms": { "type": "covered_call", "strike": 136000000000 },
      "side": "sell",
      "state": "open",
      "quantity": 5000000000,
      "price": 51000000,
      "total_premium": 253725,
      "created_at": 1710000012,
      "expiry_ts": 1710600000
    },
    "update_type": "created"
  }
}
```

---

## Spread session

A put spread on a put market. The market lists `put_spread` in `products`. The strikes are multiples of the underlying's `strike_tick`. The taker sells the 130/140 put spread on 5 SOL. Width `W = 10`, collateral `C = 50 USDC`.

### RfqRequest

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "a1b2c3d4-0001-4a5b-8c6d-7e8f90a1b2c3",
    "market": "PutMarketPdaBase58",
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "quantity": 5000000000,
    "timeout_seconds": 30
  }
}
```

### RfqCreated

`order_options` holds exactly the requested pair.

```json
{
  "type": "RfqCreated",
  "data": {
    "rfq_id": "b2c3d4e5-0002-4b6c-9d7e-8f90a1b2c3d4",
    "rfq_version": 1,
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "request_id": "a1b2c3d4-0001-4a5b-8c6d-7e8f90a1b2c3",
    "expires_at": 1710000030,
    "created_at": 1710000000,
    "order_options": [
      { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 }
    ]
  }
}
```

### QuoteReceived

The gross price `G = 1.2` is below the width `10`. `net_price` is `null`.

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "b2c3d4e5-0002-4b6c-9d7e-8f90a1b2c3d4",
    "terms": { "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 },
    "side": "sell",
    "maker": "MakerAOwnerPubkeyBase58",
    "price": 1200000000,
    "valid_until": 1710000330,
    "nonce": 11,
    "order_id": "0x3333333333333333333333333333333333333333333333333333333333333333",
    "net_price": null
  }
}
```

Gross premium is `6000000` quote atoms. With a premium fee of 50 bps and a volume fee of 10 bps on the `50000000` exposure, the fee is `min(30000, 50000) = 30000` and the net premium is `5970000`. Read the fee bps from `GET /api/v1/venue`. Accept, sign and submit as in steps 9 and 10.

### Settled position

At settlement price `133` the intrinsic value is `7`. The maker receives `35000000` and the taker the remaining `15000000` of the `50000000` collateral.

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": { "slot": 312999000, "transaction_index": 3, "instruction_index": 1 },
      "pda": "PositionPdaBase58",
      "market": "PutMarketPdaBase58",
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

### Cash put

A cash put carries one strike `K`. It executes as a put spread with legs `(0, K)`, so its width is `K`. Only a put market whose `products` include `cash_put` accepts it.

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "c3d4e5f6-0003-4c7d-8e8f-90a1b2c3d4e5",
    "market": "PutMarketPdaBase58",
    "terms": { "type": "cash_put", "strike": 136000000000 },
    "side": "sell",
    "quantity": 5000000000,
    "timeout_seconds": 30
  }
}
```

### Spread rejections

A spread whose strikes are unordered, or whose lower strike is `0` for a call spread:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "a1b2c3d4-0001-4a5b-8c6d-7e8f90a1b2c3",
    "error": { "type": "Generic", "data": { "code": "spread_invalid_strikes", "message": "spreads require ordered strikes; call lower must be positive" } },
    "delivery": null
  }
}
```

Strikes that are not multiples of the underlying's `strike_tick` fail with `spread_tick_violation`. A product that is not enabled for the pair fails with `product_not_allowed`. `"side": "buy"` fails with `taker_buy_unsupported`.

---

## Additional scenarios

### Session resume (skip the wallet signature)

```json
{ "type": "ResumeAuth", "data": { "session_id": "6d1f0c2a-9b7e-4a35-8c41-2e5f7a9b0d13" } }
```

A valid session returns `AuthSuccess` (as in step 3, without `AuthRequest` or signing). An invalid or expired session returns `AuthError`:

```json
{ "type": "AuthError", "data": { "reason": "session_expired", "message": "session expired" } }
```

### Cancel an RFQ

```json
{ "type": "CancelRfq", "data": { "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e", "request_id": "b3d5f7a9-1c2e-4f40-8b6d-9e0a2c4e6f81" } }
```

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "taker_cancelled",
    "your_quote": null,
    "winner": null,
    "closed_at": 1710000020
  }
}
```

### Blockhash expiry, reopen and reconcile

The keeper may retry retryable submission failures within five attempts. Exhaustion does not guarantee either message below. An uncertain keeper failure leaves the order queued; reconcile with `GetOrderStatus`. A failure observation can look like:

```json
{
  "type": "OrderFailed",
  "data": {
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222",
    "reason": "blockhash_expired_before_send",
    "order_version": 3
  }
}
```

Only a locally proven-unforwarded failure may reopen an eligible RFQ with `reason: "tx_failed"`:

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

Refresh the reopened RFQ and select a currently available quote. Rollback discards the winning quote; do not replay its old `order_id`. `RfqAvailableAgain.reason` is an enum: `signature_timeout`, `tx_failed`, `tx_build_failed`. `OrderFailed.reason` is a free-form diagnostic string. Show it, do not match on it, and do not re-accept.

Reconcile by `order_version`: accepted is `1`, submitted `2`, failed or expired `3`, confirmed `4`. A late on-chain confirmation can replace a local failure; a later failure never replaces confirmation.

### Submission failure with delivery

A failed `SubmitSignedSponsoredTx` says whether this request could have been handed off. `not_forwarded` means this request was not sent. `unknown` means it may have been.

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f",
    "error": { "type": "Generic", "data": { "code": "rate_limited", "message": "rate limit exceeded" } },
    "delivery": "not_forwarded"
  }
}
```

Treat the order as not executed only if every submission you made returned `not_forwarded`. Otherwise call `GetOrderStatus` first.

### RFQ expired with no fill

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "9a4f8f7b-5a6d-4c8d-8a2e-2a3b4c5d6e7f",
    "rfq_version": 1,
    "reason": "expired",
    "your_quote": null,
    "winner": null,
    "closed_at": 1710000060
  }
}
```

`reason` is `expired`, `taker_cancelled`, `filled`, `market_expired` or `ladder_timeout`.

### Invite gating (closed mainnet only)

After `AuthSuccess`, an unregistered taker receives (unit variant, no `data`):

```json
{ "type": "RequireInvite" }
```

Redeem before trading:

```json
{ "type": "RedeemInvite", "data": { "request_id": "c8e2a4f6-3b5d-4a71-9c0e-5f7a9b1d3e26", "code": "abc123" } }
```

```json
{
  "type": "InviteRedeemed",
  "data": { "request_id": "c8e2a4f6-3b5d-4a71-9c0e-5f7a9b1d3e26", "referral_code": "mycode" }
}
```

An `RfqRequest` before redemption fails with a typed `InviteRequired` error in a `RequestError` that carries its `request_id`. Devnet is open and needs no invite.

### Size-rule violation (RequestError)

`quantity` must satisfy `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`:

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

### Reconcile after reconnect

Subscriptions and in-flight state are not replayed. Re-authenticate, resubscribe, then query:

```json
{ "type": "GetMyActiveRfqs", "data": { "request_id": "d2f4a6c8-7e9b-4b13-8d5f-0a2c4e6a8b37" } }
{ "type": "GetOrderStatus", "data": { "request_id": "e5a7c9e1-2b4d-4c68-9f1a-3c5e7a9b1d48", "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222" } }
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
        "terms": { "type": "covered_call", "strike": 136000000000 },
        "side": "sell",
        "state": "open",
        "quantity": 5000000000,
        "price": 51000000,
        "total_premium": 253725,
        "created_at": 1710000012,
        "expiry_ts": 1710600000
      }
    ]
  }
}
```

Merge each row into your position store by `pda`, and keep it only if its `revision` is newer than the stored row.

---

## Related

- [Taker API reference](../reference/taker-api.md): message catalogue and error variants
- [Taker quickstart](taker-quickstart.md): narrative walkthrough and raw sponsored-tx signing
- [WS common conventions](../reference/ws-common.md): units, envelopes, timeouts

## Recovery after reconnect

After `AuthSuccess`, reconcile `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus`. Only a successful resume of the same credential transfers ownership of an unfinished signature. For the same pending order whose signature was not sent, repeat the exact `AcceptQuote` to retrieve the signing payload. For a submitted or enqueued order, query status without replaying trading commands.

`OrderStatus` carries `{ request_id, order_id, state }`, with `state` equal to `{ "type": "pending" }`, `{ "type": "confirmed", "position_pda": "..." }` or `{ "type": "unknown" }`. It has no `order_version`; versions apply to lifecycle pushes. `unknown`, missing positions and timeouts leave the outcome unresolved. See [Delivery and recovery](../reference/taker-api.md#delivery-and-recovery).
