# Acta HTTP API
> Trading is WebSocket-based. See the [Maker API reference](maker-api.md) and [Taker API reference](taker-api.md).

## Base URL

```
https://devnet-api.acta.markets
https://beta-api.acta.markets
```

## Response and units

- JSON: `application/json`
- Timestamps: Unix seconds
- `price` / `strike`: `u64` in 1e9 scale (premium and strike conventions match WS docs)
- `quantity`: underlying atomic units
- Pubkeys/addresses: base58 strings

## Error format

```json
{ "error": "Human-readable message", "code": "error_code" }
```

Common codes:
- `invalid_id` (400)
- `invalid_cursor` (400)
- `not_found` (404)
- `db_disabled` (503) — server runs without a database; permanent, do not retry
- `db_busy` (503) — database pool exhausted or query timed out; retry with backoff
- `server_busy` (503) — global in-flight limit reached; retry with backoff
- `rate_limited` (429) — per-IP rate limit; retry with backoff
- `data_validation_error` (500)
- `schema_error` (500)
- `db_error` (500)

Temporary unavailability may return HTTP `503`.

---

## Health

### GET `/health`

Returns service health and metadata.

### GET `/ready`

Returns readiness status: `200` when every gated component is healthy, `503`
otherwise. The required components depend on the server configuration:

<!-- generated:rfq-ready -->
| Configuration | `/ready` gates on |
|---|---|
| database + DB feed ids + caps (production) | redis, kernel, maker_registry, market_metadata, postgres, feed_mapping, price_stream, caps |
| database + caps, static feed ids | redis, kernel, maker_registry, market_metadata, postgres, caps |
| database only | redis, kernel, maker_registry, market_metadata, postgres |
| no database (local demo) | redis, kernel, maker_registry, market_metadata |

`kernel` is also the `/live` gate: a closed kernel channel needs a restart. During shutdown drain, `/ready` returns 503 while `/live` remains governed by the kernel.
<!-- /generated:rfq-ready -->

### GET `/live`

Returns process liveness.

### GET `/metrics`

Prometheus metrics endpoint.

---

## Reference data

`/venue`, `/tokens` and `/markets` are served from one in-memory snapshot that also answers the WS
discovery requests (`GetTokens`, `GetMarkets`, `GetMarketDescriptors`, `GetExpiries` and
`Snapshot.markets`), so HTTP and WS never disagree. They need no database and share the light per-IP
rate limit (30 req/s, burst 60).

