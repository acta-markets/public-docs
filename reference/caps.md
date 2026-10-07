# Capacity Limits (Caps)

Caps enforce risk limits on quote submission. They apply at three scopes:

- **Platform** — global open-interest and notional ceilings per underlying mint and per quote mint.
- **Market** — open-interest ceilings per market PDA (underlying, quote, expiry and put/call), shared by all its strikes.
- **Maker** — per-account position count, notional exposure, and deposited premium balance.

A breach detected at quote-validation produces `QuoteRejected` with `reason: { "cap_exceeded": <CapError> }`. A breach predictable from the maker's current state pre-filters the RFQ before broadcast and produces `RfqSkipped` instead. `GetTokenCaps` returns platform token, market, and quote-mint caps; `GetMyCaps` returns the maker layer.

## Platform caps

### Token OI cap

Maximum open interest allowed for a given underlying mint across all markets.

| Field | Type | Description |
|-------|------|-------------|
| `underlying_mint` | pubkey (base58) | Underlying token mint |
| `symbol` | string | Human-readable symbol, e.g. `"SOL"` |
| `current_oi` | u64 | Current aggregate OI (underlying atomic units) |
| `max_oi` | u64 | Maximum allowed OI |
| `utilization` | f64 | `current_oi / max_oi`; not clamped to 1.0 |

### Market OI cap

Maximum OI for a single market. A market is keyed by its on-chain PDA —
one budget per (underlying, quote, expiry, put/call). **All strikes of an
expiry share that one budget**; there is no per-strike capacity.

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

`markets` and `quotes` are omitted only when the corresponding arrays are empty. Missing token, market, or quote entries mean no budget is configured for that scope; the backend treats that scope as uncapped. A configured platform budget of `0` is the opposite — zero capacity, everything rejected. (Maker-level limits use the reverse convention: `0` there means unlimited.)

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

## How quotes consume capacity

Capacity is a concurrent-risk budget, not a lifetime quota — and **a live
quote holds capacity the moment it is accepted by the venue**, not on fill:

- Submitting a quote atomically reserves the full quantity/premium against
  every applicable dimension (platform OI and notional, your position count,
  your notional and committed premium) for the quote's validity window.
- The reservation is released when the quote is cancelled, replaced (a
  replacement only needs headroom for the delta), expires, or is rejected —
  and converts into position exposure when it fills.
- Your own polled numbers (`GetTokenCaps`, `GetMyCaps`) **include your live
  reservations**: quoting many RFQs concurrently visibly consumes your
  position-count and notional ceilings even with zero fills. Size
  `max_open_positions` and notional limits for your intended quoting breadth,
  not just expected inventory.
- Competing quotes on one RFQ each reserve full size while the auction runs.
  Near a platform cap this crowds out later quoters; expect
  `token_oi_cap_exceeded` rejections close to the cap even when your own
  exposure is small.

## How caps affect quoting

| Scenario | What happens |
|----------|--------------|
| Platform OI cap reached | `RfqSkipped` — RFQ not broadcast to you |
| Maker position cap reached | `RfqSkipped` — RFQ pre-filtered |
| Maker notional cap reached | `RfqSkipped` — RFQ pre-filtered (quantity known ahead) |
| Maker balance is zero | `RfqSkipped` — RFQ pre-filtered |
| Maker balance insufficient for premium | `QuoteRejected` with `reason: cap_exceeded` |
| Quote notional cap reached | `QuoteRejected` with `reason: cap_exceeded` |
| Caps authority temporarily unavailable | `RfqSkipped` / `QuoteRejected` with `caps_unavailable` — retry with backoff |

`QuoteRejected` carries the machine-readable detail inside the reason:
`reason` is `{ "cap_exceeded": <CapError> }`, where the `CapError` is keyed by
the exact code from the table below and carries the failing dimension's
`current`/`limit` amounts. `RfqSkipped` carries the same structure as
`cap_detail`. Parse those, not the prose `message`.

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
| `caps_unavailable` | — | Caps authority temporarily unavailable (venue state, not your account). Retry with backoff |

## Monitoring caps

`GetMyCaps` is cheap enough to poll; 60 seconds is a reasonable default. `utilization` on platform caps is `current / max` and is **not clamped**: values above `1.0` are a real over-budget state (limits can be tightened below live usage). A zero platform limit reports saturated utilization `1.0`, not spare capacity. Many makers stop submitting new quotes once utilization exceeds `0.9`, leaving headroom for in-flight quotes.

`PositionUpdated` carries `caps_snapshot` only for `update_type: "funded"`; fills and settlements do not push it. Poll `GetMyCaps` (or refetch on `ChainEvent` notifications) to track capacity freed by settlement.

`caps_unavailable` rejections mean the venue's risk projection is catching up (typically after a fill, for well under a second, or during listener incidents). They are retryable venue state: back off briefly and requote. They do not mean your balance or limits changed.

## Freeing capacity

Reserved capacity returns when a quote is cancelled, replaced, expires, or is rejected. Position exposure returns when the position settles. Deposited premium not backing an open position or live quote can be retrieved via the on-chain `WithdrawPremium` instruction.

## Reference

- Wire-level request/response framing for `GetTokenCaps` / `GetMyCaps` (envelope shape, `request_id` correlation): [maker-api.md](maker-api.md).
- `RfqSkipped` broadcast (sent in place of `RfqBroadcast` when a cap blocks delivery): [maker-api.md](maker-api.md#rfqskipped-server---maker).
