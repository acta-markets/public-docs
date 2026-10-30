# Acta Maker Wire Examples

Payloads illustrate wire shapes. Replace abbreviated signatures and past timestamps with real values; request/session/RFQ IDs must be UUIDs and order IDs must decode to 32 bytes. These examples are not transactions to send unchanged.

Every frame is closed: all fields are present, absent facts are explicit `null`, and state-dependent shapes are tagged unions. The field reference is in [`../reference/maker-api.md`](../reference/maker-api.md).

Example clock: the session starts at Unix time `1710000000`. The venue window from `Welcome` is `min_quote_validity_seconds = 60` and `settlement_buffer_seconds = 50`.

## Complete Session

### 1) Hello

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

`protocol_version` must be exactly `1.0.0`. `on_disconnect` is `cancel_quotes` or `keep_quotes`. Any other version gets this frame, followed by a WebSocket close with code `4006`:

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

### 2) Welcome

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
    "signature": "<base58 ed25519 signature>",
    "pubkey": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP"
  }
}
```

The server verifies the signature by `quote_signing` key.

```json
{
  "type": "AuthSuccess",
  "data": {
    "session_id": "3a5c7e9b-1d2f-4e60-8a4c-6b8d0f2a4c71",
    "expires_at": 1710086400,
    "maker_pda": "HYe9CryCnQ3ZviCwoG8X7REFoSBK7HCjrhWWMpNtTah1"
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

`products` lists only the pair's products for this market's side. Call markets list `covered_call` and `call_spread`. Put markets list `cash_secured_put`, `put_spread` and `cash_put`.

### 5) Subscribe

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "products": ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
    "channels": ["rfqs"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

`products` must be nonempty. `[]` in a mint list means every mint. The ack lists the channels added by this request:

```json
{
  "type": "SubscribeAck",
  "data": {
    "request_id": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "subscribed": ["rfqs"]
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

### 7) Quote -> QuoteAcknowledged -> QuoteSelected

`valid_until` is at least `expires_at + min_quote_validity_seconds`, here `1710000050 + 60 = 1710000110`. The quote uses `1710000120`. Its trading cutoff is `valid_until - settlement_buffer_seconds = 1710000070`. Selection at `1710000010` gives the 30-second signature deadline shown below.

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

`price` is the gross premium per underlying unit, scaled by 1e9. The order ID is `sha256` of a 182-byte [preimage](../reference/maker-api.md#order-id-preimage) for covered calls and cash-secured puts.

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

### 8) RfqClosed

```json
{
  "type": "RfqClosed",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 3,
    "reason": "filled",
    "your_quote": {
      "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
      "status": "filled",
      "price": 5000000000
    },
    "winner": {
      "maker": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
      "price": 5000000000,
      "tx_signature": "<base58 transaction signature>"
    },
    "closed_at": 1710000042
  }
}
```

`your_quote` and `winner` are always present. On a fill every recipient gets `winner`, and the session that won gets `your_quote.status` `filled`. See [RfqClosed](../reference/maker-api.md#rfqclosed-server---maker) for the other cases.

### 9) QuoteFilled

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

`RfqClosed` is the terminal RFQ event for maker-side state.
For successful fills, the winning maker receives `RfqClosed`, then `QuoteFilled`; `expiry_ts` is `null` when the server no longer knows the market.
`QuoteFilled` is the fill-details event; RFQ closure should still be keyed by `RfqClosed`.

`QuoteFilled` is the fill fact: it carries `price` and `quantity`, with no fee or net-premium field. The authoritative net premium is `total_premium` on the `PositionUpdated` row:

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
      "state": "open",
      "quantity": 1000000000,
      "price": 5000000000,
      "total_premium": 4975000,
      "created_at": 1710000042,
      "expiry_ts": 1710600000
    },
    "update_type": "created"
  }
}
```

With a premium fee of 50 bps and a volume fee of 10 bps, the gross premium is 5000000 quote atoms. The premium fee is 25000 and the volume fee is 160000. The protocol fee is the smaller one, so `total_premium` is `5000000 - 25000 = 4975000`, which is what the taker received. The maker's cost is the gross `5000000`, and the fee is `5000000 - 4975000 = 25000`. Apply the row only if its `revision` is newer than the stored one for the same `pda`. To have the amounts before the row arrives, compute them locally as described under [QuoteFilled](../reference/maker-api.md#quotefilled-server---maker).

## Spread session

A spread or cash-put RFQ carries its strikes in `terms`. The quote repeats them exactly. On these RFQs `collateral_mint` and `settlement_mint` are both the quote mint.

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "market": {
      "chain_id": 0,
      "program_id": "33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U",
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 1800086400,
      "is_put": false,
      "collateral_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "settlement_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
    },
    "terms": { "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 },
    "side": "sell",
    "quantity": 1000000000,
    "expires_at": 1799999800,
    "taker": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "order_options": [
      { "type": "call_spread", "lower_strike": 100000000000, "upper_strike": 120000000000 }
    ],
    "sent_at_unix_ms": 1799999770000
  }
}
```

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

The order ID is `sha256` of a 190-byte preimage. Admission needs `0 < price < width` and `0 < gross < C`. For this quote, `width` is `20000000000`, `C = 20000000` and the gross premium is `5000000`. Both are in quote atoms for 1 SOL with 9 and 6 decimals. With the same fee rates, the premium fee is 25000, the volume fee is 20000, the protocol fee is 20000 and the net premium is 4980000.

A `cash_put` carries one strike. It executes as a put spread with legs `(0, K)`:

```json
{ "type": "cash_put", "strike": 150000000000 }
```

Spreads and cash puts go from `open` to `settled` only. A settled row carries the cash result:

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": { "slot": 250900000, "transaction_index": 3, "instruction_index": 0 },
      "pda": "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
      "market": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
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
      "created_at": 1799999770,
      "expiry_ts": 1800086400
    },
    "update_type": "settled"
  }
}
```

A spread quote the venue refuses carries a specific reason:

```json
{
  "type": "QuoteRejected",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "order_id": "0x0101010101010101010101010101010101010101010101010101010101010101",
    "reason": "spread_premium_invalid",
    "message": "gross premium must be positive and below width liability"
  }
}
```

A `buy` side is rejected with `taker_buy_unsupported`.

## Additional scenarios

### QuoteRefreshRequested

If the quote remains active instead of being selected, the refresh timer fires at `valid_until - settlement_buffer_seconds - quote_refresh_lead_seconds`. With a 10-second lead that is `1710000120 - 50 - 10 = 1710000060`. The minimum new validity is that time plus the 50-second buffer, the 10-second lead and one second.

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

### Reconcile after reconnect

After authentication, restore the quote plane on `/maker`. Ask what the server holds, then replace the subscription with the desired target:

```json
{ "type": "GetSubscriptions", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000004" } }
```

```json
{
  "type": "Subscriptions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000004",
    "channels": ["rfqs", "trades"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

The server holds `trades`, which the client does not want. Remove it, then subscribe:

```json
{ "type": "Unsubscribe", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000005", "channels": ["trades"] } }
```

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000006",
    "products": ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
    "channels": ["rfqs"],
    "underlying_mints": [],
    "quote_mints": []
  }
}
```

Then read the open RFQs and the live quotes:

```json
{ "type": "GetActiveRfqs", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000001" } }
{ "type": "GetMyQuotes", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000002", "scope": { "type": "live" } } }
```

On `/maker/data`, read the summary. Owner pushes start after it:

```json
{ "type": "GetMmSummary", "data": { "request_id": "a1b2c3d4-0000-4000-8000-000000000003" } }
```

Then the strategy rebuilds its position book and settles unfinished orders. The steps are in [Delivery and recovery](../reference/maker-api.md#delivery-and-recovery).

`GetMyQuotes` with `scope.type = "live"` returns the full, unpaged owner set, including retained and selected quotes. History is a separate request with `scope.type = "history"`. Use `GetMmSummary` for dashboard bootstrap or recovery, not as a polling request.

### Page the position book

If `MmSummary.positions_has_more` is `true`, read the rest from `GetMakerPositions`. `limit` is required and must be in `1..=500`. Start with `cursor.type = "start"`:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000011",
    "market": { "type": "all" },
    "underlying_mint": { "type": "all" },
    "status": [],
    "min_expiry_ts": { "type": "unbounded" },
    "limit": 500,
    "cursor": { "type": "start" }
  }
}
```

`status: []` matches every status. The response lists the newest rows first:

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
        "state": "open",
        "quantity": 1000000000,
        "price": 5000000000,
        "total_premium": 4975000,
        "created_at": 1710000042,
        "expiry_ts": 1710600000
      }
    ],
    "unrenderable": [],
    "has_more": true
  }
}
```

`has_more` is `true`, so request the next page. Set `cursor` to `after` the last row: `at` is its `created_at` and `id` is its `pda`.

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000012",
    "market": { "type": "all" },
    "underlying_mint": { "type": "all" },
    "status": [],
    "min_expiry_ts": { "type": "unbounded" },
    "limit": 500,
    "cursor": {
      "type": "after",
      "at": 1710000042,
      "id": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP"
    }
  }
}
```

Stop when `has_more` is `false`. To read only rows that can still change, set `min_expiry_ts` to `{ "type": "bounded", "value": <unix seconds> }`. It keeps positions whose market expiry is at or after the value.

### Reconcile unfinished orders

Send `GetOrderStatus` for one unfinished order and wait for its response before the next:

```json
{
  "type": "GetOrderStatus",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"
  }
}
```

```json
{
  "type": "OrderStatus",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "order_id": "0x9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": { "type": "confirmed", "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP" }
  }
}
```

`state` is `pending`, `confirmed` with `position_pda`, or `unknown`. `unknown` does not prove that the order did not execute. Keep it as an open obligation.

A rejected request comes back as a `RequestError`. Stop the loop at the first one:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000022",
    "error": {
      "type": "Generic",
      "data": { "code": "rate_limited", "message": "query rate limit exceeded" }
    },
    "delivery": null
  }
}
```