The server rebuilds the snapshot when the market registry changes, when a market reaches its trading
cutoff or expiry, and at least every 30 s. When the token list or the market list changes, every
authenticated maker, taker and vault operator WS session receives
[`ReferenceChanged`](ws-common.md#referencechanged) with the new ETags.

### Caching

Every response carries a strong `ETag` and `Cache-Control`:

| Route | `Cache-Control` |
|---|---|
| `/venue`, `/tokens` | `public, max-age=60` |
| `/markets` | `public, max-age=5` |

Send the last `ETag` back in `If-None-Match`; an unchanged body returns `304 Not Modified` with no
body and the same `ETag` and `Cache-Control`. `If-None-Match` accepts a comma-separated list, `*`
and the `W/` prefix. Polling with `If-None-Match` is cheap, but `ReferenceChanged` tells you when to
re-read.

`ReferenceChanged.tokens_etag` equals the `/tokens` ETag. `ReferenceChanged.markets_etag` is the ETag of
the unfiltered `GET /api/v1/markets?active=false` body, not of the default active list or of a
filtered query, so compare it only with that request.

### GET `/api/v1/venue`

Program id and the limits the server validates against. `null` means no limit. The values come from
server configuration and change only on restart.

```json
{
  "program_id": "ProgramIdBase58",
  "min_quote_validity_seconds": 100,
  "settlement_buffer_seconds": 90,
  "quote_refresh_lead_seconds": 10,
  "min_rfq_timeout_seconds": 5,
  "max_rfq_timeout_seconds": 120,
  "signature_timeout_seconds": 30,
  "max_quotes_per_rfq": 50,
  "max_active_rfqs_per_taker": 50,
  "pre_expiry_cutoff_seconds": 10800
}
```

| Field | Meaning |
|---|---|
| `min_quote_validity_seconds` | Minimum `valid_until - now` accepted on a quote (`settlement_buffer_seconds + quote_refresh_lead_seconds`) |
| `settlement_buffer_seconds` | A quote stops being acceptable at `valid_until - settlement_buffer_seconds` |
| `quote_refresh_lead_seconds` | How long before that cutoff the server sends `QuoteRefreshRequested` |
| `min_rfq_timeout_seconds` / `max_rfq_timeout_seconds` | Accepted range for `RfqRequest.timeout_seconds` |
| `signature_timeout_seconds` | Time a taker has to return a signed sponsored transaction |
| `max_quotes_per_rfq` | Quotes kept per RFQ, or `null` |
| `max_active_rfqs_per_taker` | Concurrent open RFQs per taker, or `null` |
| `pre_expiry_cutoff_seconds` | Default no-trade window before market expiry; a market may override it |

The example shows the server defaults; deployments may differ, so read the live endpoint.

`500` `invalid_program_id` when the server's program id is misconfigured.

### GET `/api/v1/tokens`

Every token of the active markets, once, sorted by mint. Markets reference tokens by mint; join on
`mint` for symbol and decimals.

```json
{
  "tokens": [
    {
      "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "symbol": "USDC",
      "decimals": 6,
      "oracle_feed_ids": ["eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a"]
    },
    {
      "mint": "So11111111111111111111111111111111111111112",
      "symbol": "SOL",
      "decimals": 9,
      "oracle_feed_ids": ["ef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d"],
      "size_rule": { "min_size": 100000000, "max_size": 1000000000000, "step": 10000000 },
      "strike_tick": 500000000,
      "quote_mints": ["EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"]
    }
  ]
}
```

`TokenInfo` is the same object on WS; field meanings are in
[WS common conventions](ws-common.md#tokeninfo). `size_rule`, `strike_tick` and `quote_mints` are
present only on underlyings; `oracle_feed_ids` is omitted when no feed is known.

`503` when token metadata is inconsistent (for example the same mint with two decimal values) or an
underlying has no configured size rule (`code: "missing_size_rule_for_underlying_mint"`, as on WS).
The body is the usual error object; retry later.

### GET `/api/v1/markets`

Query (all optional):

| Parameter | Type | Meaning |
|---|---|---|
| `underlying` | mint (base58) | Underlying mint |
| `quote` | mint (base58) | Quote mint |
| `expiry` | u64 | Exact `expiry_ts`, Unix seconds |
| `is_put` | bool | `true` for puts, `false` for calls |
| `active` | bool | Default `true`: tradable markets only. `false`: every market the server holds in memory |

`underlying` and `quote` are mints, not symbols. Markets are sorted by `expiry_ts`, then PDA.

```json
{
  "markets": [
    {
      "pda": "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
      "underlying": "So11111111111111111111111111111111111111112",
      "quote": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 1767225600,
      "is_put": true,
      "status": "finalized",
      "settlement_price": 150000000000
    },
    {
      "pda": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
      "underlying": "So11111111111111111111111111111111111111112",
      "quote": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 4102444800,
      "is_put": false,
      "status": "active"
    }
  ]
}
```

| `status` | Meaning |
|---|---|
| `active` | Tradable: not finalized, not disabled, before the trading cutoff |
| `closed` | Disabled, past its trading cutoff, or expired and not yet finalized |
| `finalized` | Settlement price published; `settlement_price` is set (1e9 scale) |

`active=false` is not a full history: the server holds finalized markets only from the time it
started, so a market finalized before the last restart is missing. Use your own records or the
chain for older markets.

There is no single-market route; `GET /api/v1/markets/{pda}` was removed. Filter the list, or use
`GetMarketDescriptors` on WS for the full descriptor (oracle PDAs, decimals, collateral and
settlement mints).

---

## Makers

### GET `/api/v1/makers`

```json
{
  "makers": [
    {
      "pda": "MakerPdaBase58",
      "owner": "MakerOwnerBase58",
      "quote_signing": "MakerQuoteSigningBase58",
      "total_trades": 12,
      "total_volume": 1000000,
      "total_volume_exact": "1000000",
      "total_quotes": 55
    }
  ]
}
```

`total_volume_exact` is authoritative and always present. `total_volume` is the same figure as a
JSON number and is `null` when the total does not fit in a `u64`.

### GET `/api/v1/makers/{pda}`

Get a single maker DTO.

---

## Vaults

Read-only projections of on-chain vault state. Every numeric field that can exceed `2^53` is a
decimal string.

`/vaults` and `/vaults/{pda}` are served from memory, work without a database and share the light
per-IP rate limit (30 req/s, burst 60). All other vault and depositor routes query the database
per request and share the heavy per-IP limit (10 req/s, burst 20). IPv6 clients are grouped by /64.
Successful `/vaults`, `/vaults/{pda}`, `/vaults/{pda}/cycles`, `/vaults/{pda}/positions` and the
request-list routes carry `Cache-Control: public, max-age=5`; per-wallet routes carry no cache header.

Every per-vault route returns `404` `not_found` for an unknown vault or a PDA that is not base58.
`GET /vaults` and `GET /vaults/{pda}` list open vaults only; a closed vault returns `404` there, while
its cycles, positions and depositor routes stay available.

### GET `/api/v1/vaults`

```json
{
  "vaults": [
    {
      "vault_pda": "VaultPdaBase58",
      "vault_id": "<64 hex chars>",
      "governance": "GovernanceBase58",
      "fee_recipient": "FeeRecipientBase58",
      "primary_delegate": "DelegateBase58",
      "main_mint": "MainMintBase58",
      "second_mint": "SecondMintBase58",
      "share_mint": "ShareMintBase58",
      "main_token_program": "legacy_spl",
      "second_token_program": "legacy_spl",
      "main_decimals": 6,
      "second_decimals": 9,
      "phase": "active",
      "safety": "normal",
      "capabilities": 3,
      "active_expiry": "1710000000",
      "open_positions": "2",
      "total_shares": "1000000000",
      "share_price": "1000000000",
      "equity": "1000000000",
      "capital_generation": "4",
      "accepting_deposits": true,
      "equity_computed": true,
      "pending_deposit_count": 0,
      "pending_withdraw_count": 0,
      "pending_deposit_amount": "0",
      "pending_withdraw_shares": "0",
      "performance_fee_bps": 1000,
      "management_fee_bps": 100,
      "manager_stake_min_bps": 500,
      "share_price_hwm": "1000000000",
      "last_hwm_update_ts": "1710000000",
      "last_hwm_reset_ts": "0",
      "last_settlement_ts": "1710000000",
      "cap_limit": "0",
      "capital_fallback_delay_secs": "43200",
      "deposit_window_set_ts": "1709990000",
      "main_balance": "1000000",
      "second_balance": "0",
      "balance_updated_at": "1710000000",
      "live_share_supply": "1000000000",
      "observed_manager_stake_shares": "50000000",
      "balance_chain_slot": "312000000",
      "policy": {
        "max_open_positions": 8,
        "max_rebalance_drift_bps": 200,
        "min_main_liquidity_bps": 1000
      },
      "projection": {
        "version": "42",
        "slot": "312000000",
        "transaction_index": 3,
        "signature": "TxSignatureBase58",
        "event_index": 0
      },
      "performance": {
        "status": "annualized",
        "cycles_completed": 12,
        "last_cycle_return_pct": 0.8,
        "cumulative_return_pct": 12.4,
        "annualized_return_pct": 23.4,
        "track_record_secs": "7257600"
      }
    }
  ]
}
```

| Field | Values / meaning |
|---|---|
| `phase` | `idle`, `active`, `settling`, `processing_capital` |
| `safety` | `normal`, `frozen`, `emergency_unwind`, `withdraw_only`, `closing` |
| `main_token_program` / `second_token_program` | `legacy_spl`, `token_2022` |
| `total_shares` | Boundary checkpoint adjusted for contract mint/burn — **not** live SPL supply. Use `live_share_supply` for the observed supply. |
| `main_balance`, `second_balance`, `balance_updated_at`, `live_share_supply`, `observed_manager_stake_shares`, `balance_chain_slot` | All `null` together when no balance observation has landed yet. |
| `capital_fallback_delay_secs` | Delay (default `43200`, max `259200`) after which anyone, not only the operator, may process a queued request. In `idle` it counts from the request's `created_at`; in `processing_capital` it counts from `active_expiry`. In `idle`, deposits also need priced equity, which a non-operator can only compute with an aged withdraw request; otherwise cancel the deposit. |
| `deposit_window_set_ts` | Chain time of the last `SetAcceptingDeposits`, or `null` if the window was never set. |
| `projection` | Position of this row in the listener's event stream. Compare `slot` / `transaction_index` / `event_index` to detect staleness. |
| `performance` | Tagged by `status`. `unavailable`: the server has not loaded performance yet; show "—", not "new vault". `no_track_record`: no finished cycle yet, no other fields. `short_track_record`: `cycles_completed`, `last_cycle_return_pct`, `cumulative_return_pct`, `track_record_secs`. `annualized`: the same plus `annualized_return_pct`, once `track_record_secs` is at least 30 days. `track_record_secs` is the calendar span from the start of the first finished cycle to the end of the last one, idle gaps included. Returns are percentages (`1.0` = 1%), after fees, compounded from finished cycles. Refreshed in the background every 15 s. |

### GET `/api/v1/vaults/{pda}`

Returns a single vault object (the element shape above), or `404` `not_found` for an unknown, closed or
malformed PDA.

### GET `/api/v1/vaults/{pda}/positions`

Query:

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `settled` | bool | unset | Filters on `settled_at`: `true` returns only rows that have one, `false` only rows that do not; unset returns both |
| `limit` | u16 | `100` | Page size, `1..=200`. Out-of-range values return `400` `invalid_limit`. |
| `cursor` | string | unset | `next_cursor` from the previous page |

```json
{
  "positions": [
    {
      "vault": "VaultPdaBase58",
      "position": "PositionPdaBase58",
      "order_id": "<64 hex chars>",
      "role": "writer",
      "market": "MarketPdaBase58",
      "position_type": "covered_call",
      "strike": "1000000000",
      "quantity": "1000000",
      "net_premium": "25000000",
      "counterparty": "CounterpartyBase58",
      "funded_amount": null,
      "status": "open",
      "is_otm": null,
      "created_at": "1710000000",
      "acquired_at": "1710000000",
      "settled_at": null,
      "transaction_signature": "TxSignatureBase58"
    }
  ],
  "next_cursor": "312000000:3:0:PositionPdaBase58"
}
```

| Field | Values / meaning |
|---|---|
| `role` | `writer` (the vault posted collateral) or `holder` (the vault bought the option) |
| `position_type` | `covered_call`, `cash_secured_put` |
| `status` | `open`, `funded`, `liquidated`, `settled` |
| `is_otm` | `null` until the market is finalized |
| `next_cursor` | `slot:transaction_index:event_index:position`. Opaque — pass it back verbatim. `null` on the last page. |

A malformed `cursor`, including the legacy position cursor format, returns `400` with code `invalid_cursor`.

### GET `/api/v1/vaults/{pda}/cycles`

Query: `limit` (u16, default `50`, `1..=200`) and `cursor` (`next_cursor` from the previous page).
Cycles are returned newest first.

```json
{
  "cycles": [
    {
      "cycle_number": "3",
      "capital_generation": "3",
      "expiry": "1710604800",
      "started_at": "1710000000",
      "finished_at": "1710604900",
      "share_price_start": "1000000000",
      "share_price_end": "1010000000",
      "equity_start": "1000000",
      "equity_end": "1010000",
      "positions_opened": 4,
      "positions_settled": 4,
      "return_pct": 1.0,
      "performance_fee_assessed": "1000",
      "management_fee_assessed": "190"
    }
  ],
  "next_cursor": "2"
}
```

A cycle is finished only when `finished_at` is set. The running cycle has `finished_at: null`, but
`share_price_end`, `equity_end` and `return_pct` may already be set at the equity boundary, before
finalize.
`return_pct` is a display value; compute exact returns from the share prices with BigInt.

### GET `/api/v1/vaults/{pda}/deposit-requests` and `/withdraw-requests`

Pending capital requests, for curators batching `ProcessDeposits` / `ProcessWithdrawals`.
Query: `limit` (u16, default `50`, `1..=100`) and `cursor` (`next_cursor` from the previous page).
Requests are ordered by wallet; `next_cursor` is `null` on the last page. Withdraw requests carry
`shares` instead of `amount`. `transaction_signature` may be null.

```json
{
  "requests": [
    {
      "wallet": "WalletBase58",
      "amount": "1000000",
      "created_at": "1710000000",
      "transaction_signature": "TxSignatureBase58"
    }
  ],
  "next_cursor": null
}
```

| Status | Code | When |
|---|---|---|
| `400` | `invalid_limit` | `limit` is 0 or above 100 |
| `400` | `invalid_cursor` | `cursor` is not a wallet address |
| `404` | `not_found` | the vault is unknown |
| `429` | `rate_limited` | per-IP limit; retry with backoff |
| `503` | `db_busy`, `server_busy` | transient; retry with backoff |
| `503` | `db_disabled` | the server runs without a database; permanent, do not retry |

### GET `/api/v1/vaults/{pda}/depositors/{wallet}`

The wallet's pending requests and lifetime totals. A wallet that never used the vault gets `200`
with `deposit_request`, `withdraw_request` and `totals` all `null`. A request's `created_at` is chain
block time. `totals.total_withdrawn` counts the main leg only; second-leg payouts (final redemption)
appear per event as `second_amount` in the history.

```json
{
  "vault": "VaultPdaBase58",
  "wallet": "WalletBase58",
  "deposit_request": { "amount": "1000000", "created_at": "1710000000", "transaction_signature": "TxSignatureBase58" },
  "withdraw_request": null,
  "totals": {
    "shares_minted": "1000000000",
    "shares_burned": "0",
    "total_deposited": "1000000",
    "total_withdrawn": "0",
    "last_activity_at": "1710000000"
  }
}
```

### GET `/api/v1/vaults/{pda}/depositors/{wallet}/history`

Query: `limit` (u16, default `50`, `1..=200`) and `cursor`. Events are returned newest first.

```json
{
  "events": [
    {
      "vault": "VaultPdaBase58",
      "action": "deposit_processed",
      "amount": "1000000",
      "second_amount": "0",
      "shares": "1000000000",
      "boundary_price": "1000000000",
      "event_time": "1710000000",
      "transaction_signature": "TxSignatureBase58"
    }
  ],
  "next_cursor": "312000000:3:0:42"
}
```

| Field | Values / meaning |
|---|---|
| `action` | `deposit_processed`, `withdraw_processed`, `deposit_cancelled`, `withdraw_cancelled`, `deposit_refunded` |
| `boundary_price` | Share price the request was processed at; `null` for cancellations and refunds |
| `next_cursor` | Opaque — pass it back verbatim. `null` on the last page. |

### GET `/api/v1/depositors/{wallet}`

The wallet's portfolio: one entry per vault the wallet has a request or history in. Each entry has
the same shape as `/vaults/{pda}/depositors/{wallet}`.

```json
{ "wallet": "WalletBase58", "vaults": [ { "vault": "VaultPdaBase58", "wallet": "WalletBase58", "deposit_request": null, "withdraw_request": null, "totals": null } ] }
```

### GET `/api/v1/depositors/{wallet}/history`

The same event feed as `/vaults/{pda}/depositors/{wallet}/history`, across all vaults. It takes the
same `limit` and `cursor` and returns the same page shape.

Errors for the positions, cycle and depositor endpoints:

| Status | Code | When |
|---|---|---|
| `400` | `invalid_limit` | `limit` is 0 or above 200 |
| `400` | `invalid_cursor` | `cursor` is malformed |
| `400` | `invalid_wallet` | `wallet` is not a base58 address |
| `404` | `not_found` | the vault is unknown |
| `429` | `rate_limited` | per-IP limit; retry with backoff |
| `503` | `db_busy`, `server_busy` | transient; retry with backoff |
| `503` | `db_disabled` | the server runs without a database; permanent, do not retry |

---

## Participant history

RFQ, quote, and order history is not exposed by the public HTTP API. Use the
authenticated maker/taker session protocols for participant-specific state.

---

## Stats

### GET `/api/v1/stats`

```json
{
  "usd": { "notional_24h": "1500.00", "premium_24h": "49.00", "priced_trades_24h": 149 },
  "total_volume_24h": 1000000,
  "total_volume_24h_exact": "1000000",
  "total_trades_24h": 150,
  "active_markets": 12,
  "active_makers": 5,
  "connected_makers": 3
}
```

This differs from WS `GlobalStats` in `Snapshot` and `StatsUpdate`. HTTP includes `connected_makers` (live WS sessions) and omits `total_price_24h` and `active_rfqs`.

`usd` separates underlying notional from gross paid premium, both as exact decimal
dollar strings. Prices are captured from the trusted local oracle cache when the
signed transaction is submitted; only confirmed trades are counted. USDC uses its
oracle price too. `priced_trades_24h < total_trades_24h` means some confirmed trades
lack a stored USD valuation; those are not repriced using today's market.
The legacy `total_volume_24h` fields are raw aggregates, not dollars.

---

## Related

- [WS common conventions](ws-common.md)
- [Maker API reference](maker-api.md)
- [Taker API reference](taker-api.md)
