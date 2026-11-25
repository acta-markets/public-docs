# Acta Maker Wire Examples

Payloads illustrate wire shapes. Replace abbreviated signatures and past timestamps with real values; request/session/RFQ IDs must be UUIDs and order IDs must decode to 32 bytes. These examples are not transactions to send unchanged.

Every frame is closed: all fields are present, absent facts are tagged unions (`null` only as `next_cursor` on the last page), and state-dependent shapes are tagged unions. The field reference is in [`../reference/maker-api.md`](../reference/maker-api.md).

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
    "server_now_at_ms": 1710000000000,
    "min_quote_validity_seconds": 60,
    "settlement_buffer_seconds": 50,
    "rate_limits": {
      "connection": { "type": "bucket", "per_second": 30, "burst": 60 },
      "wallet": { "type": "unlimited" },
      "rfqs": { "type": "unlimited" },
      "queries": { "type": "bucket", "per_second": 20, "burst": 40 },
      "liquidity_provider": { "type": "shared", "per_second": 1000, "burst": 2000 }
    }
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
    "signature": "5V994XKESUifbX9zQ5ydLfse5Mkghbnb9E1yPgwmmbq3UNj4rfunARGjRD2Qyxbhjt8CfKD3UesBhJ7PWuMqVovG",
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
    "role": {
      "type": "maker",
      "maker_pda": "HYe9CryCnQ3ZviCwoG8X7REFoSBK7HCjrhWWMpNtTah1"
    }
  }
}
```

`role.maker_pda` is the on-chain maker account PDA for this pubkey.


### 4) Snapshot

```json
{
  "type": "Snapshot",
  "data": {
    "markets": [
      {
        "pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "expiry_ts": 1710600000,
        "is_put": false,
        "products": [
          "covered_call",
          "call_spread"
        ],
        "status": "active",
        "settlement": {
          "type": "pending"
        }
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

`products` must be nonempty. `[]` in a mint list means every mint. The `Reply` lists the channels added by this request:

```json
{
  "type": "Reply",
  "data": {
    "request_id": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "result": { "type": "subscribed", "channels": ["rfqs"] }
  }
}
```

### 6) RfqBroadcast

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "market_descriptor": {
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
    "terms": { "type": "covered_call", "strike": "160000000000" },
    "side": "sell",
    "quantity": "1000000000",
    "expires_at": 1710000050,
    "taker_owner": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "order_options": [
      { "type": "covered_call", "strike": "150000000000" },
      { "type": "covered_call", "strike": "160000000000" }
    ],
    "sent_at_ms": 1710000000123
  }
}
```

### 7) Quote -> QuoteUpdate live -> selected

`valid_until` is at least `expires_at + min_quote_validity_seconds`, here `1710000050 + 60 = 1710000110`. The quote uses `1710000120`. Its trading cutoff is `valid_until - settlement_buffer_seconds = 1710000070`. Selection at `1710000010` gives the 30-second signature deadline shown below.

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": "160000000000" },
    "side": "sell",
    "price": "5000000000",
    "valid_until": 1710000120,
    "nonce": "42",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "signature": "5ACJRnXT6rB2MKNMHAmf4q2u7hDfZgGw5jTtcznzJpQtgGb6S7FutRovj21ysRN1UXQsPA7eBXxNq7EoLvsXDcLj"
  }
}
```

`price` is the gross premium per underlying unit, scaled by 1e9. The order ID is `sha256` of a 182-byte [preimage](../reference/maker-api.md#order-id-preimage) for covered calls and cash-secured puts.

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "live",
      "rfq_version": 2,
      "is_best": true,
      "best_price": {
        "type": "quoted",
        "price": "5000000000"
      }
    }
  }
}
```

`live` repeats on every rank change. Selection:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "selected",
      "rfq_version": 3,
      "taker_owner": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
      "price": "5000000000",
      "quantity": "1000000000",
      "terms": {
        "type": "covered_call",
        "strike": "160000000000"
      },
      "side": "sell",
      "signature_deadline_at": 1710000040
    }
  }
}
```

### 8) RfqUpdate closed

```json
{
  "type": "RfqUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 5,
    "state": {
      "type": "closed",
      "reason": "filled",
      "winner": {
        "type": "winner",
        "maker_owner": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
        "price": "5000000000",
        "transaction": {
          "type": "sent",
          "signature": "ZzRPR6pWiL3B77tGitQFve4G2s431KhVgF1Z14vdHUMSqVFPuyCT2Bx4iK7ofDAj41hGHJhf9qAaYiR5qD5b25S"
        }
      },
      "at": 1710000042
    }
  }
}
```

Every participant gets the same close. It ends the RFQ, not your orders: each order ends with its own `QuoteUpdate`. See [RfqUpdate](../reference/maker-api.md#rfqupdate-server---maker).

### 9) QuoteUpdate filled

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "filled",
      "rfq_version": 5,
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "terms": {
        "type": "covered_call",
        "strike": "160000000000"
      },
      "side": "sell",
      "taker_owner": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
      "price": "5000000000",
      "quantity": "1000000000",
      "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
      "transaction_signature": "ZzRPR6pWiL3B77tGitQFve4G2s431KhVgF1Z14vdHUMSqVFPuyCT2Bx4iK7ofDAj41hGHJhf9qAaYiR5qD5b25S",
      "filled_at": 1710000042
    }
  }
}
```

The winning maker receives `RfqUpdate` `closed`, then `QuoteUpdate` `filled`, which ends the order.

`filled` is the fill fact: it carries `price` and `quantity`, with no fee or net-premium field. The authoritative net premium is `total_premium` on the `PositionUpdated` row:

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": {
        "slot": 250000000,
        "transaction_index": 12,
        "instruction_index": 1
      },
      "pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_symbol": "SOL",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_symbol": "USDC",
      "quote_decimals": 6,
      "terms": {
        "type": "covered_call",
        "strike": "160000000000"
      },
      "side": "sell",
      "state": "open",
      "quantity": "1000000000",
      "price": "5000000000",
      "total_premium": "4975000",
      "created_at": 1710000042,
      "expiry_ts": 1710600000,
      "underlying_price": {
        "type": "available",
        "price": "151000000000",
        "published_at": 1710000010
      }
    },
    "update_type": "created"
  }
}
```

