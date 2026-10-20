# Acta Taker Wire Examples

Payloads illustrate wire shapes. Replace abbreviated IDs, addresses, signatures and past timestamps with real values; request/session/RFQ IDs must be UUIDs and order IDs must decode to 32 bytes. These examples are not transactions to send unchanged.


Concrete JSON for a full taker session and the common branch scenarios. Message shapes and field semantics are in [`../reference/taker-api.md`](../reference/taker-api.md); the narrative walkthrough is in [`taker-quickstart.md`](taker-quickstart.md).

All pubkeys/mints are base58, `order_id` is 64-char hex (optional `0x`), amounts are `u64` (price/strike 1e9-scaled, quantity in underlying atomic units), timestamps are Unix seconds unless the field name ends in `_ms`. Placeholder strings like `MarketPdaBase58` stand in for real base58 values.

## Complete Session (happy path)

### 1) Hello — client → server

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "features": [],
    "client_name": "my-app",
    "client_version": "0.1.0"
  }
}
```

### 2) Welcome — server → client

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

### 3) StartAuth → AuthRequest → AuthChallenge → AuthSuccess

Taker auth is lazy; you may defer it until the first authenticated action. Start the fresh-sign flow with your wallet pubkey:

```json
{ "type": "StartAuth", "data": { "pubkey": "TakerWalletPubkeyBase58" } }
```

The server replies with a challenge. The **taker** challenge includes a `Wallet:` line (the maker challenge does not):

```json
{
  "type": "AuthRequest",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: TakerWalletPubkeyBase58\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2024-03-09T12:00:00Z\n"
  }
}
```

Validate the complete [canonical challenge](../reference/ws-common.md#what-to-sign), including the expected wallet, before signing. Sign its original UTF-8 bytes, base58-encode the 64-byte signature, and echo it back:

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
    "expires_at": 1710086400
  }
}
```

For takers, `expires_at` is always a number. Persist `session_id` + `expires_at` for `ResumeAuth`.

### 4) Snapshot — server → client

Sent automatically after `AuthSuccess`. `markets` lists the active markets in the compact `MarketInfo` shape — fetch full descriptors separately (step 5) before creating an RFQ:

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
        "status": "active"
      }
    ]
  }
}
```

### 5) GetMarketDescriptors → MarketDescriptors

Full descriptors carry `size_rule`, decimals, and oracle PDAs needed to validate `quantity` and render prices. Call this before `RfqRequest`.

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

### 6) Subscribe → SubscribeAck

`request_id` is required for takers. A taker subscribes to `chain_events` (position settled/liquidated), and optionally `trades` / `stats` / `markets`. `rfqs` is a **maker** channel — a taker's own RFQ/quote/order events arrive session-direct, and `PositionUpdated` is never delivered to takers.

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "4e9a2b17-8c6d-4f31-a5e0-7d3b1c9f2a64",
    "channels": ["chain_events", "trades"],
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

### 7) RfqRequest → RfqCreated

`quantity` is underlying atomic units (also for cash-secured puts). Here: 5 SOL, covered call, strike 136.0.

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "market": "MarketPdaBase58",
    "position_type": "covered_call",
    "strike": 136000000000,
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
    "request_id": "7a4c4b3e-2f1d-4c55-9a51-4f0d0e2b6a42",
    "expires_at": 1710000030,
    "created_at": 1710000000,
    "order_options": [{ "strike": 136000000000 }]
  }
}
```

### 8) QuoteReceived (streamed, 0..N) — server → client

Each quote is firm and hash-bound via `order_id`. Two makers respond:

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "strike": 136000000000,
    "maker": "MakerAOwnerPubkeyBase58",
    "price": 50000000,
    "net_price": 49750000,
    "valid_until": 1710000330,
    "nonce": 42,
    "order_id": "0x1111111111111111111111111111111111111111111111111111111111111111"
  }
}
```

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "strike": 136000000000,
    "maker": "MakerBOwnerPubkeyBase58",
    "price": 51000000,
    "net_price": 50745000,
    "valid_until": 1710000335,
    "nonce": 7,
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222"
  }
}
```

You pick the winner by `order_id` (here maker B, the higher premium). Show `net_price`; use `price` + `order_id` for the accept.

### 9) AcceptQuote → CommandAccepted → SponsoredTxToSign

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
    "order_id": "0x2222...2222",
    "tx_base64": "AQABAg... (base64 v0 VersionedTransaction) ...",
    "last_valid_block_height": 312456789,
    "signature_deadline": 1710000030
  }
}
```

The receipt and the lifecycle push (`SponsoredTxToSign` here, `OrderAccepted` in step 10) can arrive in either order. Sign `tx_base64` and return it before `signature_deadline`. On an ordinary maker quote the taker fills **signature slot 1** (slot 0 is the keeper fee-payer); on a vault quote (`maker` is a vault PDA) the taker is the fee payer and fills slot 0; see [Sponsored transaction: raw signing](taker-quickstart.md#sponsored-transaction-raw-signing) for the byte-level layout.

### 10) SubmitSignedSponsoredTx → CommandAccepted → OrderAccepted → OrderSubmitted → OrderConfirmed

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": {
    "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f",
    "order_id": "0x2222...2222",
    "tx_base64": "AQABAg... (same tx, taker slot now filled) ..."
  }
}
```

```json
{ "type": "CommandAccepted", "data": { "request_id": "5f2b8a61-3d4e-4b7c-8e9f-1a2b3c4d5e6f" } }
```

