# Acta Taker Wire Examples

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
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: TakerWalletPubkeyBase58\nNonce: a1b2c3d4e5f6...hex64\nIssued At: 2024-03-09T12:00:00Z"
  }
}
```

Sign the raw UTF-8 bytes of `challenge` with your wallet key, base58-encode the 64-byte signature, and echo it back:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: TakerWalletPubkeyBase58\nNonce: a1b2c3d4e5f6...hex64\nIssued At: 2024-03-09T12:00:00Z",
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
    "session_id": "sess-abc-123",
    "expires_at": 1710086400
  }
}
```

For takers, `expires_at` is always a number. Persist `session_id` + `expires_at` for `ResumeAuth`.

### 4) Snapshot — server → client

Sent automatically after `AuthSuccess`. `markets` is the compact `MarketInfo` shape — fetch full descriptors separately (step 5) before creating an RFQ:

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
        "is_put": false
      }
    ]
  }
}
```

### 5) GetMarketDescriptors → MarketDescriptors

Full descriptors carry `size_rule`, decimals, and oracle PDAs needed to validate `quantity` and render prices. Call this before `RfqRequest`.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "req-md-1", "active_only": true } }
```

```json
{
  "type": "MarketDescriptors",
  "data": {
    "request_id": "req-md-1",
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

`request_id` is required for takers. A taker subscribes to `chain_events` (position settled/liquidated), and optionally `trades` / `stats` / `markets`. `rfqs` and `positions` are **maker** channels — a taker's own RFQ/quote/order events arrive session-direct, and `PositionUpdated` is never delivered to takers.

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "req-sub-1",
    "channels": ["chain_events", "trades"],
    "underlying_mints": null,
    "quote_mints": null
  }
}
```

```json
{
  "type": "SubscribeAck",
  "data": { "request_id": "req-sub-1", "subscribed": ["chain_events", "trades"] }
}
```

### 7) RfqRequest → RfqCreated

`quantity` is underlying atomic units (also for cash-secured puts). Here: 5 SOL, covered call, strike 136.0.

```json
{
  "type": "RfqRequest",
  "data": {
    "market": "MarketPdaBase58",
    "position_type": "covered_call",
    "strike": 136000000000,
    "quantity": 5000000000,
    "timeout_seconds": 30,
    "client_request_id": "cli-req-42"
  }
}
```

```json
{
  "type": "RfqCreated",
  "data": {
    "rfq_id": "rfq-777",
    "rfq_version": 1,
    "client_request_id": "cli-req-42",
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
    "rfq_id": "rfq-777",
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
    "rfq_id": "rfq-777",
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

### 9) AcceptQuote → OrderAccepted → SponsoredTxToSign

```json
{
  "type": "AcceptQuote",
  "data": {
    "rfq_id": "rfq-777",
    "maker": "MakerBOwnerPubkeyBase58",
    "order_id": "0x2222222222222222222222222222222222222222222222222222222222222222"
  }
}
```

```json
{ "type": "OrderAccepted", "data": { "order_id": "0x2222...2222" } }
```

```json
{
  "type": "SponsoredTxToSign",
  "data": {
    "order_id": "0x2222...2222",
    "tx_base64": "AQABAg... (base64 v0 VersionedTransaction) ...",
    "signature_deadline": 1710000030
  }
}
```

Sign `tx_base64` and return it before `signature_deadline`. The taker fills **signature slot 1** (slot 0 is the keeper fee-payer); see [Sponsored transaction: raw signing](taker-quickstart.md#sponsored-transaction-raw-signing) for the byte-level layout.

### 10) SubmitSignedSponsoredTx → OrderSubmitted → OrderConfirmed

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": {
    "order_id": "0x2222...2222",
    "tx_base64": "AQABAg... (same tx, taker slot now filled) ..."
  }
}
```

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x2222...2222",
    "tx_signature": "5eyk...base58sig",
    "order_version": 1
  }
}
```

```json
{
  "type": "OrderConfirmed",
  "data": {
    "order_id": "0x2222...2222",
    "position_pda": "PositionPdaBase58",
    "order_version": 2
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
    "rfq_id": "rfq-777",
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
{ "type": "ResumeAuth", "data": { "session_id": "sess-abc-123" } }
```

Valid session → `AuthSuccess` (as in step 3, no `AuthRequest`/signing). Invalid/expired → `AuthError`:

```json
{ "type": "AuthError", "data": { "reason": "session_expired" } }
```

### Cancel an RFQ

```json
{ "type": "CancelRfq", "data": { "rfq_id": "rfq-777", "request_id": "req-cancel-1" } }
```

```json
{
  "type": "RfqClosed",
  "data": { "rfq_id": "rfq-777", "rfq_version": 2, "reason": "taker_cancelled", "closed_at": 1710000020 }
}
```

### Blockhash expiry → auto-reopen → re-accept

Under congestion the sponsored tx can miss its blockhash. The server retries internally up to 5×; if all fail it emits `OrderFailed` then `RfqAvailableAgain`:

```json
{
  "type": "OrderFailed",
  "data": { "order_id": "0x2222...2222", "reason": "blockhash_expired", "order_version": 2 }
}
```

```json
{
  "type": "RfqAvailableAgain",
  "data": {
    "rfq_id": "rfq-777",
    "rfq_version": 3,
    "reason": "tx_failed",
    "available_again_at": 1710000015
  }
}
```

Re-send `AcceptQuote` for the same (or a different still-valid) `order_id` to retry. `reason` for a signature timeout is `signature_timeout`; for a tx-build failure, `tx_build_failed`. Non-retryable `OrderFailed` reasons: `on_chain`, `submission_rejected`, `safety_timeout`, `shutdown` — surface to the user, do not re-accept.

### RFQ expired with no fill

```json
{
  "type": "RfqClosed",
  "data": { "rfq_id": "rfq-778", "rfq_version": 1, "reason": "expired", "closed_at": 1710000060 }
}
```

### Invite gating (closed mainnet only)

After `AuthSuccess`, an unregistered taker receives (unit variant, no `data`):

```json
{ "type": "RequireInvite" }
```

Redeem before trading:

```json
{ "type": "RedeemInvite", "data": { "request_id": "req-inv-1", "code": "abc123" } }
```

```json
{
  "type": "InviteRedeemed",
  "data": { "request_id": "req-inv-1", "referral_code": "my-code" }
}
```

An `RfqRequest` before redemption fails with a typed `InviteRequired` error (delivered as `RequestError` when the request carried a `request_id`). Devnet is open — no invite needed.

### Size-rule violation (RequestError)

`quantity` must satisfy `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "req-rfq-bad",
    "error": { "type": "Generic", "data": { "code": "invalid_quantity_size_rule", "message": "quantity violates min/max/step" } }
  }
}
```

### Reconcile after reconnect

Subscriptions and in-flight state are not replayed; re-auth, resubscribe, then query:

```json
{ "type": "GetMyActiveRfqs", "data": { "request_id": "req-rec-1" } }
{ "type": "GetOrderStatus",  "data": { "request_id": "req-rec-2", "order_id": "0x2222...2222" } }
{ "type": "GetPositions",    "data": { "request_id": "req-rec-3" } }
```

```json
{
  "type": "Positions",
  "data": {
    "request_id": "req-rec-3",
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
