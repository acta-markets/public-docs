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

A query string that cannot be parsed, for example `?active=maybe`, gets a plain-text `400` without this body.

Codes (HTTP status):
- `invalid_request`, `invalid_cursor`, `invalid_limit`, `invalid_delegate`, `invalid_wallet` (400)
- `not_found` (404)
- `rate_limited` (429) — per-IP rate limit; retry with backoff
- `db_disabled` (503) — server runs without a database; permanent, do not retry
- `db_busy` (503) — database pool exhausted or query timed out; retry with backoff
- `server_busy` (503) — global in-flight limit reached; retry with backoff
- `unavailable`, `caps_projection_unavailable`, `chain_replay_unavailable`, `position_history_unavailable`, `venue_unavailable` (503) — the backing projection or program id is not available yet; retry with backoff
- `token_metadata_incomplete`, `missing_size_rule_for_underlying_mint` (503) — `/tokens` while an active market lacks decimals or a size rule
- `history_projection_incomplete`, `caps_projection_incomplete`, `data_validation_error`, `schema_error`, `db_error`, `encode_failed`, `invalid_stats` (500)

WebSocket codes are a separate set; see the [WS error code catalog](ws-common.md#ws-error-codes).

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
| database + DB feed ids (production) | redis, kernel, maker_registry, market_metadata, caps, postgres, feed_mapping, price_stream |
| database, static feed ids | redis, kernel, maker_registry, market_metadata, caps, postgres |
| no database (local demo) | redis, kernel, maker_registry, market_metadata, caps |

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
| `/venue` | `public, no-cache` |
| `/tokens` | `public, max-age=60` |
| `/markets` | `public, max-age=5` |

Send the last `ETag` back in `If-None-Match`; an unchanged body returns `304 Not Modified` with no
body and the same `ETag` and `Cache-Control`. `If-None-Match` accepts a comma-separated list, `*`
and the `W/` prefix. Polling with `If-None-Match` is cheap, but `ReferenceChanged` tells you when to
re-read.

`ReferenceChanged.tokens_etag` equals the `/tokens` ETag. `ReferenceChanged.markets_etag` is the ETag of
the unfiltered `GET /api/v1/markets?active=false` body, not of the default active list or of a
filtered query, so compare it only with that request.

### GET `/api/v1/venue`

Program id, protocol fees and the limits the server validates against. The values come from server
configuration and change only on restart. `program_id` is authoritative; devnet and mainnet use the
same program id.

```json
{
  "protocol_fee_bps_premium": 100,
  "protocol_fee_bps_volume": 10,
  "program_id": "ProgramIdBase58",
  "min_quote_validity_seconds": 60,
  "settlement_buffer_seconds": 50,
  "quote_refresh_lead_seconds": 10,
  "min_rfq_timeout_seconds": 5,
  "max_rfq_timeout_seconds": 120,
  "signature_timeout_seconds": 30,
  "max_quotes_per_rfq": { "type": "limited", "value": 50 },
  "max_active_rfqs_per_taker": { "type": "unlimited" },
  "pre_expiry_cutoff_seconds": 10800
}
```

| Field | Meaning |
|---|---|
| `protocol_fee_bps_premium` | Premium fee in basis points of the premium. See the [fee model](ws-common.md#fee-model) |
| `protocol_fee_bps_volume` | Volume fee in basis points of the exposure: strike notional for physical products, width notional for spreads and cash puts. The charged fee is the smaller of the two fees |
| `program_id` | The Acta program id |
| `min_quote_validity_seconds` | Minimum `valid_until - now` accepted on a quote; at least `settlement_buffer_seconds + quote_refresh_lead_seconds`. Use this value, not the sum |
| `settlement_buffer_seconds` | A quote stops being acceptable at `valid_until - settlement_buffer_seconds` |
| `quote_refresh_lead_seconds` | How long before that cutoff the server sends `QuoteRefreshRequested` |
| `min_rfq_timeout_seconds` / `max_rfq_timeout_seconds` | Accepted range for `RfqRequest.timeout_seconds` |
| `signature_timeout_seconds` | Time a taker has to return a signed sponsored transaction |
| `max_quotes_per_rfq` | `CountLimit`: quotes kept per RFQ |
| `max_active_rfqs_per_taker` | `CountLimit`: concurrent open RFQs per taker |
| `pre_expiry_cutoff_seconds` | Default no-trade window before market expiry; a market may override it |

`CountLimit` is a tagged union: `{ "type": "unlimited" }` or `{ "type": "limited", "value": N }`.

The example shows one configuration; read the live endpoint for the values.

`503` `venue_unavailable` when the program id or protocol fees are not available.

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
      "oracle_feed_ids": ["eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a"],
      "role": { "type": "quote" }
    },
    {
      "mint": "So11111111111111111111111111111111111111112",
      "symbol": "SOL",
      "decimals": 9,
      "oracle_feed_ids": ["ef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d"],
      "role": {
        "type": "underlying",
        "size_rule": { "min_size": 100000000, "max_size": 1000000000000, "step": 10000000 },
        "strike_tick": 500000000,
        "quote_mints": ["EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"]
      }
    }
  ]
}
```

`TokenInfo` is the same object on WS; field meanings are in
[WS common conventions](ws-common.md#tokeninfo). `role` is `quote` or `underlying`; `size_rule`,
`strike_tick` and `quote_mints` exist only inside the `underlying` role. `strike_tick` is 1e9
scaled like strikes (`500000000` is 0.5 quote units). `oracle_feed_ids` is `[]` when no feed is
known.

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
      "settlement_price": 150000000000,
      "products": ["cash_secured_put", "put_spread", "cash_put"]
    },
    {
      "pda": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
      "underlying": "So11111111111111111111111111111111111111112",
      "quote": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 4102444800,
      "is_put": false,
      "status": "active",
      "settlement_price": null,
      "products": ["covered_call", "call_spread"]
    }
  ]
}
```