```json
{ "type": "OrderAccepted", "data": { "order_id": "0x2222...2222", "order_version": 1 } }
```

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x2222...2222",
    "tx_signature": "5eyk...base58sig",
    "order_version": 2
  }
}
```

```json
{
  "type": "OrderConfirmed",
  "data": {
    "order_id": "0x2222...2222",
    "position_pda": "PositionPdaBase58",
    "order_version": 4
  }
}
```

`tx_signature` arrives on `OrderSubmitted` (not repeated on `OrderConfirmed`).

### 11) RfqClosed — server → client

Terminal RFQ event; follows `OrderConfirmed` on a fill. Drop per-RFQ state here.

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "filled",
    "winner": {
      "maker": "MakerBOwnerPubkeyBase58",
      "price": 51000000,
      "tx_signature": "5eyk...base58sig"
    },
    "closed_at": 1710000012
  }
}
```

---

## Additional scenarios

### Session resume (skip the wallet signature)

```json
{ "type": "ResumeAuth", "data": { "session_id": "6d1f0c2a-9b7e-4a35-8c41-2e5f7a9b0d13" } }
```

Valid session → `AuthSuccess` (as in step 3, no `AuthRequest`/signing). Invalid/expired → `AuthError`:

```json
{ "type": "AuthError", "data": { "reason": "session_expired" } }
```

### Cancel an RFQ

```json
{ "type": "CancelRfq", "data": { "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e", "request_id": "b3d5f7a9-1c2e-4f40-8b6d-9e0a2c4e6f81" } }
```

```json
{
  "type": "RfqClosed",
  "data": { "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e", "rfq_version": 2, "reason": "taker_cancelled", "closed_at": 1710000020 }
}
```

### Blockhash expiry → reopen and reconcile

Keeper may retry retryable submission failures within five attempts. Exhaustion does not guarantee either message below. An uncertain keeper failure leaves Core `Enqueued`; reconcile with `GetOrderStatus`. A wire failure observation can look like:

```json
{
  "type": "OrderFailed",
  "data": { "order_id": "0x2222...2222", "reason": "blockhash_expired", "order_version": 3 }
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

Refresh the reopened RFQ and select a currently available quote. Rollback discards the winning quote; do not replay its old `order_id`. `RfqAvailableAgain.reason` is an enum: `signature_timeout`, `tx_failed`, `tx_build_failed`. `OrderFailed.reason` is not — it is a free-form diagnostic string. Surface it, do not match on it, and do not re-accept.

Reconcile by `order_version`: accepted is `1`, submitted `2`, failed/expired `3`,
and confirmed `4`. A late on-chain confirmation may therefore replace a local
failure; a later failure can never replace confirmation.

### RFQ expired with no fill

```json
{
  "type": "RfqClosed",
  "data": { "rfq_id": "9a4f8f7b-5a6d-4c8d-8a2e-2a3b4c5d6e7f", "rfq_version": 1, "reason": "expired", "closed_at": 1710000060 }
}
```

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
  "data": { "request_id": "c8e2a4f6-3b5d-4a71-9c0e-5f7a9b1d3e26", "referral_code": "my-code" }
}
```

An `RfqRequest` before redemption fails with a typed `InviteRequired` error in a `RequestError` carrying its `request_id`. Devnet is open — no invite needed.

### Size-rule violation (RequestError)

`quantity` must satisfy `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "1e3a5c7e-9b0d-4f26-8a4c-6e8a0c2e4f93",
    "error": { "type": "Generic", "data": { "code": "invalid_quantity_size_rule", "message": "quantity violates min/max/step" } }
  }
}
```

### Reconcile after reconnect

Subscriptions and in-flight state are not replayed; re-auth, resubscribe, then query:

```json
{ "type": "GetMyActiveRfqs", "data": { "request_id": "d2f4a6c8-7e9b-4b13-8d5f-0a2c4e6a8b37" } }
{ "type": "GetOrderStatus",  "data": { "request_id": "e5a7c9e1-2b4d-4c68-9f1a-3c5e7a9b1d48", "order_id": "0x2222...2222" } }
{ "type": "GetPositions",    "data": { "request_id": "f1b3d5e7-9a2c-4d84-8e6a-4b6c8d0e2f59" } }
```

```json
{
  "type": "Positions",
  "data": {
    "request_id": "f1b3d5e7-9a2c-4d84-8e6a-4b6c8d0e2f59",
    "positions": [
      {
        "pda": "PositionPdaBase58",
        "market": "MarketPdaBase58",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "position_type": "covered_call",
        "status": "open",
        "strike": 136000000000,
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

---

## Related

- [Taker API reference](../reference/taker-api.md) — message catalogue and error variants
- [Taker quickstart](taker-quickstart.md) — narrative walkthrough + raw sponsored-tx signing
- [WS common conventions](../reference/ws-common.md) — units, envelopes, timeouts

## Recovery after reconnect

After `AuthSuccess`, reconcile `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus`. Only a successful resume of the same credential transfers ownership of an unfinished signature. For the same pending order whose signature was not sent, repeat the exact `AcceptQuote` to retrieve the signing payload. For a submitted/enqueued order, query status without replaying trading commands.

`OrderStatus` carries `{ request_id, order_id, state }`, with `state` equal to `{ "type": "pending" }`, `{ "type": "confirmed", "position_pda": "..." }` or `{ "type": "unknown" }`. It has no `order_version`; versions apply to lifecycle pushes. `unknown`, missing positions and timeouts leave the outcome unresolved. See [Delivery & recovery](../reference/taker-api.md#delivery-and-recovery).
