# Acta HTTP API
> Trading is WebSocket-based. See the [Maker API reference](maker-api.md) and [Taker API reference](taker-api.md).

## Base URL

```
https://devnet-api.acta.markets
https://beta-api.acta.markets
```

API routes are under `/api/v1` (`https://devnet-api.acta.markets/api/v1/venue`); `/health`, `/ready`,
`/live` and `/metrics` are at the root. A route without the prefix returns `404`.

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

`/api/v1/venue`, `/api/v1/tokens` and `/api/v1/markets` are served from one in-memory snapshot that also answers the WS
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
| `/api/v1/venue` | `public, no-cache` |
| `/api/v1/tokens` | `public, max-age=60` |
| `/api/v1/markets` | `public, max-age=5` |

Send the last `ETag` back in `If-None-Match`; an unchanged body returns `304 Not Modified` with no
body and the same `ETag` and `Cache-Control`. `If-None-Match` accepts a comma-separated list, `*`
and the `W/` prefix. Polling with `If-None-Match` is cheap, but `ReferenceChanged` tells you when to
re-read.

`ReferenceChanged.tokens_etag` equals the `/tokens` ETag. `ReferenceChanged.markets_etag` is the ETag of
the unfiltered `GET /api/v1/markets?active=false` body, not of the default active list or of a
filtered query, so compare it only with that request.

### GET `/api/v1/venue`

