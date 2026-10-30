# Capacity Limits (Caps)

Caps enforce risk limits on quote submission. They apply at three scopes:

- **Platform** — global open-interest and notional ceilings per underlying mint and per quote mint, and a liability ceiling per quote mint for cash-settled spreads.
- **Market** — open-interest ceilings per market PDA (underlying, quote, expiry and put/call), shared by all its strikes.
- **Maker** — per-account position count, notional exposure, deposited premium balance, and spread liability per quote mint.

A breach detected at quote-validation produces `QuoteRejected` with `reason: { "cap_exceeded": <CapError> }`. A breach predictable from the maker's current state pre-filters the RFQ before broadcast and produces `RfqSkipped` instead. `GetTokenCaps` returns platform token, market, quote-mint and quote-liability caps; `GetMyCaps` returns the maker layer. `CheckQuote` runs the quote admission checks without reserving anything.

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
| `market_id` | pubkey (base58) | Market PDA |
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

### Quote liability cap

Maximum cash-settled liability per quote mint. The liability of a spread or cash put is its collateral `C` (`ceil(width × quantity)`), the most the position can pay out.

| Field | Type | Description |
|-------|------|-------------|
| `quote_mint` | pubkey (base58) | Quote token mint |
| `symbol` | string | Human-readable symbol |
| `current_liability` | u64 | Current liability (quote atomic units) |
| `policy` | object | `QuoteLiabilityPolicy`, see below |

`policy` is a tagged union on `type`:

| `policy.type` | Fields | Meaning |
|---------------|--------|---------|
| `unconfigured` | — | No limit is configured for this quote mint. Liability is counted but not enforced |
| `unlimited` | — | A limit is configured and set to unlimited (maker scope only) |
| `limited` | `max_liability`, `headroom` | `max_liability` is the ceiling; `headroom` is `max_liability - current_liability`, floored at 0 |

### Querying platform caps

Send `GetTokenCaps` for platform-level caps. `request_id` and `include_markets` are required. The backend returns token, market, quote-mint and quote-liability caps regardless of the value of `include_markets`.

```json
{ "type": "GetTokenCaps", "data": { "request_id": "uuid", "include_markets": true } }
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
        "market_id": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
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
    ],
    "quote_liabilities": [
      {
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "current_liability": 50000000,
        "policy": { "type": "limited", "max_liability": 100000000, "headroom": 50000000 }
      }
    ]
  }
}
```

All four arrays are always present; an empty scope is `[]`. Missing token, market, or quote entries mean no budget is configured for that scope; the backend treats that scope as uncapped. A configured platform budget of `0` is the opposite — zero capacity, everything rejected. (Maker-level limits use the reverse convention: `0` there means unlimited.)

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

### Maker quote liability

