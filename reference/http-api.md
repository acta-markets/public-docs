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

## Markets

### GET `/api/v1/markets`

List markets.

Query:
- `underlying` (optional symbol string)

`GET /api/v1/markets` returns tradable markets only: not finalized, not disabled, and before the effective trading cutoff. `underlying` filters by symbol before that tradability filter. There is no `active=false` mode; use `/api/v1/markets/{pda}` for a specific market.

Response:

```json
{
  "markets": [
    {
      "pda": "MarketPdaBase58",
      "underlying_mint": "UnderlyingMintBase58",
      "quote_mint": "QuoteMintBase58",
      "expiry_ts": 1710600000,
      "is_put": false,
      "is_finalized": false,
      "settlement_price": null,
      "underlying_symbol": "SOL",
      "quote_symbol": "USDC"
    }
  ]
}
```

`underlying_feed_id_hex` / `quote_feed_id_hex` are not part of current HTTP `MarketDto`.

### GET `/api/v1/markets/{pda}`

Get a single market DTO.

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
