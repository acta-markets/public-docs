# Acta Maker Wire Examples

Payloads illustrate wire shapes. Replace abbreviated IDs, addresses, signatures and past timestamps with real values; request/session/RFQ IDs must be UUIDs and order IDs must decode to 32 bytes. These examples are not transactions to send unchanged.


## Complete Session

### 1) Hello

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

### 2) Welcome

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

### 3) AuthRequest -> AuthChallenge -> AuthSuccess


```json
{
  "type": "AuthRequest",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2024-03-09T12:00:00Z\n"
  }
}
```

Validate the complete [canonical challenge](../reference/ws-common.md#what-to-sign) before signing. Then echo it and send the `maker_owner` pubkey:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2024-03-09T12:00:00Z\n",
    "signature": "3q7uQqYc3...base58sig",
    "pubkey": "MakerOwnerPubkeyBase58"
  }
}
```

The server verifies the signature by `quote_signing` key.

```json
{
  "type": "AuthSuccess",
  "data": {
    "session_id": "sess-123",
    "expires_at": 1710086400,
    "maker_pda": "MakerPdaBase58"
  }
}
```

`maker_pda` is the on-chain maker account PDA for this pubkey. It is `null` if the maker is not yet registered on-chain.


### 4) Snapshot

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

### 5) Subscribe

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "channels": ["rfqs"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

### 6) RfqBroadcast

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
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

### 7) Quote -> QuoteAcknowledged -> QuoteSelected

At example time `1710000000`, this quote has 140 seconds of on-chain validity and a trading cutoff at `1710000050` after the 90-second settlement buffer. Selection at `1710000010` gives the 30-second signature deadline shown below.

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "strike": 160000000000,
    "price": 50000000,
    "valid_until": 1710000140,
    "nonce": 42,
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "signature": "4kZ7kZ...base58sig"
  }
}
```

```json
{
  "type": "QuoteAcknowledged",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"
  }
}
```

```json
{
  "type": "QuoteSelected",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "taker": "TakerPubkeyBase58",
    "price": 50000000,
    "quantity": 1000000000,
    "strike": 160000000000,
    "signature_deadline": 1710000040
  }
}
```

### 8) QuoteFilled

```json
{
  "type": "QuoteFilled",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
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

### 9) RfqClosed

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 3,
    "reason": "filled",
    "closed_at": 1710000042
  }
}
```

`RfqClosed` is the terminal RFQ event for maker-side state.
For successful fills, the winning maker receives `QuoteFilled` before `RfqClosed`.
`QuoteFilled` is the fill-details event; RFQ closure should still be keyed by `RfqClosed`.

## Additional scenarios

### QuoteRefreshRequested

If the quote remains active instead of being selected, refresh fires at `1710000040`: ten seconds before its trading cutoff. The minimum new validity is that time plus the 90-second buffer and 10-second refresh lead.

```json
{
  "type": "QuoteRefreshRequested",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "strike": 160000000000,
    "min_valid_until": 1710000140,
    "reason": "expiring_soon"
  }
}
```

### Reconcile after reconnect

Send maker-private recovery reads on `/maker/data`:

```json
{ "type": "GetMyQuotes", "data": { "request_id": "a1b2c3d4-0001", "scope": "live" } }
{ "type": "GetMakerPositions", "data": { "request_id": "a1b2c3d4-0002" } }
{ "type": "GetMyTrades", "data": { "request_id": "a1b2c3d4-0003" } }
{ "type": "GetMmSummary", "data": { "request_id": "a1b2c3d4-0005" } }
```

`GetSubscriptions` stays on the quote connection because it reports `/maker`
subscription state:

```json
{ "type": "GetSubscriptions", "data": { "request_id": "a1b2c3d4-0004" } }
```

`GetMyQuotes { scope: "live" }` returns the full, unpaged owner set, including retained and selected quotes; History is queried separately with `scope: "history"`. Keep unresolved orders across reconnect and query `GetOrderStatus`; missing rows do not prove nonexecution.
Use `GetMmSummary` for dashboard bootstrap or recovery, not as a polling request. `GetMyTrades`
defaults to `50` rows and caps at `200`; `GetMyQuotes { scope: "history" }` defaults to `200`
historical rows and caps at `1000`.

### GetMyTrades (paginated)

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "a1b2c3d4-0005",
    "limit": 50
  }
}
```

Response:

```json
{
  "type": "MyTrades",
  "data": {
    "request_id": "a1b2c3d4-0005",
    "trades": [
      {
        "id": "trade-uuid-1",
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
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
    "has_more": false
  }
}
```

Next page (keyset pagination):

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "a1b2c3d4-0006",
    "limit": 50,
    "cursor": 1710000042,
    "cursor_id": "trade-uuid-1"
  }
}
```

### ReplaceQuote

```json
{
  "type": "ReplaceQuote",
  "data": {
    "old_order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "strike": 160000000000,
    "price": 55000000,
    "valid_until": 1710000140,
    "nonce": 43,
    "order_id": "0xb2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3",
    "signature": "5kA8bR...base58sig"
  }
}
```

### BatchQuotes

```json
{
  "type": "BatchQuotes",
  "data": {
    "quotes": [
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "strike": 150000000000,
        "price": 45000000,
        "valid_until": 1710000140,
        "nonce": 44,
        "order_id": "0xaaaa2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "3mR9xY...base58sig"
      },
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "strike": 160000000000,
        "price": 50000000,
        "valid_until": 1710000140,
        "nonce": 45,
        "order_id": "0xbbbb2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "7pQ2wK...base58sig"
      }
    ]
  }
}
```

### QuoteRejected

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "reason": { "cap_exceeded": { "token_oi_cap_exceeded": { "underlying_mint": "So11111111111111111111111111111111111111112", "current": 1000000000000, "limit": 1000000000000 } } },
    "message": "token_oi_cap_exceeded"
  }
}
```

### CancelAllQuotesAck

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

`CancelAllQuotesAck` follows Core application. This example means no orders were removed, not merely that the command was queued. Selected/executing obligations may remain. The receipt is correlated by `request_id`; lifecycle `QuoteCancelled` messages do not replace it.

### QuoteBestStatus

```json
{
  "type": "QuoteBestStatus",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "is_best": true,
    "current_best_price": 50000000
  }
}
```

### QuoteOutbid

```json
{
  "type": "QuoteOutbid",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "your_price": 50000000,
    "current_best_price": 55000000
  }
}
```

### CancelQuote

```json
{
  "type": "CancelQuote",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  }
}
```

### CancelAllQuotes

```json
{
  "type": "CancelAllQuotes",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "market": "MarketPdaBase58"
  }
}
```

### QuoteCancelled

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

### RfqAvailableAgain

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

### RfqClosed (expired, no fill)

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 2,
    "reason": "expired",
    "your_quote": {
      "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
      "status": "expired",
      "price": 50000000
    },
    "closed_at": 1710000050
  }
}
```

### RfqClosed (filled, maker lost)

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

### VersionMismatch

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