Maximum spread liability per quote mint for your maker account. Same fields as the platform [quote liability cap](#quote-liability-cap). `unconfigured` means no limit is configured, and `unlimited` means the configured limit is `0`.

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
    ],
    "quote_liabilities": [
      {
        "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "symbol": "USDC",
        "current_liability": 20000000,
        "policy": { "type": "limited", "max_liability": 100000000, "headroom": 80000000 }
      }
    ]
  }
}
```

### Checking a quote

`CheckQuote` is a maker-only dry run of the cap admission that `Quote` runs. It reserves nothing, so a passing check is advice, not a hold. All fields are required.

```json
{
  "type": "CheckQuote",
  "data": {
    "request_id": "uuid",
    "terms": { "type": "covered_call", "strike": 150000000000 },
    "side": "sell",
    "market_id": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "quantity": 1000000000,
    "price": 5000000000
  }
}
```

`price` is the gross price per underlying unit, 1e9 scale, as on `Quote`. The response `QuoteCheck` carries `verdict`, a tagged union on `type`:

```json
{ "type": "QuoteCheck", "data": { "request_id": "uuid", "verdict": { "type": "admissible" } } }
```

```json
{
  "type": "QuoteCheck",
  "data": {
    "request_id": "uuid",
    "verdict": {
      "type": "rejected",
      "cap_detail": { "maker_position_cap_exceeded": { "current": 50, "limit": 50 } }
    }
  }
}
```

`cap_detail` is the `CapError` that a `QuoteRejected` would carry. `"caps_unavailable"` means retry, not a cap.

## How quotes consume capacity

Capacity is a concurrent-risk budget, not a lifetime quota — and **a live
quote holds capacity the moment it is accepted by the venue**, not on fill:

- Submitting a quote atomically reserves the full quantity/premium against
  every applicable dimension (platform OI, notional and spread liability, your
  position count, your notional, committed premium and spread liability) for
  the quote's validity window.
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
| Maker balance is zero | `RfqSkipped` — RFQ pre-filtered; `maker_insufficient_balance.required` is `0` because nothing is priced yet |
| Platform or maker spread liability cap reached | `QuoteRejected` with `reason: cap_exceeded` |
| Maker balance insufficient for premium | `QuoteRejected` with `reason: cap_exceeded` |
| Quote notional cap reached | `QuoteRejected` with `reason: cap_exceeded` |
| Caps authority temporarily unavailable | `RfqSkipped` / `QuoteRejected` with `cap_detail` / `cap_exceeded` set to `caps_unavailable` — retry with backoff |

`QuoteRejected` carries the machine-readable detail inside the reason:
`reason` is `{ "cap_exceeded": <CapError> }`. `RfqSkipped` and `QuoteCheck` carry the
same `CapError` as `cap_detail`. Parse those, not the prose `message`.

A `CapError` is externally tagged by the code from the table below. A variant with fields
is an object with the code as its only key and the failing dimension's amounts inside it:

```json
{ "token_oi_cap_exceeded": { "underlying_mint": "So11111111111111111111111111111111111111112", "current": 900000000, "limit": 1000000000 } }
```

`caps_unavailable` has no fields and is the bare string `"caps_unavailable"`.

When an RFQ is skipped you receive an `RfqSkipped` notification. It is sent before the RFQ is
broadcast, so `maker_insufficient_balance.required` is always `0` there:

```json
{
  "type": "RfqSkipped",
  "data": {
    "rfq_id": "6f9619ff-8b86-d011-b42d-00c04fc964ff",
    "market_id": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "quantity": 1000000000,
    "cap_detail": {
      "token_oi_cap_exceeded": {
        "underlying_mint": "So11111111111111111111111111111111111111112",
        "current": 900000000000,
        "limit": 1000000000000
      }
    }
  }
}
```

## CapError variants

These variants appear in `QuoteRejected`, `RfqSkipped` and `QuoteCheck` when a cap is breached. Amounts are numbers; `current` and `limit` of spread liability variants are in quote atoms.

| Variant | Fields | Description |
|---------|--------|-------------|
| `token_oi_cap_exceeded` | `underlying_mint`, `current`, `limit` | Platform OI limit reached |
| `market_oi_cap_exceeded` | `market_id`, `current`, `limit` | Market-specific OI limit |
| `maker_position_cap_exceeded` | `current`, `limit` | Too many open positions |
| `maker_notional_cap_exceeded` | `underlying_mint`, `current`, `limit` | Notional exposure limit |
| `maker_insufficient_balance` | `available`, `required` | Not enough deposited balance. `required` is `0` in `RfqSkipped` |
| `quote_liability_cap_exceeded` | `quote_mint`, `current`, `limit` | Platform spread liability limit for the quote mint |
| `quote_notional_cap_exceeded` | `quote_mint`, `current`, `limit` | Quote mint notional limit |
| `maker_quote_notional_cap_exceeded` | `quote_mint`, `current`, `limit` | Maker quote-mint premium commitment limit |
| `maker_quote_liability_cap_exceeded` | `quote_mint`, `current`, `limit` | Maker spread liability limit for the quote mint |
| `caps_unavailable` | — | Caps authority temporarily unavailable (venue state, not your account). Retry with backoff |

## Monitoring caps

`GetMyCaps` is cheap enough to poll; 60 seconds is a reasonable default. `utilization` on platform caps is `current / max` and is **not clamped**: values above `1.0` are a real over-budget state (limits can be tightened below live usage). A zero platform limit reports saturated utilization `1.0`, not spare capacity. Many makers stop submitting new quotes once utilization exceeds `0.9`, leaving headroom for in-flight quotes.

`MakerCapsUpdated` is pushed to the maker's session without a subscription when the caps projection changes for that maker. It carries `caps_snapshot` (the same `positions`, `notional`, `balances` and `quote_liabilities` as `MyCaps`) and `reason`: `lifecycle` when a settled or liquidated position released capacity, `policy` when an administrator changed a limit that applies to the maker.

```json
{
  "type": "MakerCapsUpdated",
  "data": {
    "caps_snapshot": {
      "positions": { "current": 11, "limit": 50 },
      "notional": [],
      "balances": [],
      "quote_liabilities": []
    },
    "reason": "lifecycle"
  }
}
```

Fills do not push it. Poll `GetMyCaps` to track capacity between pushes.

`caps_unavailable` rejections mean the venue's risk projection is catching up (typically after a fill, for well under a second, or during listener incidents). They are retryable venue state: back off briefly and requote. They do not mean your balance or limits changed.

## Freeing capacity

Reserved capacity returns when a quote is cancelled, replaced, expires, or is rejected. Position exposure returns when the position settles. Deposited premium not backing an open position or live quote can be retrieved via the on-chain `WithdrawPremium` instruction.

## Reference

- Wire-level request/response framing for `GetTokenCaps` / `GetMyCaps` (envelope shape, `request_id` correlation): [maker-api.md](maker-api.md).
- `RfqSkipped` broadcast (sent in place of `RfqBroadcast` when a cap blocks delivery): [maker-api.md](maker-api.md#rfqskipped-server---maker).