### GetMyTrades (paginated)

`limit` and `cursor` are required. `limit` is in `1..=200`.

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000031",
    "limit": 50,
    "cursor": { "type": "start" },
    "market": { "type": "all" }
  }
}
```

Response:

```json
{
  "type": "MyTrades",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000031",
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

Next page (keyset pagination). `at` is the last trade's `confirmed_at` and `id` is its trade `id`:

```json
{
  "type": "GetMyTrades",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000032",
    "limit": 50,
    "cursor": {
      "type": "after",
      "at": 1710000042,
      "id": "7c2e9a41-6b3d-4f58-9e1a-0d4b6f8a2c35"
    },
    "market": { "type": "all" }
  }
}
```

### Quote history (paginated)

History pages with the `scope` cursor. `limit` is in `1..=1000`:

```json
{
  "type": "GetMyQuotes",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000041",
    "scope": { "type": "history", "limit": 200, "cursor": { "type": "start" } }
  }
}
```

The next page uses `cursor.type = "after"` with the last row's `created_at` as `at` and its `order_id` as `id`.

A limit outside the range is rejected, not clamped:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000042",
    "error": {
      "type": "Generic",
      "data": { "code": "invalid_limit", "message": "limit must be within 1..=1000" }
    },
    "delivery": null
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

On success the server answers with `QuoteAcknowledged` carrying `replaced_order_id`.

### BatchQuotes

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

The ack has one tagged result per quote. Correlate by `order_id`:

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
          "reason": "rate_limited",
          "message": "quote rate limit exceeded"
        }
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
    "reason": {
      "cap_exceeded": {
        "token_oi_cap_exceeded": {
          "underlying_mint": "So11111111111111111111111111111111111111112",
          "current": 1000000000000,
          "limit": 1000000000000
        }
      }
    },
    "message": "Token OI cap exceeded for So11111111111111111111111111111111111111112 (1000000000000/1000000000000)"
  }
}
```

### RfqSkipped

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "market_id": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "quantity": 1000000000,
    "cap_detail": {
      "maker_insufficient_balance": { "available": 1000000, "required": 0 }
    }
  }
}
```

The caps filter runs before broadcast and knows no premium, so `required` is `0`.

### CheckQuote

```json
{
  "type": "CheckQuote",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000051",
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
    "request_id": "a1b2c3d4-0000-4000-8000-000000000051",
    "verdict": { "type": "admissible" }
  }
}
```

A refusal is `{ "type": "rejected", "cap_detail": <CapError> }`.

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

`CancelAllQuotesAck` is sent after the server applied the cancel. This example means no orders were removed, not merely that the command was queued. Selected/executing obligations may remain. The receipt is correlated by `request_id`; lifecycle `QuoteCancelled` messages do not replace it.

### QuoteBestStatus

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

### QuoteOutbid

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

### QuoteExpired

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
    "market": { "type": "only", "value": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi" }
  }
}
```

For every market, send `"market": { "type": "all" }`.

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
      "price": 5000000000
    },
    "winner": null,
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