Program id and the limits the server validates against. Protocol fees belong to each quote token; read them from [`/tokens`](#get-apiv1tokens). The values come from server
configuration and change only on restart. `program_id` is authoritative; devnet and mainnet use the
same program id.

```json
{
  "program_id": "ProgramIdBase58",
  "min_quote_validity_seconds": 60,
  "settlement_buffer_seconds": 50,
  "quote_refresh_lead_seconds": 10,
  "min_rfq_timeout_seconds": 5,
  "max_rfq_timeout_seconds": 120,
  "signature_timeout_seconds": 30,
  "max_quotes_per_rfq": { "type": "limited", "value": 50 },
  "max_active_rfqs_per_taker": { "type": "limited", "value": 50 },
  "pre_expiry_cutoff_seconds": 10800,
  "max_page_size": 200,
  "max_batch_quotes": 50
}
```

| Field | Meaning |
|---|---|
| `program_id` | The Acta program id |
| `min_quote_validity_seconds` | Minimum `valid_until - now` accepted on a quote; at least `settlement_buffer_seconds + quote_refresh_lead_seconds`. Use this value, not the sum |
| `settlement_buffer_seconds` | A quote stops being acceptable at `valid_until - settlement_buffer_seconds` |
| `quote_refresh_lead_seconds` | How long before that cutoff the server sends `QuoteRefreshRequested` |
| `min_rfq_timeout_seconds` / `max_rfq_timeout_seconds` | Accepted range for `RfqRequest.timeout_seconds` |
| `signature_timeout_seconds` | Time a taker has to return a signed sponsored transaction |
| `max_quotes_per_rfq` | `CountLimit`: quotes kept per RFQ |
| `max_active_rfqs_per_taker` | `CountLimit`: concurrent open RFQs per taker |
| `pre_expiry_cutoff_seconds` | Default no-trade window before market expiry; a market may override it |
| `max_page_size` | Largest `limit` any paged request accepts, on WS and REST |
| `max_batch_quotes` | Most quotes one `BatchQuotes` message may carry |

`CountLimit` is a tagged union: `{ "type": "unlimited" }` or `{ "type": "limited", "value": N }`.

The example shows one configuration; read the live endpoint for the values.

`503` `venue_unavailable` when the program id is not available.

### GET `/api/v1/tokens`

Uses [common paging](ws-common.md#paging).

Every token of the active markets, once, sorted by mint. Markets reference tokens by mint; join on
`mint` for symbol and decimals.

```json
{
  "tokens": [
    {
      "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "symbol": "USDC",
      "decimals": 6,
      "oracle_feed_ids": [
        "eaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a"
      ],
      "role": {
        "type": "quote"
      },
      "schedule": {
        "type": "continuous"
      },
      "session": {
        "type": "continuous"
      },
      "protocol_fees": {
        "type": "quote",
        "premium_bps": 1250,
        "volume_bps": 50
      }
    },
    {
      "mint": "So11111111111111111111111111111111111111112",
      "symbol": "SOL",
      "decimals": 9,
      "oracle_feed_ids": [
        "ef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d"
      ],
      "role": {
        "type": "underlying",
        "size_rule": {
          "min_size": 100000000,
          "max_size": 1000000000000,
          "step": 10000000
        },
        "strike_tick": 500000000,
        "quote_mints": [
          "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
        ]
      },
      "schedule": {
        "type": "continuous"
      },
      "session": {
        "type": "continuous"
      },
      "protocol_fees": {
        "type": "not_quote_token"
      }
    }
  ],
  "next_cursor": null
}
```

`TokenInfo` is the same object on WS; field meanings are in
[WS common conventions](ws-common.md#tokeninfo). `role` is `quote` or `underlying`; `size_rule`,
`strike_tick` and `quote_mints` exist only inside the `underlying` role. `strike_tick` is 1e9
scaled like strikes (`500000000` is 0.5 quote units). `oracle_feed_ids` is `[]` when no feed is
known.

`schedule` is the token's price feed schedule. `{ "type": "continuous" }` trades around the clock.
A token with trading hours, such as TSLA or XAU, has
`{ "type": "pyth", "spec": "America/New_York;0930-1600,0930-1600,0930-1600,0930-1600,0930-1600,C,C;1225/C" }`:
the Pyth schedule of its feed, `TZ;Mon,..,Sun;MMDD/spec,...`. Each spec is `O` (open all day), `C`
(closed) or `HHMM-HHMM` windows joined by `&`, in the listed timezone; a holiday entry replaces that
date's weekday entry. `{ "type": "unknown" }` means no schedule is known for the feed yet.

`session` is whether the feed trades now: `{ "type": "continuous" }`, `{ "type": "open", "closes_at" }`,
`{ "type": "closed", "opens_at" }` (Unix seconds), `{ "type": "closed_indefinitely" }` (no session in the
next two weeks) or `{ "type": "unknown" }`. The times are absolute, so a cached response stays
readable: once `closes_at` has passed the market is closed. The server rebuilds the list at each
session change, so the `ETag` changes then. While the market is closed the oracle keeps its last
price, and the server prices RFQs from it.

`protocol_fees` is `{ "type": "quote", "premium_bps", "volume_bps" }` for a token that quotes
markets, from its on-chain config; `{ "type": "not_quote_token" }` for an underlying, and
`{ "type": "unavailable" }` for a quote token whose fee config is not projected yet (not a zero fee). Each quote token has its own
fees. The program charges the smaller of `premium_bps` of the premium and `volume_bps` of the
exposure: the strike notional for physical products and cash puts, the width notional for spreads.
See the [fee model](ws-common.md#fee-model).

`503` when token metadata is inconsistent (for example the same mint with two decimal values) or an
underlying has no configured size rule (`code: "missing_size_rule_for_underlying_mint"`, as on WS).
The body is the usual error object; retry later.

### GET `/api/v1/markets`

Uses [common paging](ws-common.md#paging).

Query (all optional):

| Parameter | Type | Meaning |
|---|---|---|
| `underlying_mint` | mint (base58) | Underlying mint |
| `quote_mint` | mint (base58) | Quote mint |
| `expiry_ts` | u64 | Exact `expiry_ts`, Unix seconds |
| `is_put` | bool | `true` for puts, `false` for calls |
| `active` | bool | Default `true`: tradable markets only. `false`: every market the server holds in memory |

`underlying_mint` and `quote_mint` are mints, not symbols. Markets are sorted by `expiry_ts`, then PDA.

```json
{
  "markets": [
    {
      "pda": "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 1767225600,
      "is_put": true,
      "status": "finalized",
      "settlement": {
        "type": "fixed",
        "price": 150000000000
      },
      "products": [
        "cash_secured_put",
        "put_spread",
        "cash_put"
      ]
    },
    {
      "pda": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
      "underlying_mint": "So11111111111111111111111111111111111111112",
      "quote_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "expiry_ts": 4102444800,
      "is_put": false,
      "status": "active",
      "settlement": {
        "type": "pending"
      },
      "products": [
        "covered_call",
        "call_spread"
      ]
    }
  ],
  "next_cursor": null
}
```

`products` lists the products enabled for the pair that fit the market's side: a subset of
`cash_secured_put`, `put_spread` and `cash_put` on a put market, a subset of `covered_call` and
`call_spread` on a call market. A market with none is not listed. `settlement` is `{ "type": "pending" }`
until the market is settled, then `{ "type": "fixed", "price" }`.

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

Uses [common paging](ws-common.md#paging).

Optional `limit` defaults to 50. Omit `cursor` for the first page; pass the opaque
`next_cursor` string directly for the next page.
Rows are ordered by PDA ascending. `next_cursor: null` means the final page.
Invalid bounds return `invalid_limit`; malformed cursors return `invalid_cursor`.

```json
{
  "makers": [
    {
      "pda": "MakerPdaBase58",
      "maker_owner": "MakerOwnerBase58",
      "quote_signing": "MakerQuoteSigningBase58",
      "total_trades": 12,
      "total_volume": "1000000",
      "total_quotes": 55
    }
  ],
  "next_cursor": null
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

Uses [common paging](ws-common.md#paging).

Optional query: `delegate=<pubkey>` returns only vaults that key can operate as delegate: it is the
`primary_delegate` and holds an unexpired delegate grant. A value that is not a base58 pubkey returns `400` `invalid_delegate`.

```json
{
  "vaults": [
    {
      "vault_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "vault_id": "0909090909090909090909090909090909090909090909090909090909090909",
      "governance": "8qbHbw2BbbTHBW1sbeqakYXVKRQM8Ne7pLK7m6CVfeR",
      "fee_recipient": "CktRuQ2mttgRGkXJtyksdKHjUdc2C4TgDzyB98oEzy8",
      "primary_delegate": "GgBaCs3NCBuZN12kCJgAW63ydqohFkHEdfdEXBPzLHq",
      "main_mint": "LbUiWL3xVV8hTFYBVdbTNrpDo41NKS6o3LHHuDzjfcY",
      "second_mint": "QWmroo4YnnMqYW3cnxWkFdaTxGD3P7vMSzwMHGbUzwF",
      "second_leg_generation": "0",
      "share_mint": "US517G5965aydkZ46HS38QLi7UQiSojurfbQfKCELFx",
      "main_token_program": "legacy_spl",
      "second_token_program": "token_2022",
      "main_decimals": 9,
      "second_decimals": 6,
      "phase": "active",
      "safety": "normal",
      "capabilities": 5,
      "active_expiry_ts": "1703000000",
      "open_positions": "2",
      "total_shares": "100",
      "share_price": "1100000000",
      "equity": "120",
      "capital_generation": "8",
      "accepting_deposits": true,
      "equity_computed": false,
      "pending_deposit_count": 2,
      "pending_withdraw_count": 3,
      "pending_deposit_amount": "20",
      "pending_withdraw_shares": "30",
      "performance_fee_bps": 1000,
      "management_fee_bps": 200,
      "manager_stake_min_bps": 500,
      "share_price_hwm": "1200000000",
      "last_hwm_update_at": "40",
      "last_hwm_reset_at": "41",
      "last_settlement_at": "42",
      "capacity": {
        "type": "limited",
        "cap_limit": "1000",
        "remaining": "860"
      },
      "capital_fallback_delay_seconds": "43200",
      "deposit_window": {
        "type": "set",
        "set_at": "1700000000"
      },
      "balances": {
        "type": "observed",
        "main": "900",
        "second": "80",
        "live_share_supply": "80",
        "manager_stake_shares": "20",
        "observed_at": "43",
        "slot": "1000"
      },
      "staging": {
        "type": "staged",
        "mint": "cGfHiC6Kgg3FpFZvgwGcswsCRtp4aBP2fzuXRQPizuN",
        "from_leg": "second",
        "amount": "70",
        "staged_at": "1700000050",
        "generation": "2"
      },
      "staged_generation": "2",
      "policy": {
        "max_open_positions": 12,
        "max_rebalance_drift_bps": 100,
        "min_main_liquidity_bps": 500
      },
      "projection": {
        "version": "10",
        "slot": "11",
        "transaction_index": 3,
        "signature": "signature",
        "event_index": 12
      },
      "tvl_usd": {
        "type": "priced",
        "value_1e9": "18000",
        "price_published_at": "1702600050"
      },
      "stats": {
        "type": "current",
        "created_at": "1700000000",
        "computed_at": "1702600060",
        "generation": {
          "type": "covered",
          "started": {
            "type": "at",
            "at": "1700000100"
          },
          "premium": [
            {
              "quote_mint": "6URwbPipuA4MJLG7LCRRZuWnms3JZ9cRG3z9indXWz8G",
              "cycle": {
                "received": "5",
                "paid": {
                  "type": "available",
                  "atoms": "0"
                }
              },
              "generation": {
                "received": "50",
                "paid": {
                  "type": "available",
                  "atoms": "7"
                }
              }
            },
            {
              "quote_mint": "6YMEjhBqVTMaSRWcmVkLrnHZ22FWEDJEpTeonAg8GKSy",
              "cycle": {
                "received": "0",
                "paid": {
                  "type": "available",
                  "atoms": "0"
                }
              },
              "generation": {
                "received": "1",
                "paid": {
                  "type": "unavailable"
                }
              }
            }
          ],
          "fees": {
            "management_fee_assessed": "3",
            "performance_fee_assessed": "4",
            "realized_fee_value": "6"
          },
          "depositors": 11,
          "performance": {
            "returns": {
              "7d": {
                "type": "insufficient_history"
              },
              "30d": {
                "type": "available",
                "return_pct": 1.5,
                "window_start_at": "1700000100",
                "window_end_at": "1702600000"
              },
              "90d": {
                "type": "insufficient_history"
              },
              "all": {
                "type": "available",
                "return_pct": 2.0,
                "window_start_at": "1700000100",
                "window_end_at": "1702600000"
              }
            },
            "apy_30d": {
              "type": "available",
              "value": 19.9
            },
            "apy_90d": {
              "type": "insufficient_history"
            },
            "max_drawdown_pct": {
              "type": "available",
              "value": 0.5
            },
            "worst_cycle_return_pct": {
              "type": "available",
              "value": -0.25
            },
            "losing_cycles": {
              "type": "available",
              "value": 1
            },
            "cycles_completed": 4
          }
        }
      },
      "metadata": {
        "type": "set",
        "name": "SOL vault",
        "curator": "Acta",
        "strategy": "Weekly covered calls",
        "description": null,
        "website": "https://acta.markets",
        "twitter": "acta_markets",
        "logo_url": null
      }
    }
  ],
  "next_cursor": null
}
```

| Field | Values / meaning |
|---|---|
| `phase` | `idle`, `active`, `settling`, `processing_capital` |
| `safety` | `normal`, `frozen`, `emergency_unwind`, `withdraw_only`, `closing` |
| `main_token_program` / `second_token_program` | `legacy_spl`, `token_2022` |
| `total_shares` | Boundary checkpoint adjusted for contract mint/burn — **not** live SPL supply. Use `balances.live_share_supply` for the observed supply. |
| `balances` | `{ "type": "observed", "main", "second", "live_share_supply", "manager_stake_shares", "observed_at", "slot" }` from the balance watcher, or `{ "type": "unobserved" }` before the first observation lands. |
| `capital_fallback_delay_seconds` | Delay (default `43200`, max `259200`) after which anyone, not only the operator, may process a queued request. In `idle` it counts from the request's `created_at`; in `processing_capital` it counts from `active_expiry_ts`. In `idle`, deposits also need priced equity, which a non-operator can only compute with an aged withdraw request; otherwise cancel the deposit. |
| `staging` | `{ "type": "none" }` unless half of a [two-transaction swap](../quickstart/vault-operator-guide.md#two-transaction-swaps) is open: `{ "type": "staged", "mint", "from_leg": "main" \| "second", "amount", "staged_at", "generation" }`. `amount` (atoms of `mint`, decimal string) has left `from_leg` for the vault's account of `mint` and is not counted in `equity`. `staged_at` (Unix seconds) is the first stage. Capital processing waits until it is back. |
| `deposit_window` | `{ "type": "set", "set_at" }`: chain time of the last `SetAcceptingDeposits`; `{ "type": "unset" }` if the window was never set. |
| `projection` | Position of this row in the venue's on-chain event history. Compare `slot` / `transaction_index` / `event_index` to detect staleness. |
| `second_leg_generation`, `staged_generation` | Generation counters, decimal strings. |
| `capacity` | Tagged union. `{ "type": "unlimited" }` when the on-chain cap is `0`. Otherwise `{ "type": "limited", "cap_limit", "remaining" }` in main atoms, with `remaining = max(0, cap_limit − equity − pending_deposit_amount)`. |
| `stats` | Indexed statistics, refreshed every 60 s. `{ "type": "not_indexed" }` until they load; `{ "type": "behind", "created_at" }` while they are for an older capital generation than `capital_generation`; `{ "type": "current", "created_at", "computed_at", "generation" }` otherwise. `created_at` is the chain time of the vault's creation. |
| `tvl_usd` | `{ "type": "priced", "value_1e9", "price_published_at" }`: indicative `equity` at the main mint's USD oracle price, 1e9 scale; `{ "type": "unpriced" }` without a fresh price. |
| `stats.generation` | `{ "type": "none" }` when the generation's history is not covered yet. Otherwise `{ "type": "covered", "started", "premium", "fees", "depositors", "performance" }`. `started` is `{ "type": "at", "at" }` or `{ "type": "no_checkpoint" }`. `premium`: per quote mint, quote atoms received as writer and paid as holder (gross premium) in the current cycle and the generation; `paid` is `{ "type": "unavailable" }` when a holder leg's market lacks decimals or its open price. `fees`: assessed management and performance fees and realized fee value of the generation, in main atoms. `depositors`: wallets with a processed deposit in the generation. |
| `stats.generation.performance` | `returns` per window (`7d`, `30d`, `90d`, `all`): `{ "type": "available", "return_pct", "window_start_at", "window_end_at" }`, the booked share price return after fees in percent (`1.0` = 1%), or `{ "type": "insufficient_history" }` when the generation is younger than the window. `apy_30d`, `apy_90d` (the 30- and 90-day returns annualized by compounding), `max_drawdown_pct` (largest peak-to-trough fall of the booked share price), `worst_cycle_return_pct` and `losing_cycles` are `{ "type": "available", "value" }` or `{ "type": "insufficient_history" }`. `cycles_completed` counts finished cycles of the generation. |
| `metadata` | `{ "type": "none" }` when the operator set none. Otherwise `{ "type": "set", "name", "curator", "strategy", "description", "website", "twitter", "logo_url" }`. The six fields after `name` are free text the operator may leave out, so they are the API's only nullable values besides `next_cursor`. URLs are `https://`; `twitter` is a bare handle, no `@`. Edits appear within about a minute. Display only; `governance` is the authority, so show it next to the name. |

### GET `/api/v1/vaults/{pda}`

Returns a single vault object (the element shape above), or `404` `not_found` for an unknown, closed or
malformed PDA.

### GET `/api/v1/vaults/{pda}/positions`

Uses [common paging](ws-common.md#paging).

Query:

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `settled` | bool | unset | Filters on `settled_at`: `true` returns only rows that have one, `false` only rows that do not; unset returns both |
| `limit` | integer | 50 | 1 through 200 |
| `cursor` | string | unset | Opaque `next_cursor` value, URL-encoded; absent means the first page |

```json
{
  "positions": [
    {
      "vault_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "position_pda": "3EKkiwNLWqoUbzFkPrmKbtUB4EweE6f4STzevYUmezeL",
      "order_id": "0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c",
      "role": "writer",
      "market_pda": "CktRuQ2mttgRGkXJtyksdKHjUdc2C4TgDzyB98oEzy8",
      "position_type": "cash_secured_put",
      "strike": "150000000000",
      "product": {
        "type": "physical"
      },
      "quantity": "2000000000",
      "net_premium": "12500000",
      "counterparty_owner": "GgBaCs3NCBuZN12kCJgAW63ydqohFkHEdfdEXBPzLHq",
      "funded_amount": {
        "type": "available",
        "atoms": "300000000"
      },
      "status": "funded",
      "close": {
        "type": "open"
      },
      "created_at": "10",
      "acquired_at": "11",
      "transaction": {
        "type": "known",
        "signature": "sig"
      }
    },
    {
      "vault_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
      "position_pda": "8qbHbw2BbbTHBW1sbeqakYXVKRQM8Ne7pLK7m6CVfeR",
      "order_id": "0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b",
      "role": "holder",
      "market_pda": "CktRuQ2mttgRGkXJtyksdKHjUdc2C4TgDzyB98oEzy8",
      "position_type": "cash_put",
      "strike": "900",
      "product": {
        "type": "spread",
        "upper_strike": "900",
        "collateral": "20"
      },
      "quantity": "5",
      "net_premium": "7",
      "counterparty_owner": "GgBaCs3NCBuZN12kCJgAW63ydqohFkHEdfdEXBPzLHq",
      "funded_amount": {
        "type": "available",
        "atoms": "8"
      },
      "status": "settled",
      "close": {
        "type": "closed",
        "closed_at": "12",
        "is_otm": true
      },
      "created_at": "10",
      "acquired_at": "11",
      "transaction": {
        "type": "known",
        "signature": "sig"
      }
    }
  ],
  "next_cursor": "412:3:1:7cVfgArCheMR6Cs4t6vz5rfnqd56vZq4ndaBrY5xkxXy"
}
```

| Field | Values / meaning |
|---|---|
| `role` | `writer` (the vault posted collateral) or `holder` (the vault bought the option) |
| `position_type` | `covered_call`, `cash_secured_put`, `call_spread`, `put_spread`, `cash_put` |
| `strike` | Strike in 1e9 scale. The lower strike for `call_spread` and `put_spread`; `K` for a cash put, equal to its `upper_strike` |
| `product` | `{ "type": "physical" }` for covered calls and cash-secured puts. `{ "type": "spread", "upper_strike", "collateral" }` for spreads and cash puts: the upper strike and the quote atoms the position reserves |
| `net_premium` | Net premium in quote atoms after the protocol fee; negative for a `holder` row |
| `status` | `open`, `funded`, `liquidated`, `settled` |
| `funded_amount` | `{ "type": "available", "atoms" }` in the funding mint, or `{ "type": "unavailable" }` when no funding amount was recorded |
| `close` | `{ "type": "open" }`, or `{ "type": "closed", "closed_at", "is_otm" }` once the position is settled or liquidated |
| `counterparty_owner` | The other side's wallet |
| `transaction` | `{ "type": "known", "signature" }` of the open, or `{ "type": "unknown" }` when it was projected without its transaction |
| `next_cursor` | `slot:transaction_index:event_index:position`. Opaque — pass it back verbatim. `null` on the last page. |

A malformed `cursor` returns `400` with code `invalid_cursor`.

### GET `/api/v1/vaults/{pda}/exposure`

Uses [common paging](ws-common.md#paging).

Positions are paged by PDA ascending using [common paging](ws-common.md#paging).
Totals always cover the complete cached exposure snapshot, not just the page.

Open and funded positions with an indicative spot. Positions are reloaded after 10 s and never served older than 15 s (`as_of` is when they were read; a failed reload past that returns `503`); `spot` is read per request. Needs the database (`503` `db_disabled` without one).

```json
{
  "vault_pda": "4vJ9JU1bJJE96FWSJKvHsmmFADCg4gpZQff4P3bkLKi",
  "as_of": "1702600005",
  "positions": [
    {
      "position_pda": "7Z8ftDAzMvoyXnGEJye8DurzgQQXLAbYCaeeesM7UKHa",
      "market_pda": "7d3y2WdzxE7CfsWjkGy3WndkvZcj1EHMkzKJiFPiDecH",
      "position_type": "covered_call",
      "role": "writer",
      "status": "open",
      "strike": "140000000000",
      "quantity": "2000000000",
      "expiry_ts": "1703000000",
      "underlying_mint": "LbUiWL3xVV8hTFYBVdbTNrpDo41NKS6o3LHHuDzjfcY",
      "quote_mint": "6URwbPipuA4MJLG7LCRRZuWnms3JZ9cRG3z9indXWz8G",
      "decimals": {
        "type": "known",
        "underlying": 9,
        "quote": 6
      },
      "spot": {
        "type": "priced",
        "price": "150000000000",
        "published_at": "1702600000"
      },
      "moneyness": {
        "type": "available",
        "distance_pct": -6.666666666666667,
        "itm": true
      },
      "intrinsic": {
        "type": "available",
        "atoms": "-20000000"
      }
    },
    {
      "position_pda": "7gyGAp71YXQRoxmFBaHxofQXAipvgHyBKPyxmdSJxyvz",
      "market_pda": "7d3y2WdzxE7CfsWjkGy3WndkvZcj1EHMkzKJiFPiDecH",
      "position_type": "cash_secured_put",
      "role": "holder",
      "status": "funded",
      "strike": "155000000000",
      "quantity": "2000000000",
      "expiry_ts": "1703000000",
      "underlying_mint": "LbUiWL3xVV8hTFYBVdbTNrpDo41NKS6o3LHHuDzjfcY",
      "quote_mint": "6URwbPipuA4MJLG7LCRRZuWnms3JZ9cRG3z9indXWz8G",
      "decimals": {
        "type": "known",
        "underlying": 9,
        "quote": 6
      },
      "spot": {
        "type": "priced",
        "price": "150000000000",
        "published_at": "1702600000"
      },
      "moneyness": {
        "type": "available",
        "distance_pct": -3.3333333333333335,
        "itm": true
      },
      "intrinsic": {
        "type": "available",
        "atoms": "10000000"
      }
    },
    {
      "position_pda": "7porTR32j7zt69GG4AwoPQx3f3FL2RLpSDKGtPXWTeaQ",
      "market_pda": "7ktZK7a28phex41kcsct6YBHQt38MMezsoecq1UuiKFh",
      "position_type": "covered_call",
      "role": "writer",
      "status": "open",
      "strike": "140000000000",
      "quantity": "2000000000",
      "expiry_ts": "1703000000",
      "underlying_mint": "LbUiWL3xVV8hTFYBVdbTNrpDo41NKS6o3LHHuDzjfcY",
      "quote_mint": "6YMEjhBqVTMaSRWcmVkLrnHZ22FWEDJEpTeonAg8GKSy",
      "decimals": {
        "type": "unknown"
      },
      "spot": {
        "type": "unpriced"
      },
      "moneyness": {
        "type": "unavailable"
      },
      "intrinsic": {
        "type": "unavailable"
      }
    }
  ],
  "next_cursor": null,
  "totals": [
    {
      "quote_mint": "6URwbPipuA4MJLG7LCRRZuWnms3JZ9cRG3z9indXWz8G",
      "notional_at_strike": {
        "type": "available",
        "atoms": "590000000"
      },
      "outstanding_intrinsic": {
        "type": "available",
        "atoms": "-10000000"
      }
    },
    {
      "quote_mint": "6YMEjhBqVTMaSRWcmVkLrnHZ22FWEDJEpTeonAg8GKSy",
      "notional_at_strike": {
        "type": "unavailable"
      },
      "outstanding_intrinsic": {
        "type": "unavailable"
      }
    }
  ]
}
```

| Field | Values / meaning |
|---|---|
| `spot` | `{ "type": "priced", "price", "published_at" }`: underlying price in quote, 1e9 scale, from the oracle cache; `{ "type": "unpriced" }` when a leg is stale or retired |
| `moneyness` | `{ "type": "available", "distance_pct", "itm" }` for covered calls and cash-secured puts against a priced spot; `{ "type": "unavailable" }` otherwise. `distance_pct` is `100 × (strike − spot) / spot` for a call and `100 × (spot − strike) / spot` for a put; positive is out of the money. `itm` is `spot` past the strike; equality is out of the money |
| `intrinsic` | `{ "type": "available", "atoms" }`: value moved if the position expired at `spot`, in quote atoms: `0` out of the money, positive for `holder`, negative for `writer`. Not a cash flow or fair value (time value is ignored). `{ "type": "unavailable" }` without a spot, for spreads, or when decimals are unknown |
| `decimals` | `{ "type": "known", "underlying", "quote" }`, or `{ "type": "unknown" }` when market metadata is missing |
| `totals` | Per quote mint: `notional_at_strike` and `outstanding_intrinsic` (signed), each `{ "type": "available", "atoms" }` or `{ "type": "unavailable" }` when any of its positions lacks the value |

### GET `/api/v1/vaults/{pda}/cycles`

Uses [common paging](ws-common.md#paging).

Query: optional `limit` (default 50, maximum 200) and `cursor`; see [common paging](ws-common.md#paging).
Cycles are returned newest first.

```json
{
  "cycles": [
    {
      "cycle_number": "13",
      "capital_generation": {
        "type": "known",
        "generation": "8"
      },
      "expiry_ts": "1703209600",
      "started_at": "1702604800",
      "share_price_start": "1012500000",
      "equity_start": "101250000000",
      "state": {
        "type": "priced",
        "share_price_end": "1020000000",
        "equity_end": "102000000000",
        "return_pct": {
          "type": "available",
          "value": 0.0074
        }
      },
      "positions_opened": 2,
      "positions_settled": 2,
      "performance_fee_assessed": "0",
      "management_fee_assessed": "0"
    },
    {
      "cycle_number": "12",
      "capital_generation": {
        "type": "known",
        "generation": "8"
      },
      "expiry_ts": "1702604600",
      "started_at": "1701999800",
      "share_price_start": "1000000000",
      "equity_start": "100000000000",
      "state": {
        "type": "finished",
        "finished_at": "1702604800",
        "share_price_end": "1012500000",
        "equity_end": "101250000000",
        "return_pct": {
          "type": "available",
          "value": 0.0125
        }
      },
      "positions_opened": 4,
      "positions_settled": 4,
      "performance_fee_assessed": "125000000",
      "management_fee_assessed": "40000000"
    },
    {
      "cycle_number": "11",
      "capital_generation": {
        "type": "known",
        "generation": "7"
      },
      "expiry_ts": "1701999600",
      "started_at": "1701394800",
      "share_price_start": "1000000000",
      "equity_start": "100000000000",
      "state": {
        "type": "finished",
        "finished_at": "1701999800",
        "share_price_end": "1012500000",
        "equity_end": "101250000000",
        "return_pct": {
          "type": "available",
          "value": 0.0125
        }
      },
      "positions_opened": 4,
      "positions_settled": 4,
      "performance_fee_assessed": "125000000",
      "management_fee_assessed": "40000000"
    },
    {
      "cycle_number": "10",
      "capital_generation": {
        "type": "unknown"
      },
      "expiry_ts": "1701394600",
      "started_at": "1700789800",
      "share_price_start": "1000000000",
      "equity_start": "100000000000",
      "state": {
        "type": "finished",
        "finished_at": "1701394800",
        "share_price_end": "1012500000",
        "equity_end": "101250000000",
        "return_pct": {
          "type": "available",
          "value": 0.0125
        }
      },
      "positions_opened": 4,
      "positions_settled": 4,
      "performance_fee_assessed": "125000000",
      "management_fee_assessed": "40000000"
    }
  ],
  "next_cursor": "10"
}
```

`state` is `{ "type": "running" }`, then `{ "type": "priced", "share_price_end", "equity_end", "return_pct" }`
once equity is computed at the boundary, then `{ "type": "finished", "finished_at", "share_price_end",
"equity_end", "return_pct" }`. `return_pct` is `{ "type": "available", "value" }` or
`{ "type": "unavailable" }` when the start share price is zero. `capital_generation` is
`{ "type": "known", "generation" }`, or `{ "type": "unknown" }` for cycles recorded before generations
were tracked.
`return_pct` is a display value; compute exact returns from the share prices with BigInt.

### GET `/api/v1/vaults/{pda}/deposit-requests` and `/withdraw-requests`

Uses [common paging](ws-common.md#paging).

Pending capital requests, for curators batching `ProcessDeposits` / `ProcessWithdrawals`.
Query: optional `limit` (default 50, maximum 200) and `cursor`; see [common paging](ws-common.md#paging).
Requests are ordered by wallet; `next_cursor` is `null` on the last page. Withdraw requests carry
`shares` instead of `amount`. `transaction` is `{ "type": "unknown" }` when the request was projected without its transaction.

```json
{
  "requests": [
    {
      "wallet": "WalletBase58",
      "amount": "1000000",
      "created_at": "1710000000",
      "transaction": { "type": "known", "signature": "TxSignatureBase58" }
    }
  ],
  "next_cursor": null
}
```

| Status | Code | When |
|---|---|---|
| `400` | `invalid_limit` | `limit` is 0 or above 200 |
| `400` | `invalid_cursor` | `cursor` is not a wallet address |
| `404` | `not_found` | the vault is unknown |
| `429` | `rate_limited` | per-IP limit; retry with backoff |
| `503` | `db_busy`, `server_busy` | transient; retry with backoff |
| `503` | `db_disabled` | the server runs without a database; permanent, do not retry |

### GET `/api/v1/vaults/{pda}/depositors/{wallet}`

The wallet's pending requests and lifetime totals. A wallet that never used the vault gets `200`
with `deposit_request`, `withdraw_request` and `totals` all `{ "type": "none" }`. A request's `created_at` is chain
block time. `totals.total_withdrawn` counts the main leg only; second-leg payouts (final redemption)
appear per event as `second_amount` in the history.

```json
{
  "vault_pda": "VaultPdaBase58",
  "wallet": "WalletBase58",
  "deposit_request": {
    "type": "pending",
    "amount": "1000000",
    "created_at": "1710000000",
    "transaction": { "type": "known", "signature": "TxSignatureBase58" }
  },
  "withdraw_request": { "type": "none" },
  "totals": {
    "type": "recorded",
    "shares_minted": "1000000000",
    "shares_burned": "0",
    "total_deposited": "1000000",
    "total_withdrawn": "0",
    "last_activity_at": "1710000000"
  }
}
```

### GET `/api/v1/vaults/{pda}/depositors/{wallet}/history`

Uses [common paging](ws-common.md#paging).

Query: optional `limit` (default 50, maximum 200) and `cursor`; see [common paging](ws-common.md#paging).

```json
{
  "events": [
    {
      "vault_pda": "VaultPdaBase58",
      "action": "deposit_processed",
      "amount": "1000000",
      "second_amount": "0",
      "shares": "1000000000",
      "boundary_price": { "type": "set", "share_price": "1000000000" },
      "occurred_at": "1710000000",
      "transaction_signature": "TxSignatureBase58"
    }
  ],
  "next_cursor": null
}
```

| Field | Values / meaning |
|---|---|
| `action` | `deposit_processed`, `withdraw_processed`, `deposit_cancelled`, `withdraw_cancelled`, `deposit_refunded` |
| `boundary_price` | `{ "type": "set", "share_price" }`: the share price the request was processed at; `{ "type": "none" }` for cancellations and refunds |
| `next_cursor` | Opaque — pass it back verbatim. `null` on the last page. |

### GET `/api/v1/depositors/{wallet}`

Uses [common paging](ws-common.md#paging).

Vaults are paged by PDA ascending using [common paging](ws-common.md#paging).
Visibility filters apply before the page limit.

The wallet's portfolio: one entry per vault the wallet has a request or history in. Each entry has
the same shape as `/vaults/{pda}/depositors/{wallet}`.

```json
{
  "wallet": "WalletBase58",
  "vaults": [
    {
      "vault_pda": "VaultPdaBase58",
      "wallet": "WalletBase58",
      "deposit_request": { "type": "none" },
      "withdraw_request": { "type": "none" },
      "totals": { "type": "none" }
    }
  ],
  "next_cursor": null
}
```

### GET `/api/v1/depositors/{wallet}/history`

Uses [common paging](ws-common.md#paging).

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


Configuration lists (`/markets`, `/tokens`, `/vaults`) have an immutable ceiling
of 1,000 entries and return pages of at most 200. Oversized configuration fails. Portfolio history and income totals use WS `GetPositions` and
`GetPositionSummary` on the authenticated wallet connection.