`products` lists the products enabled for the pair that fit the market's side: a subset of
`cash_secured_put`, `put_spread` and `cash_put` on a put market, a subset of `covered_call` and
`call_spread` on a call market. A market with none is not listed. `settlement_price` is `null`
until the market is settled.

| `status` | Meaning |
|---|---|
| `active` | Tradable: not finalized, not disabled, before the trading cutoff |
| `closed` | Disabled, past its trading cutoff, or expired and not yet finalized |
| `finalized` | Settlement price published; `settlement_price` is set (1e9 scale) |

`active=false` is not a full history: the server holds finalized markets only from the time it
started, so a market finalized before the last restart is missing. Use your own records or the
chain for older markets.

There is no single-market route. Filter the list, or use
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
      "total_volume": "1000000",
      "total_quotes": 55
    }
  ]
}
```

`total_volume` is a decimal string.

### GET `/api/v1/makers/{pda}`

Get a single maker DTO.

---

## Vaults

Read-only projections of on-chain vault state. Every numeric field that can exceed `2^53` is a
decimal string.

`/vaults` and `/vaults/{pda}` are served from memory, work without a database and share the light
per-IP rate limit (30 req/s, burst 60). All other vault and depositor routes query the database
per request and share the heavy per-IP limit (10 req/s, burst 20). IPv6 clients are grouped by /64.
Successful `/vaults`, `/vaults/{pda}`, `/vaults/{pda}/cycles`, `/vaults/{pda}/positions`,
`/vaults/{pda}/deposit-requests` and `/vaults/{pda}/withdraw-requests` carry `Cache-Control: public, max-age=5`; `/vaults/{pda}/exposure` is `no-store`; per-wallet routes carry no cache header.

Every per-vault route returns `404` `not_found` for an unknown vault or a PDA that is not base58.
`GET /vaults` and `GET /vaults/{pda}` list open vaults only; a closed vault returns `404` there, while
its cycles, positions and depositor routes stay available.

`/maker` quote sessions and taker WebSocket sessions subscribed to the `vaults` channel receive
[`VaultCatalogChanged`](ws-common.md#vaultcatalogchanged) when a vault is created, re-delegated or
closed (not when a delegate is disabled), and
[`VaultProjectionAdvanced`](ws-common.md#vaultprojectionadvanced) when a vault's state changes.
Both are hints that are never replayed: re-read `/vaults` (or `/vaults/{pda}`) on them and after
every WebSocket connect or reconnect.

### GET `/api/v1/vaults`

Optional query: `delegate=<pubkey>` returns only vaults that key can operate as delegate: it is the
`primary_delegate` and holds an unexpired delegate grant. A value that is not a base58 pubkey returns `400` `invalid_delegate`.

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
      "second_leg_generation": "0",
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
      "capacity": { "type": "limited", "cap_limit": "1000000000", "remaining": "800000000" },
      "capital_fallback_delay_secs": "43200",
      "deposit_window_set_ts": "1709990000",
      "main_balance": "1000000",
      "second_balance": "0",
      "balance_updated_at": "1710000000",
      "live_share_supply": "1000000000",
      "observed_manager_stake_shares": "50000000",
      "balance_chain_slot": "312000000",
      "staging": null,
      "staged_generation": "0",
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
      "created_at": "1700000000",
      "generation_started_at": "1700000100",
      "tvl_usd_1e9": "1000000000000",
      "price_time": "1710000050",
      "premium": [
        {
          "quote_mint": "QuoteMintBase58",
          "cycle": { "received": "5000000", "paid": "0" },
          "generation": { "received": "50000000", "paid": "7000000" }
        }
      ],
      "fees": {
        "management_fee_assessed": "3000",
        "performance_fee_assessed": "4000",
        "realized_fee_value": "6000"
      },
      "depositors_in_generation": 11,
      "performance": {
        "capital_generation": "4",
        "computed_at": "1710000060",
        "covered": true,
        "returns": {
          "7d": { "return_pct": 0.4, "from_time": "1709395200", "to_time": "1710000000" },
          "30d": { "return_pct": 1.5, "from_time": "1707408000", "to_time": "1710000000" },
          "90d": null,
          "all": { "return_pct": 2.0, "from_time": "1700000100", "to_time": "1710000000" }
        },
        "apy_30d": 19.9,
        "apy_90d": null,
        "max_drawdown_pct": 0.5,
        "worst_cycle_return_pct": -0.25,
        "losing_cycles": 1,
        "cycles_completed": 12
      },
      "metadata": {
        "name": "SOL vault",
        "curator": "Acta",
        "strategy": "Weekly covered calls on SOL",
        "description": null,
        "website": "https://acta.markets",
        "twitter": "acta_markets",
        "logo_url": null
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
| `staging` | `null` unless half of a [two-transaction swap](../quickstart/vault-operator-guide.md#two-transaction-swaps) is open: `{ "mint", "from_leg": "main" \| "second", "amount", "staged_at", "generation" }`. `amount` (atoms of `mint`, decimal string) has left `from_leg` for the vault's account of `mint` and is not counted in `equity`. `staged_at` (Unix seconds) is the first stage. Capital processing waits until it is back. |
| `deposit_window_set_ts` | Chain time of the last `SetAcceptingDeposits`, or `null` if the window was never set. |
| `projection` | Position of this row in the venue's on-chain event history. Compare `slot` / `transaction_index` / `event_index` to detect staleness. |
| `second_leg_generation`, `staged_generation` | Generation counters, decimal strings. |
| `capacity` | Tagged union. `{ "type": "unlimited" }` when the on-chain cap is `0`. Otherwise `{ "type": "limited", "cap_limit", "remaining" }` in main atoms, with `remaining = max(0, cap_limit − equity − pending_deposit_amount)`. |
| `created_at` | Chain time of the vault's creation; `null` until the stats snapshot has loaded. |
| `tvl_usd_1e9`, `price_time` | Indicative: `equity` at the main mint's USD oracle price (1e9 scale) and that price's time; both `null` without a fresh price. |
| `generation_started_at`, `premium`, `fees`, `depositors_in_generation` | From one stats snapshot, `null` unless its generation equals `capital_generation` and the generation is covered (see `performance`). `premium`: per quote mint, quote atoms received as writer and paid as holder (gross premium) in the current cycle and the generation. `fees`: assessed management and performance fees and realized fee value of the generation, in main atoms. `depositors_in_generation`: wallets with a processed deposit in the generation. |
| `performance` | `null` until stats load or when the snapshot is from another generation. `capital_generation` and `computed_at` label the snapshot (refreshed every 60 s). `covered: false` means the generation's history is incomplete: every metric is `null`. `returns` per window (`7d`, `30d`, `90d`, `all`): booked share price return after fees, in percent (`1.0` = 1%), with the checkpoint times it spans; `null` when the generation is younger than the window. `apy_30d`, `apy_90d`: the 30- and 90-day returns annualized by compounding (APY). `max_drawdown_pct`: largest peak-to-trough fall of the booked share price. `worst_cycle_return_pct`, `losing_cycles`, `cycles_completed`: finished cycles of the generation. |
| `metadata` | `null` when the operator set none. Otherwise an object with `name` and the nullable fields `curator`, `strategy`, `description`, `website` and `logo_url` (`https://` URLs) and `twitter` (a bare handle, no `@`); all seven keys are present. Edits appear within about a minute. Display only; `governance` is the authority, so show it next to the name. |

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
      "product": { "kind": "physical" },
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
| `position_type` | `covered_call`, `cash_secured_put`, `call_spread`, `put_spread`, `cash_put` |
| `strike` | Strike in 1e9 scale. The lower strike for `call_spread` and `put_spread`; `K` for a cash put, equal to its `upper_strike` |
| `product` | `{ "kind": "physical" }` for covered calls and cash-secured puts. `{ "kind": "spread", "upper_strike", "collateral" }` for spreads and cash puts: the upper strike and the quote atoms the position reserves |
| `net_premium` | Net premium in quote atoms after the protocol fee; negative for a `holder` row |
| `status` | `open`, `funded`, `liquidated`, `settled` |
| `is_otm` | `null` until the market is finalized |
| `next_cursor` | `slot:transaction_index:event_index:position`. Opaque — pass it back verbatim. `null` on the last page. |