With a premium fee of 50 bps and a volume fee of 10 bps, the gross premium is 5000000 quote atoms. The premium fee is 25000 and the volume fee is 160000. The protocol fee is the smaller one, so `total_premium` is `5000000 - 25000 = 4975000`, which is what the taker received. The maker's cost is the gross `5000000`, and the fee is `5000000 - 4975000 = 25000`. Apply the row only if its `revision` is newer than the stored one for the same `pda`. To have the amounts before the row arrives, compute them locally as described under [Cash settlement and fees](../reference/maker-api.md#cash-settlement-and-fees).

## Spread session

A spread or cash-put RFQ carries its strikes in `terms`. The quote repeats them exactly. On these RFQs `collateral_mint` and `settlement_mint` are both the quote mint.

```json
{
  "type": "RfqBroadcast",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "market_descriptor": {
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
    "terms": { "type": "call_spread", "lower_strike": "100000000000", "upper_strike": "120000000000" },
    "side": "sell",
    "quantity": "1000000000",
    "expires_at": 1799999800,
    "taker_owner": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
    "order_options": [
      { "type": "call_spread", "lower_strike": "100000000000", "upper_strike": "120000000000" }
    ],
    "sent_at_ms": 1799999770000
  }
}
```

```json
{
  "type": "Quote",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "terms": { "type": "call_spread", "lower_strike": "100000000000", "upper_strike": "120000000000" },
    "side": "sell",
    "price": "5000000000",
    "valid_until": 1799999900,
    "nonce": "999",
    "order_id": "0101010101010101010101010101010101010101010101010101010101010101",
    "signature": "5ACJRnXT6rB2MKNMHAmf4q2u7hDfZgGw5jTtcznzJpQtgGb6S7FutRovj21ysRN1UXQsPA7eBXxNq7EoLvsXDcLj"
  }
}
```

The order ID is `sha256` of a 190-byte preimage. Admission needs `0 < price < width` and `0 < gross < C`. For this quote, `width` is `20000000000`, `C = 20000000` and the gross premium is `5000000`. Both are in quote atoms for 1 SOL with 9 and 6 decimals. With the same fee rates, the premium fee is 25000, the volume fee is 20000, the protocol fee is 20000 and the net premium is 4980000.

A `cash_put` carries one strike. It executes as a put spread with legs `(0, K)`:

```json
{ "type": "cash_put", "strike": "150000000000" }
```

Spreads and cash puts go from `open` to `settled` only. A settled row carries the cash result:

