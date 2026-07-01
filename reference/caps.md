# Capacity Limits (Caps)

Caps enforce risk limits on quote submission. They apply at three scopes:

- **Platform** — global open-interest and notional ceilings per underlying mint and per quote mint.
- **Market** — open-interest ceilings per individual market (one strike-and-expiry pair).
- **Maker** — per-account position count, notional exposure, and deposited premium balance.

A breach detected at quote-validation produces `QuoteRejected` with `reason = cap_exceeded`. A breach predictable from the maker's current state pre-filters the RFQ before broadcast and produces `RfqSkipped` instead. `GetTokenCaps` returns platform token, market, and quote-mint caps; `GetMyCaps` returns the maker layer.

## Platform caps

### Token OI cap

Maximum open interest allowed for a given underlying mint across all markets.

| Field | Type | Description |
|-------|------|-------------|
| `underlying_mint` | pubkey (base58) | Underlying token mint |
| `symbol` | string | Human-readable symbol, e.g. `"SOL"` |
| `current_oi` | u64 | Current aggregate OI (underlying atomic units) |
| `max_oi` | u64 | Maximum allowed OI |
| `utilization` | f64 | `current_oi / max_oi`, range 0.0-1.0 |

### Market OI cap

Maximum OI for a single market (specific strike / expiry pair).

| Field | Type | Description |
|-------|------|-------------|
| `market_id` | string | Market identifier |
| `current_oi` | u64 | Current market OI |
| `max_oi` | u64 | Maximum allowed OI |
| `utilization` | f64 | `current_oi / max_oi` |

### Quote notional cap

Maximum notional exposure per quote mint (e.g. USDC).

| Field | Type | Description |
|-------|------|-------------|
| `quote_mint` | pubkey (base58) | Quote token mint |
| `symbol` | string | Human-readable symbol, e.g. `"USDC"` |
| `current_notional` | u64 | Current notional (quote atomic units) |
| `max_notional` | u64 | Maximum allowed notional |
| `utilization` | f64 | `current_notional / max_notional` |

### Querying platform caps

Send `GetTokenCaps` for platform-level caps. `request_id` is required. `include_markets` exists in the wire type, but the backend returns configured token, market, and quote-mint caps regardless of its value.

```json
{ "type": "GetTokenCaps", "data": { "request_id": "uuid" } }
```

Response `TokenCaps`:

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
        "market_id": "SOL-20260401-15000-C",
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
    ]
  }
}
```

`markets` and `quotes` are omitted only when the corresponding arrays are empty. Missing token, market, or quote entries mean no budget is configured for that scope; the backend treats that scope as uncapped.

## Maker caps

### Position count

Maximum number of simultaneously open positions for your maker account.

| Field | Type | Description |
|-------|------|-------------|
| `current` | u32 | Open positions held now |
| `limit` | u32 | Maximum allowed positions |

### Notional per underlying

Maximum notional exposure per underlying mint for your maker account.

| Field | Type | Description |
|-------|------|-------------|
| `underlying_mint` | pubkey (base58) | Underlying token mint |
| `symbol` | string | Human-readable symbol |
| `current` | u64 | Current notional exposure |
| `limit` | u64 | Maximum allowed notional |

### Balance

Available balance for quoting. Only the `available` portion can back new quotes.

| Field | Type | Description |
|-------|------|-------------|
| `mint` | pubkey (base58) | Token mint |
| `symbol` | string | Human-readable symbol |
| `decimals` | u8 | On-chain mint decimals (divide atomic balances by `10^decimals` to render UI amounts) |
| `deposited` | u64 | Total deposited (atomic units) |
| `committed` | u64 | Locked by open positions / active quotes |
| `available` | u64 | `deposited - committed` |

### Querying maker caps

Send `GetMyCaps` for your maker-specific limits. `request_id` is required.

```json
{ "type": "GetMyCaps", "data": { "request_id": "uuid" } }
```

Response `MyCaps`:

```json
{
  "type": "MyCaps",
  "data": {
    "request_id": "uuid",
    "positions": {
      "current": 12,
      "limit": 50
    },
    "notional": [
      {
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "symbol": "SOL",
        "current": 800000000000,
        "limit": 2000000000000
      }
    ],
    "balances": [
      {
        "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "decimals": 6,
        "deposited": 5000000000,
        "committed": 3200000000,
        "available": 1800000000
      }
    ]
  }
}
```

## How caps affect quoting

| Scenario | What happens |
|----------|--------------|
| Token OI cap reached | `RfqSkipped` — RFQ not broadcast to you |
| Maker position cap reached | `RfqSkipped` — RFQ pre-filtered |
| Maker notional cap reached | `QuoteRejected` with `reason: cap_exceeded` |
| Maker balance insufficient | `QuoteRejected` with `reason: cap_exceeded` |
| Quote notional cap reached | `QuoteRejected` with `reason: cap_exceeded` |

When an RFQ is skipped you receive an `RfqSkipped` notification:

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "b3f1a2c4-...",
    "market_id": "SOL-20260401-15000-C",
    "quantity": 1000000000,
    "reason": "token_oi_cap_exceeded"
  }
}
```

## CapError variants

These variants appear in error responses when a cap is breached.

| Variant | Fields | Description |
|---------|--------|-------------|
| `token_oi_cap_exceeded` | `underlying_mint`, `current`, `limit` | Platform OI limit reached |
| `market_oi_cap_exceeded` | `market_id`, `current`, `limit` | Market-specific OI limit |
| `maker_position_cap_exceeded` | `current`, `limit` | Too many open positions |
| `maker_notional_cap_exceeded` | `underlying_mint`, `current`, `limit` | Notional exposure limit |
| `maker_insufficient_balance` | `available`, `required` | Not enough deposited premium |
| `quote_notional_cap_exceeded` | `quote_mint`, `current`, `limit` | Quote mint notional limit |
| `maker_quote_notional_cap_exceeded` | `quote_mint`, `current`, `limit` | Maker quote-mint premium commitment limit |

## Monitoring caps

`GetMyCaps` is cheap enough to poll; 60 seconds is a reasonable default. `utilization` on platform caps is `current / max`, normalized to `[0.0, 1.0]`. Many makers stop submitting new quotes once utilization exceeds `0.9`, leaving headroom for in-flight quotes.

`PositionUpdated` carries `caps_snapshot` after each fill. Use it to update local caps without polling after every trade.

## Freeing capacity

Open interest is held by open positions, not by quotes. Cancelled and expired quotes do not consume OI; only fills do. Capacity is released when a position settles. Deposited premium not backing an open position can be retrieved via the on-chain `WithdrawPremium` instruction.

## Reference

- Wire-level request/response framing for `GetTokenCaps` / `GetMyCaps` (envelope shape, `request_id` correlation): [maker-api.md](maker-api.md).
- `RfqSkipped` broadcast (sent in place of `RfqBroadcast` when a cap blocks delivery): [maker-api.md](maker-api.md#rfqskipped-server---maker).
