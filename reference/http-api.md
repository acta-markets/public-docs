# Acta HTTP API
> Trading is WebSocket-based. See `maker-api.md` and `taker-api.md`.

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
- `not_found` (404)
- `db_disabled` (503)
- `data_validation_error` (500)
- `schema_error` (500)
- `db_error` (500)

Temporary unavailability may return HTTP `503`.

---

## Health

### GET `/health`

Returns service health and metadata.

### GET `/ready`

Returns readiness status.

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

`GET /api/v1/markets` returns tradable markets only: not finalized, not disabled, and before the effective trading cutoff. `underlying` filters by symbol before that tradability filter. There is no `active=false` mode; use `/api/v1/markets/:pda` for a specific market.

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

### GET `/api/v1/markets/:pda`

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
      "total_quotes": 55
    }
  ]
}
```

### GET `/api/v1/makers/:pda`

Get a single maker DTO.

---

## RFQs

### GET `/api/v1/rfqs`

Query:
- `status` (optional string)
- `limit` (optional, max 100, default 50)
- `offset` (optional, default 0)

Pagination:
- Response contains the current page only. The backend does not return `total`.

```json
{
  "rfqs": [
    {
      "id": "uuid",
      "taker": "TakerBase58",
      "market_pda": "MarketPdaBase58",
      "position_type": "covered_call",
      "strike": 100000000000,
      "quantity": 1000000,
      "status": "filled",
      "created_at": 1710000000,
      "expires_at": 1710000030,
      "filled_at": 1710000040,
      "quotes_received": 3,
      "best_price": 50000000
    }
  ]
}
```

### GET `/api/v1/rfqs/:id`

Returns:
- `rfq`
- `quotes`
- optional `order`

---

## Orders

### GET `/api/v1/orders`

Query:
- `status` (optional string)
- `limit` (optional, max 100, default 50)
- `offset` (optional, default 0)

Pagination:
- Response contains the current page only. The backend does not return `total`.

### GET `/api/v1/orders/:id`

```json
{
  "id": "uuid",
  "rfq_id": "uuid",
  "taker": "TakerBase58",
  "maker_pda": "MakerPdaBase58",
  "status": "confirmed",
  "tx_signature": "5eyk...base58sig",
  "position_pda": "PositionPdaBase58",
  "price": 50000000,
  "strike": 100000000000,
  "quantity": 1000000,
  "created_at": 1710000000,
  "submitted_at": 1710000003,
  "confirmed_at": 1710000008,
  "error_message": null
}
```

---

## Stats

### GET `/api/v1/stats`

```json
{
  "total_volume_24h": 1000000,
  "total_trades_24h": 150,
  "active_markets": 12,
  "active_makers": 5,
  "connected_makers": 3
}
```

This differs from WS `GlobalStats` in `Snapshot` and `StatsUpdate`. HTTP includes `connected_makers` (live WS sessions) and omits `total_price_24h` and `active_rfqs`.

---

## Related

- WS common conventions: `ws-common.md`
- Maker WS reference: `maker-api.md`
- Taker WS reference: `taker-api.md`