A malformed `cursor` returns `400` with code `invalid_cursor`.

### GET `/api/v1/vaults/{pda}/exposure`

Open and funded positions with an indicative spot. Positions are reloaded after 10 s and never served older than 15 s (`as_of` is when they were read; a failed reload past that returns `503`); `spot` is read per request. Needs the database (`503` `db_disabled` without one).

```json
{
  "vault_pda": "VaultPdaBase58",
  "positions": [
    {
      "position": "PositionPdaBase58",
      "market": "MarketPdaBase58",
      "position_type": "covered_call",
      "role": "writer",
      "status": "open",
      "strike": "140000000000",
      "quantity": "2000000000",
      "expiry": "1710000000",
      "underlying_mint": "UnderlyingMintBase58",
      "quote_mint": "QuoteMintBase58",
      "underlying_decimals": 9,
      "quote_decimals": 6,
      "spot": "150000000000",
      "spot_time": "1709990000",
      "distance_pct": -6.666666666666667,
      "itm": true,
      "intrinsic": "-20000000"
    }
  ],
  "totals": [
    { "quote_mint": "QuoteMintBase58", "notional_at_strike": "280000000", "outstanding_intrinsic": "-20000000" }
  ]
}
```

| Field | Values / meaning |
|---|---|
| `spot`, `spot_time` | Underlying price in quote, 1e9 scale, from the oracle cache; `null` when a leg is stale or retired |
| `distance_pct` | Call `100 × (strike − spot) / spot`, put `100 × (spot − strike) / spot`; positive is out of the money |
| `itm` | `spot` past the strike; equality is out of the money |
| `intrinsic` | Value moved if the position expired at `spot`, in quote atoms: `0` out of the money, positive for `holder`, negative for `writer`. Not a cash flow or fair value (time value is ignored). |
| `underlying_decimals`, `quote_decimals` | `null` when market metadata is missing; `intrinsic` is then `null` |
| `totals` | Per quote mint: `notional_at_strike` and `outstanding_intrinsic` (signed); `null` when any of its positions lacks the value |

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
  "total_volume_24h": "1000000",
  "total_trades_24h": 150,
  "spread_trades_24h": 20,
  "active_markets": 12,
  "active_makers": 5,
  "connected_makers": 3
}
```

This differs from WS `GlobalStats` in `Snapshot` and `StatsUpdate`. HTTP includes `connected_makers` (live WS sessions) and `spread_trades_24h`, and omits `total_price_24h` and `active_rfqs`. The route needs the database (`503` `db_disabled` without one).

`usd` separates underlying notional from gross paid premium, both as exact decimal
dollar strings. Prices are captured from the trusted local oracle cache when the
signed transaction is submitted; only confirmed trades are counted. USDC uses its
oracle price too. `priced_trades_24h < total_trades_24h` means some confirmed trades
lack a stored USD valuation; those are not repriced using today's market.
`total_volume_24h` is a raw aggregate string, not dollars.

---

## Related

- [WS common conventions](ws-common.md)
- [Maker API reference](maker-api.md)
- [Taker API reference](taker-api.md)