```json
{
  "type": "PositionUpdated",
  "data": {
    "position": {
      "revision": {
        "slot": 250900000,
        "transaction_index": 3,
        "instruction_index": 0
      },
      "pda": "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
      "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "underlying_symbol": "SOL",
      "underlying_decimals": 9,
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "quote_symbol": "USDC",
      "quote_decimals": 6,
      "terms": {
        "type": "call_spread",
        "lower_strike": "100000000000",
        "upper_strike": "120000000000"
      },
      "side": "sell",
      "state": "settled",
      "settlement": {
        "type": "cash",
        "settlement_price": "110000000000",
        "holder_payoff": "10000000",
        "writer_return": "10000000"
      },
      "quantity": "1000000000",
      "price": "5000000000",
      "total_premium": "4980000",
      "created_at": 1799999770,
      "expiry_ts": 1800086400,
      "underlying_price": {
        "type": "unavailable"
      }
    },
    "update_type": "settled"
  }
}
```

A spread quote the venue refuses carries a specific reason:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "00000000-0000-0000-0000-000000000001",
    "order_id": "0101010101010101010101010101010101010101010101010101010101010101",
    "state": {
      "type": "rejected",
      "reason": "spread_premium_invalid",
      "message": "gross premium must be positive and below width liability"
    }
  }
}
```

A `buy` side arrives only on the `taker_buy_rfqs` channel, and the maker writes it from Maker PDA custody.

## Additional scenarios

### QuoteUpdate refresh_requested

If the quote remains active instead of being selected, the refresh timer fires at `valid_until - settlement_buffer_seconds - quote_refresh_lead_seconds`. With a 10-second lead that is `1710000120 - 50 - 10 = 1710000060`. The minimum new validity is that time plus the 50-second buffer, the 10-second lead and one second.

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "refresh_requested",
      "rfq_version": 4,
      "terms": {
        "type": "covered_call",
        "strike": "160000000000"
      },
      "side": "sell",
      "min_valid_until": 1710000121,
      "reason": "expiring_soon"
    }
  }
}
```

This ends the order. To stay in the RFQ, sign a new quote with a fresh `nonce` and `order_id` and `valid_until >= min_valid_until`.

### Reconcile after reconnect

After authentication, restore the quote plane on `/maker`. Ask what the server holds, then replace the subscription with the desired target:

```json
{
  "type": "GetSubscriptions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000004",
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

```json
{
  "type": "Subscriptions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000004",
    "channels": [
      "rfqs",
      "trades"
    ],
    "underlying_mints": [],
    "quote_mints": [],
    "next_cursor": null
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
{
  "type": "GetActiveRfqs",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000001",
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

```json
{
  "type": "GetMyQuotes",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000002",
    "scope": {
      "type": "live"
    },
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

On `/maker/data`, read the summary. Owner pushes start after it:

```json
{
  "type": "GetMmSummary",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000003",
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

Then the strategy rebuilds its position book and settles unfinished orders. The steps are in [Delivery and recovery](../reference/maker-api.md#delivery-and-recovery).

`GetMyQuotes` with `scope.type = "live"` pages the owner set, including retained and selected quotes. History is a separate request with `scope.type = "history"`. Use `GetMmSummary` for dashboard bootstrap or recovery, not as a polling request.

### Page the position book

Use [common paging](../reference/ws-common.md#paging) for maker positions. Start with the following request:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000011",
    "market_pda": {
      "type": "all"
    },
    "underlying_mint": {
      "type": "all"
    },
    "status": [],
    "min_expiry_ts": {
      "type": "unbounded"
    },
    "limit": 200,
    "cursor": {
      "type": "start"
    }
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
        "revision": {
          "slot": 250000000,
          "transaction_index": 12,
          "instruction_index": 1
        },
        "pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP",
        "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "underlying_symbol": "SOL",
        "underlying_decimals": 9,
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "quote_symbol": "USDC",
        "quote_decimals": 6,
        "terms": {
          "type": "covered_call",
          "strike": "160000000000"
        },
        "side": "sell",
        "state": "open",
        "quantity": "1000000000",
        "price": "5000000000",
        "total_premium": "4975000",
        "created_at": 1710000042,
        "expiry_ts": 1710600000,
        "underlying_price": { "type": "available", "price": "151000000000", "published_at": 1710000040 }
      }
    ],
    "unrenderable": [],
    "next_cursor": null
  }
}
```

When `next_cursor` is non-null, wrap that opaque string in an `after` cursor. The following frame shows the first-page shape:

```json
{
  "type": "GetMakerPositions",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000012",
    "market_pda": {
      "type": "all"
    },
    "underlying_mint": {
      "type": "all"
    },
    "status": [],
    "min_expiry_ts": {
      "type": "unbounded"
    },
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

Stop when `next_cursor` is null. Keep the same filters throughout a read.

### Reconcile unfinished orders

Send `GetOrderStatus` for one unfinished order and wait for its response before the next:

```json
{
  "type": "GetOrderStatus",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a"
  }
}
```

```json
{
  "type": "OrderStatus",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000021",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": { "type": "confirmed", "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP" }
  }
}
```

`state` is `pending`, `confirmed` with `position_pda`, `expired`, or `unknown`. `expired` proves the order never executed and never can. `unknown` does not prove that the order did not execute. Keep it as an open obligation.

A rejected request comes back as a `Reply` `error`. Stop the loop at the first one and send again after `retry_after_ms`:

```json
{
  "type": "Reply",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000022",
    "result": {
      "type": "error",
      "error": {
        "type": "RateLimited",
        "data": {
          "retry_after_ms": 50
        }
      },
      "delivery": "not_applicable"
    }
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
    "market_pda": { "type": "all" }
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
        "terms": {
          "type": "covered_call",
          "strike": "160000000000"
        },
        "side": "sell",
        "taker_owner": "5DNHbugre3SXXogpZroA5SEsENdAd69AfNK5Z7ZCWr84",
        "quantity": "1000000000",
        "price": "5000000000",
        "transaction_signature": "ZzRPR6pWiL3B77tGitQFve4G2s431KhVgF1Z14vdHUMSqVFPuyCT2Bx4iK7ofDAj41hGHJhf9qAaYiR5qD5b25S",
        "position": {
          "type": "confirmed",
          "position_pda": "75NqraLLofiLUpNQg2wUPG6CJpxcaGEZUwuNrnAgvcTP"
        },
        "confirmed_at": 1710000042
      }
    ],
    "next_cursor": null
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
      "type": "start"
    },
    "market_pda": {
      "type": "all"
    }
  }
}
```

### Quote history (paginated)

History uses top-level paging fields. See [common paging](../reference/ws-common.md#paging):

```json
{
  "type": "GetMyQuotes",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000041",
    "scope": {
      "type": "history"
    },
    "limit": 200,
    "cursor": {
      "type": "start"
    }
  }
}
```

The next page uses `cursor.type = "after"` with the last row's `created_at` as `at` and its `order_id` as `id`.

Normal SDK list calls collect all pages through one helper. Live quotes and catalogs are paged too. Do not use query reads as polling paths.

```json
{
  "type": "Reply",
  "data": {
    "request_id": "a1b2c3d4-0000-4000-8000-000000000042",
    "result": {
      "type": "error",
      "error": {
        "type": "Generic",
        "data": {
          "code": "invalid_limit",
          "message": "limit must be within 1..=200"
        }
      },
      "delivery": "not_applicable"
    }
  }
}
```

### ReplaceQuote

```json
{
  "type": "ReplaceQuote",
  "data": {
    "replaced_order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "terms": { "type": "covered_call", "strike": "160000000000" },
    "side": "sell",
    "price": "5500000000",
    "valid_until": 1710000120,
    "nonce": "43",
    "order_id": "b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3",
    "signature": "5ACJRnXT6rB2MKNMHAmf4q2u7hDfZgGw5jTtcznzJpQtgGb6S7FutRovj21ysRN1UXQsPA7eBXxNq7EoLvsXDcLj"
  }
}
```

On success the old order ends first, then the new order gets `live`, both at the same `rfq_version`:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": { "type": "cancelled", "rfq_version": 4, "reason": "replaced" }
  }
}
```

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3",
    "state": {
      "type": "live",
      "rfq_version": 4,
      "is_best": true,
      "best_price": { "type": "quoted", "price": "5500000000" }
    }
  }
}
```

An `order_id` the server already knows is rejected for this submission; if you already hold that order live or selected, it keeps its state:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "rejected",
      "reason": "duplicate_order_id",
      "message": "order_id already belongs to a known order"
    }
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
        "terms": { "type": "covered_call", "strike": "150000000000" },
        "side": "sell",
        "price": "4500000000",
        "valid_until": 1710000120,
        "nonce": "44",
        "order_id": "aaaa2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "5ACJRnXT6rB2MKNMHAmf4q2u7hDfZgGw5jTtcznzJpQtgGb6S7FutRovj21ysRN1UXQsPA7eBXxNq7EoLvsXDcLj"
      },
      {
        "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
        "terms": { "type": "covered_call", "strike": "160000000000" },
        "side": "sell",
        "price": "5000000000",
        "valid_until": 1710000120,
        "nonce": "45",
        "order_id": "bbbb2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
        "signature": "5ACJRnXT6rB2MKNMHAmf4q2u7hDfZgGw5jTtcznzJpQtgGb6S7FutRovj21ysRN1UXQsPA7eBXxNq7EoLvsXDcLj"
      }
    ]
  }
}
```

There is no batch-level answer: each quote gets its own `QuoteUpdate`, `live` or `rejected`, exactly as if sent alone. Correlate by `order_id`.

### QuoteUpdate rejected

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "rejected",
      "reason": {
        "cap_exceeded": {
          "maker_position_cap_exceeded": {
            "current": 3,
            "limit": 3
          }
        }
      },
      "message": "Maker position cap exceeded (3/3)"
    }
  }
}
```

The server reserves capacity for every live quote, so a quote beyond your balance or limits is refused here, not at fill. `rejected` refuses this submission and carries no `rfq_version`.

### RfqSkipped

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "quantity": "1000000000",
    "cap_detail": {
      "maker_insufficient_balance": { "available": "1000000", "required": "0" }
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
    "market_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
    "terms": { "type": "call_spread", "lower_strike": "100000000000", "upper_strike": "120000000000" },
    "side": "sell",
    "quantity": "1000000000",
    "price": "5000000000"
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

### Reply to CancelAllQuotes

```json
{
  "type": "Reply",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "result": { "type": "accepted" }
  }
}
```

Sent after the server applied the cancel. Every order it removed ends with its own `QuoteUpdate` `cancelled`, `reason` `requested`; none means no orders were removed, not merely that the command was queued. Selected/executing obligations may remain.

### QuoteUpdate live (rank change)

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "live",
      "rfq_version": 3,
      "is_best": false,
      "best_price": {
        "type": "quoted",
        "price": "5500000000"
      }
    }
  }
}
```

Another maker now has the best price. A repeat `live` is a new rank, not a new quote.



### QuoteUpdate expired

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "expired",
      "rfq_version": 4,
      "reason": "expired"
    }
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

```json
{
  "type": "Reply",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "result": { "type": "accepted" }
  }
}
```

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": { "type": "cancelled", "rfq_version": 5, "reason": "requested" }
  }
}
```

### CancelAllQuotes

```json
{
  "type": "CancelAllQuotes",
  "data": {
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "market_pda": { "type": "only", "value": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi" }
  }
}
```

For every market, send `"market_pda": { "type": "all" }`.

### QuoteUpdate cancelled (server-initiated)

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": { "type": "cancelled", "rfq_version": 5, "reason": "risk_check" }
  }
}
```

The same state your own cancel produces, with the server's reason (`risk_check`, `maker_disconnected`).

### RfqUpdate locked

```json
{
  "type": "RfqUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 3,
    "state": {
      "type": "locked",
      "at": 1710000030
    }
  }
}
```

### RfqUpdate available

```json
{
  "type": "RfqUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 4,
    "state": {
      "type": "available",
      "reason": "signature_timeout",
      "at": 1710000045
    }
  }
}
```

The selected order itself ends with:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "released",
      "rfq_version": 4,
      "reason": "signature_timeout"
    }
  }
}
```

### RfqUpdate closed (expired, no fill)

```json
{
  "type": "RfqUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 5,
    "state": {
      "type": "closed",
      "reason": "expired",
      "winner": {
        "type": "none"
      },
      "at": 1710000050
    }
  }
}
```

Each of your orders still in the book ends with:

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "lost",
      "rfq_version": 5,
      "reason": "expired"
    }
  }
}
```

### RfqUpdate closed (filled, maker lost)

```json
{
  "type": "RfqUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "rfq_version": 6,
    "state": {
      "type": "closed",
      "reason": "filled",
      "winner": {
        "type": "winner",
        "maker_owner": "A87ysNWAmmqQHafLSQGbYSLhtcV3BjXctR88EWuSobGP",
        "price": "5500000000",
        "transaction": {
          "type": "sent",
          "signature": "ZzRPR6pWiL3B77tGitQFve4G2s431KhVgF1Z14vdHUMSqVFPuyCT2Bx4iK7ofDAj41hGHJhf9qAaYiR5qD5b25S"
        }
      },
      "at": 1710000042
    }
  }
}
```

```json
{
  "type": "QuoteUpdate",
  "data": {
    "rfq_id": "8f3e7e6a-4f5c-4b7c-9f1d-1f2a3b4c5d6e",
    "order_id": "9d1c2a6a0c2f5e7d9b4d8d8f2a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a",
    "state": {
      "type": "lost",
      "rfq_version": 6,
      "reason": "filled"
    }
  }
}
```
