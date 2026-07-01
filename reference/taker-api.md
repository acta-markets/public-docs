# Acta Taker API Reference

## Endpoint

```
wss://devnet-api.acta.markets/taker
wss://beta-api.acta.markets/taker
```


## Connection flow

```
Connect -> Hello -> Welcome -> Auth -> AuthSuccess -> Snapshot -> Subscribe -> ...
```

Auth is one of:
- **Resume** (returning user): `ResumeAuth { session_id }` → `AuthSuccess { session_id, expires_at }`
- **Fresh sign** (first time or resume failed): `StartAuth { pubkey }` → `AuthRequest` → `AuthChallenge` → `AuthSuccess { session_id, expires_at }`

### Protocol version and handshake constraints

- Current server protocol: `protocol_version=1.0.0`
- Current server minimum supported version: `min_supported_version=1.0.0`
- `Hello` MUST be the first client message
- `Hello` timeout: `5000ms`
- If a client sends non-`Hello` first, server closes with `hello_required` / `hello_timeout`

### Connection policy defaults

- `auth_deadline`: `15s` (after auth challenge was issued)
- `idle_timeout`: `90s`
- server WS ping interval: `30s`
- server liveness check interval: `1s`
- max consecutive parse errors before close: `3`

### Taker auth lifecycle

- Taker auth is lazy by design: `StartAuth`/`ResumeAuth` can be sent immediately after `Welcome` or later.
- `Snapshot` is sent after `AuthSuccess`.

### ResumeAuth (session resume)

If the client has a saved `session_id` from a previous `AuthSuccess`, it can skip the wallet signature:

```json
{ "type": "ResumeAuth", "data": { "session_id": "uuid-from-previous-auth" } }
```

Server responds with:
- `AuthSuccess { session_id, expires_at }` if the session is valid
- `AuthError { reason: "session_expired" }` if the session is invalid, expired, or revoked

On `AuthError`, client falls back to the full sign flow (`StartAuth`).

Failed `ResumeAuth` counts as an auth attempt toward the `max_auth_attempts` limit (default: 3).
After the limit is reached, the server closes the connection.

**Session lifetime:**
- Server stores sessions with a TTL (default: 24 hours).
- Sessions survive WS disconnects (not revoked on close).
- `expires_at` may be extended on activity (sliding window), capped at an absolute maximum (7 days).
- Client should persist `session_id` + `expires_at` (e.g. localStorage) and use `ResumeAuth` on reconnect if `now < expires_at`.

### StartAuth / AuthChallenge (full sign flow)

```json
{ "type": "StartAuth", "data": { "pubkey": "WalletPubkeyBase58" } }
```

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "sign-me",
    "signature": "base58sig",
    "pubkey": "WalletPubkeyBase58"
  }
}
```

### AuthSuccess

```json
{
  "type": "AuthSuccess",
  "data": {
    "session_id": "uuid",
    "expires_at": 1710086400
  }
}
```

`expires_at` is a Unix timestamp (seconds). For taker sessions it is always a number.
For maker sessions it is `null` (makers do not use session resume).
Client should persist both `session_id` and `expires_at` for future `ResumeAuth`.

### AuthError

```json
{
  "type": "AuthError",
  "data": {
    "reason": "session_expired",
    "message": "optional detail"
  }
}
```

`reason` is a `snake_case` code. `message` is optional and may be omitted.

### Subscribe

```json
{
  "type": "Subscribe",
  "data": {
    "request_id": "uuid",
    "channels": ["markets", "trades", "stats"],
    "underlying_mints": ["UnderlyingMintBase58"],
    "quote_mints": ["QuoteMintBase58"]
  }
}
```

`request_id` is required. The server always responds with `SubscribeAck` echoing the
`request_id` and the channels that were newly added.

`underlying_mints` (optional): filter broadcast events to markets with specific underlying mints. `quote_mints` (optional): filter by quote mint. If both are omitted, subscriptions apply to all markets.

> **Important:** Subscriptions do not persist across WebSocket disconnects. After reconnect, resend `Subscribe` to restore channel subscriptions.

### Snapshot (server -> taker)

```json
{
  "type": "Snapshot",
  "data": {
    "markets": [
      {
        "pda": "MarketPdaBase58",
        "underlying": "UnderlyingMintBase58",
        "quote": "QuoteMintBase58",
        "expiry_ts": 1710600000,
        "is_put": false
      }
    ]
  }
}
```

`Snapshot.markets` uses a compact `MarketInfo` shape (`pda`, `underlying`, `quote`, `expiry_ts`, `is_put`). This is a different DTO from the `MarketDescriptor` used in `RfqCreated.order_options` and discovery responses. Use `GetMarketDescriptors` for the full market descriptor with `chain_id`, `program_id`, `collateral_mint`, `settlement_mint`, etc.

---

## Message index

### Client -> Server

- `Hello`, `ResumeAuth`, `StartAuth`, `AuthChallenge`, `Logout`
- `Subscribe`, `Unsubscribe`, `Ping`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `RfqRequest`, `AcceptQuote`, `CancelRfq`, `SubmitSignedSponsoredTx`
- `GetIndicativePrices`, `GetEarnSummary`, `GetTokenMarketsInfo`
- `GetMarkets`, `GetMarketDescriptors`, `GetExpiries`, `GetTokens`
- `GetPositions`, `GetMyActiveRfqs`, `GetOrderStatus`, `GetSubscriptions`
- `GetTokenCaps`
- `RedeemInvite`, `ClaimReferralCode`, `GetMyReferralInfo`

### Server -> Taker (direct)

- `Welcome`, `VersionMismatch`, `LogoutSuccess`
- `AuthRequest`, `AuthSuccess`, `AuthError`
- `Snapshot`
- `RfqCreated`, `QuoteReceived`
- `SponsoredTxToSign`, `OrderAccepted`, `OrderSubmitted`, `OrderConfirmed`, `OrderFailed`
- `RfqAvailableAgain`, `RfqClosed`
- `IndicativePrices`, `EarnSummary`
- `RequireInvite`, `InviteRedeemed`, `ReferralCodeClaimed`, `MyReferralInfo`
- `SubscriptionUpdated`
- `Error`, `RequestError`, `Pong`
- `SubscribeAck`, `UnsubscribeAck`

### Server -> Taker (responses / query results)

- `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`
- `Positions`, `MyActiveRfqs`, `OrderStatus`, `Subscriptions`
- `TokenCaps`, `TokenMarketsInfo`

### Server -> Taker (broadcast if subscribed)

- `TradeExecuted`, `StatsUpdate`, `ChainEvent`
- `MarketCreated`, `MarketFinalized`

`PositionUpdated` is **not** delivered to takers — it is a maker-owner-only push. Track your position state via `ChainEvent` (position settled/liquidated on the `chain_events` channel) plus `GetPositions`.

---

## RFQ flow

### RfqRequest (taker -> server)

```json
{
  "type": "RfqRequest",
  "data": {
    "market": "MarketPdaBase58",
    "position_type": "cash_secured_put",
    "strike": 136000000000,
    "quantity": 20000000,
    "timeout_seconds": 30,
    "client_request_id": "optional-uuid"
  }
}
```

`client_request_id` is optional and used for idempotent RFQ creation.

### RfqCreated (server -> taker)

```json
{
  "type": "RfqCreated",
  "data": {
    "rfq_id": "uuid",
    "rfq_version": 1,
    "client_request_id": "optional-uuid",
    "expires_at": 1710000030,
    "created_at": 1710000000,
    "order_options": [{ "strike": 136000000000 }]
  }
}
```

`order_options` contains the server-computed strike set for this RFQ — the same set that makers receive in `RfqBroadcast.order_options`. The number of strikes depends on the market and current index price.

### QuoteReceived (server -> taker)

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "uuid",
    "strike": 136000000000,
    "maker": "MakerOwnerPubkeyBase58",
    "price": 50000000,
    "net_price": 49750000,
    "valid_until": 1710000310,
    "nonce": 42,
    "order_id": "0x...64chars"
  }
}
```

- `price` — gross quote from the maker. Hash-bound via `order_id` — use this for `AcceptQuote` and all order operations.
- `net_price` — display-only net price estimate after protocol fee deduction. Present when the server has loaded the on-chain fee config. The contract's authoritative fee is computed at `OpenPosition` as `min(premium_fee, volume_fee)` after token-decimal scaling. Use `net_price` for UI display and `price` for order operations.

### AcceptQuote (taker -> server)

```json
{
  "type": "AcceptQuote",
  "data": {
    "rfq_id": "uuid",
    "maker": "MakerOwnerPubkeyBase58",
    "order_id": "0x...64chars"
  }
}
```

### CancelRfq (taker -> server)

```json
{
  "type": "CancelRfq",
  "data": {
    "rfq_id": "uuid",
    "request_id": "uuid"
  }
}
```

Cancels an active RFQ. `request_id` is required on the wire. The server responds with `RfqClosed` and `reason: "taker_cancelled"`.

### SponsoredTxToSign (server -> taker)

```json
{
  "type": "SponsoredTxToSign",
  "data": {
    "order_id": "0x...64chars",
    "tx_base64": "...",
    "signature_deadline": 1710000040
  }
}
```

### SubmitSignedSponsoredTx (taker -> server)

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": {
    "order_id": "0x...64chars",
    "tx_base64": "..."
  }
}
```

### OrderAccepted (server -> taker)

Sent after `AcceptQuote` is processed and the quote is locked.

```json
{
  "type": "OrderAccepted",
  "data": {
    "order_id": "0x...64chars"
  }
}
```

### OrderStatus (server -> taker)

```json
{
  "type": "OrderStatus",
  "data": {
    "request_id": "uuid",
    "order_id": "0x...64chars",
    "status": "submitted",
    "rfq_id": "uuid",
    "tx_signature": "5eyk...base58sig",
    "position_pda": "PositionPdaBase58",
    "error_reason": null
  }
}
```

`OrderStatusMessage` does not include `order_version` in current wire schema.
Order lifecycle versioning is carried by `OrderSubmitted` / `OrderConfirmed` / `OrderFailed`
via `order_version` fields (always present; `#[serde(default)]` keeps backward compatibility with older clients).

### OrderSubmitted (server -> taker)

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x...64chars",
    "tx_signature": "5eyk...base58sig",
    "order_version": 1
  }
}
```

### OrderConfirmed (server -> taker)

```json
{
  "type": "OrderConfirmed",
  "data": {
    "order_id": "0x...64chars",
    "position_pda": "PositionPdaBase58",
    "order_version": 2
  }
}
```

### OrderFailed (server -> taker)

```json
{
  "type": "OrderFailed",
  "data": {
    "order_id": "0x...64chars",
    "reason": "transaction simulation failed",
    "order_version": 2
  }
}
```

`reason` values:

| Value | Meaning | Retryable? |
|-------|---------|------------|
| `blockhash_expired` | Solana blockhash expired before confirmation | Yes — wait for `RfqAvailableAgain`, then re-send `AcceptQuote` |
| `on_chain` | On-chain program error (simulation or execution) | No |
| `submission_rejected` | Solana RPC rejected the transaction | No |
| `safety_timeout` | Confirmation timed out after max retries | No |
| `shutdown` | Server shutting down during settlement | No |

For `blockhash_expired`: the server automatically retries settlement up to 5 times. If all retries fail, the taker receives `OrderFailed` with `reason: "blockhash_expired"` followed by `RfqAvailableAgain`. The taker can then re-accept the same or a different quote.

### Terminal semantics: OrderConfirmed vs RfqClosed

- `OrderConfirmed` is the order-level confirmation event: the selected order is confirmed on-chain and
  carries `position_pda` (and `order_version`). The `tx_signature` is delivered earlier, on
  `OrderSubmitted` — it is not repeated on `OrderConfirmed`.
- `RfqClosed` is the terminal RFQ event: the RFQ is finished and must be
  treated as closed for further quoting/accept actions.
- On a successful fill, takers receive `OrderConfirmed` followed by `RfqClosed`
  in close succession.
- `RfqClosed.your_quote` / `RfqClosed.winner` are optional fields and may be omitted.

---

## Discovery

Current taker discovery requests:

```json
{ "type": "GetMarkets", "data": { "request_id": "uuid" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "uuid", "active_only": true } }
{ "type": "GetExpiries", "data": { "request_id": "uuid" } }
{ "type": "GetTokens", "data": { "request_id": "uuid", "active_only": true } }
{ "type": "GetPositions", "data": { "request_id": "uuid" } }
{ "type": "GetMyActiveRfqs", "data": { "request_id": "uuid" } }
{ "type": "GetOrderStatus", "data": { "request_id": "uuid", "order_id": "0x...64chars" } }
{ "type": "GetSubscriptions", "data": { "request_id": "uuid" } }
{ "type": "GetIndicativePrices", "data": { "request_id": "uuid", "market": "MarketPdaBase58", "position_type": "covered_call" } }
{ "type": "GetTokenCaps", "data": { "request_id": "uuid" } }
{ "type": "GetEarnSummary", "data": { "request_id": "uuid" } }
{ "type": "GetTokenMarketsInfo", "data": { "request_id": "uuid", "underlying_mint": "UnderlyingMintBase58" } }
```

`request_id` is required for these query-style requests.

Query-style responses echo the same `request_id`:

- `Markets`, `MarketDescriptors`, `Expiries`, `Tokens`
- `Positions`, `MyActiveRfqs`, `OrderStatus`, `Subscriptions`
- `IndicativePrices`
- `TokenCaps`
- `EarnSummary`
- `TokenMarketsInfo`

`active_only` behavior:
- default is `true` when omitted (wire default for `GetMarketDescriptors`, `GetTokens`)
- for `GetMarketDescriptors`/`GetTokens`, `active_only=true` means non-finalized markets with `expiry_ts > now`; `active_only=false` returns all markets/tokens

### Discovery strict policy

- `GetMarketDescriptors` and `GetTokens` require `prices.size_by_mint` config for every underlying mint in the DB.
- If any market is missing a size rule, the entire request fails with `missing_size_rule_for_underlying_mint`.
- Partial discovery is not supported.
- `config.toml` `[prices.size_by_mint]` must include all underlying mints used by markets.

### MarketDescriptors payload

`MarketDescriptors` returns `MarketDescriptorInfo[]`:

```typescript
type MarketDescriptorInfo = {
  market: {
    chain_id: number;
    program_id: string;
    market_pda: string;
    underlying_mint: string;
    quote_mint: string;
    expiry_ts: number;
    is_put: boolean;
    collateral_mint: string;
    settlement_mint: string;
  };
  underlying_oracle_pda: string;
  quote_oracle_pda: string;
  underlying_decimals: number;
  quote_decimals: number;
  size_rule: {
    min_size: number;
    max_size: number;
    step: number;
  };
  underlying_symbol: string;
  quote_symbol: string;
};
```

`size_rule` semantics (per underlying mint):
- `min_size <= quantity <= max_size`
- `(quantity - min_size) % step == 0`
- if rule for market underlying mint is missing, server rejects RFQ (`Generic` error with `code: "missing_size_rule_for_underlying_mint"`)

For cash-secured puts, the size_rule still constrains the underlying quantity on the wire, not the quote deposit amount. Frontends that collect user input in quote terms (e.g. USDC) must convert before sending `RfqRequest`. See `ws-common.md` "Quantity and collateral by position type" for conversion formulas and SDK helpers.

`underlying_symbol`, `quote_symbol`, and token `symbol` are mandatory and non-null in discovery payloads.

### Tokens payload

`Tokens` returns:

```typescript
type TokenInfo = {
  mint: string;
  decimals: number;
  size_rule: {
    min_size: number;
    max_size: number;
    step: number;
  };
  symbol: string;
};

type Tokens = {
  request_id: string;
  underlyings: TokenInfo[];
  quotes_by_underlying: Record<string, TokenInfo[]>;
};
```

`size_rule` and `symbol` are always present. For `quotes_by_underlying`, each quote token entry carries the same rule as its parent underlying key.

### Positions payload

`Positions` returns `PositionInfo[]`:

```json
{
  "type": "Positions",
  "data": {
    "request_id": "uuid",
    "positions": [
      {
        "pda": "PositionPdaBase58",
        "market": "MarketPdaBase58",
        "underlying_mint": "UnderlyingMintBase58",
        "quote_mint": "QuoteMintBase58",
        "position_type": "cash_secured_put",
        "status": "open",
        "strike": 136000000000,
        "quantity": 20000000,
        "price": 50000000,
        "total_premium": 987654,
        "created_at": 1710000042,
        "expiry_ts": 1710600000,
        "is_otm": null
      }
    ]
  }
}
```
`is_otm` is `null` for open/funded positions. Set to `true` (out-of-the-money) or `false` (in-the-money) after settlement or liquidation. Omitted from the wire when `null`.

Field semantics:
- `price`: gross premium per 1 underlying unit (1e9 scale)
- `total_premium`: net premium amount from on-chain position state (quote atomic units)
- all amount fields are unsigned (`u64`); timestamps remain Unix seconds

### MyActiveRfqs payload

```json
{
  "type": "MyActiveRfqs",
  "data": {
    "request_id": "uuid",
    "rfqs": [
      {
        "rfq_id": "uuid",
        "market": "MarketPdaBase58",
        "position_type": "cash_secured_put",
        "strike": 136000000000,
        "quantity": 20000000,
        "expires_at": 1710000030,
        "state": "active",
        "locked_order_id": null,
        "quotes_count": 3,
        "best_price": 50000000
      }
    ]
  }
}
```

`state` values: `active`, `pending_signature`, `enqueued`.
`locked_order_id` is present when the RFQ is locked on a specific order (state = `pending_signature` or `enqueued`).
`best_price` may be `null` if no quotes have been submitted.

### Position lifecycle (taker perspective)

Once your position is `open`:

1. The maker may deposit the settlement asset before market expiry.
2. After expiry, the market is finalized with an oracle price.
3. If your position is OTM, you get your collateral back. If ITM, you receive the maker's settlement asset.
4. If the maker did not fund and your position is ITM, normal settlement fails until a liquidator submits a liquidation transaction. You receive the settlement asset when that transaction lands.

See [Protocol Flow](protocol-flow.md) for all settlement scenarios.

---

## Dynamic subscription management

Modify subscriptions incrementally without replacing the full set:

```json
{ "type": "AddMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"], "quote_mints": ["MintBase58"] } }
{ "type": "RemoveMints", "data": { "request_id": "uuid", "underlying_mints": ["MintBase58"] } }
{ "type": "AddChannels", "data": { "request_id": "uuid", "channels": ["trades"] } }
{ "type": "RemoveChannels", "data": { "request_id": "uuid", "channels": ["stats"] } }
```

All four respond with `SubscriptionUpdated` containing the subscription state after the operation:

```json
{
  "type": "SubscriptionUpdated",
  "data": {
    "request_id": "uuid",
    "channels": ["markets", "trades"],
    "underlying_mints": ["MintBase58"],
    "quote_mints": null
  }
}
```

`underlying_mints` and `quote_mints` are `null` when unfiltered (all mints).

---

## Invite gating (closed mainnet)

During closed mainnet, takers need an invite code to trade. The server sends `RequireInvite` after auth if the taker is not registered.

### RequireInvite (server -> taker)

```json
{ "type": "RequireInvite" }
```

Unit variant, no `data` field. Sent once after `AuthSuccess` if the taker has not redeemed an invite.

### RedeemInvite (taker -> server)

```json
{
  "type": "RedeemInvite",
  "data": {
    "request_id": "uuid",
    "code": "abc123"
  }
}
```

The authenticated taker session proves wallet ownership; `RedeemInvite` does not carry a separate signature field.

### InviteRedeemed (server -> taker)

```json
{
  "type": "InviteRedeemed",
  "data": {
    "request_id": "uuid",
    "referral_code": "mycode"
  }
}
```

After redemption, the taker receives a personal `referral_code` they can share.

### ClaimReferralCode (taker -> server)

```json
{
  "type": "ClaimReferralCode",
  "data": {
    "request_id": "uuid",
    "code": "mypreferred"
  }
}
```

Claim a custom referral code (replaces the auto-assigned one).

### ReferralCodeClaimed (server -> taker)

```json
{
  "type": "ReferralCodeClaimed",
  "data": {
    "request_id": "uuid",
    "referral_code": "mypreferred"
  }
}
```

### GetMyReferralInfo (taker -> server)

```json
{ "type": "GetMyReferralInfo", "data": { "request_id": "uuid" } }
```

### MyReferralInfo (server -> taker)

```json
{
  "type": "MyReferralInfo",
  "data": {
    "request_id": "uuid",
    "referral_code": "mycode",
    "status": "active",
    "total_invited": 3,
    "invited_this_period": 1,
    "max_invites_per_period": 5,
    "next_slot_frees_in_seconds": 86400
  }
}
```

### Invite error types

Invite errors use typed `ServerError` variants (delivered as `RequestError` when `request_id` is present):

**`InviteRequired`** — sent as a typed error when an unregistered taker attempts a trading action.

**`Invite`** — `RedeemInvite` errors:

| Value | Meaning |
|-------|---------|
| `invalid_code` | Code does not exist |
| `code_exhausted` | Code has no remaining uses |
| `code_expired` | Code has expired |
| `code_disabled` | Code has been disabled |
| `code_owner_inactive` | Code owner is inactive |
| `code_owner_blacklisted` | Code owner is blacklisted |
| `already_registered` | Taker already registered |
| `internal_error` | Server error |

**`Claim`** — `ClaimReferralCode` errors:

| Value | Meaning |
|-------|---------|
| `not_registered` | Taker not registered (must redeem invite first) |
| `invalid_format` | Code format invalid |
| `code_taken` | Code already claimed by another user |
| `reserved` | Code is reserved |
| `internal_error` | Server error |

---

## Token caps

Platform-level open interest and notional caps. Useful for showing remaining capacity on the UI
(e.g. "X SOL available out of Y SOL limit" per market or underlying).
No authentication required.

### GetTokenCaps (taker -> server)

```json
{ "type": "GetTokenCaps", "data": { "request_id": "uuid" } }
```

### TokenCaps (server -> taker)

```typescript
type TokenCapsData = {
  request_id: string;          // echoed from request
  tokens: TokenCapInfo[];      // OI caps by underlying mint
  markets?: MarketCapInfo[];   // OI caps by market PDA, omitted when empty
  quotes?: QuoteCapInfo[];     // notional caps by quote mint, omitted when empty
};

type TokenCapInfo = {
  underlying_mint: string;     // base58
  symbol: string;              // e.g. "SOL"
  current_oi: number;          // current open interest (1e9 scale)
  max_oi: number;              // authorized limit (1e9 scale); 0 = no entry / unlimited
  utilization: number;         // current_oi / max_oi (0.0–1.0)
};

type MarketCapInfo = {
  market_id: string;           // market PDA base58
  current_oi: number;          // current OI for this market
  max_oi: number;              // authorized limit
  utilization: number;
};

type QuoteCapInfo = {
  quote_mint: string;          // base58
  symbol: string;              // e.g. "USDC"
  current_notional: number;    // sum of active position premiums (quote atomic units)
  max_notional: number;        // authorized limit (quote atomic units)
  utilization: number;
};
```

All amount fields are `u64` represented as JSON numbers. Scale is token-specific
(1e9 for underlying quantities, quote token atomic units for notional).

Only entries with a configured budget are returned. If a token or market has no budget, it will not appear in the response and the backend treats that scope as uncapped.

`GetTokenCaps.include_markets` is accepted by the wire type but currently ignored by the backend; configured market caps are returned when present.

---

## Indicative prices

### GetIndicativePrices (taker -> server)

```json
{
  "type": "GetIndicativePrices",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "position_type": "covered_call"
  }
}
```

### IndicativePrices (server -> taker)

```json
{
  "type": "IndicativePrices",
  "data": {
    "request_id": "uuid",
    "market": "MarketPdaBase58",
    "position_type": "covered_call",
    "updated_at": 1710000000,
    "is_stale": false,
    "strikes": [
      { "strike": 450000000000, "best_price": 5000000 },
      { "strike": 460000000000, "best_price": null }
    ]
  }
}
```

---

## Earn summary

Aggregated earn data per asset: APR ranges, capacity utilization, nearest market.
No authentication required.

### GetEarnSummary (taker -> server)

```json
{ "type": "GetEarnSummary", "data": { "request_id": "uuid" } }
```

### EarnSummary (server -> taker)

```json
{
  "type": "EarnSummary",
  "data": {
    "request_id": "uuid",
    "assets": [
      {
        "underlying_mint": "UnderlyingMintBase58",
        "underlying_symbol": "SOL",
        "quote_mint": "QuoteMintBase58",
        "quote_symbol": "USDC",
        "position_type": "covered_call",
        "min_apr": 5.2,
        "max_apr": 18.7,
        "cap_filled_pct": 0.45,
        "cap_total": 100000000000,
        "cap_used": 45000000000,
        "strikes_count": 3,
        "nearest_market_pda": "MarketPdaBase58",
        "markets_count": 2,
        "nearest_expiry_ts": 1710600000
      }
    ],
    "computed_at": 1710000000
  }
}
```

Field semantics:
- `min_apr` / `max_apr`: annualized percentage rate range across active strikes. `null` if no indicative prices available.
- `cap_filled_pct`: fraction of capacity used (0.0–1.0).
- `cap_total` / `cap_used`: underlying atomic units.
- `nearest_market_pda`: market PDA with the nearest expiry. Use this to navigate to the market page.
- `nearest_expiry_ts`: Unix seconds of the nearest market expiry.
- `computed_at`: Unix seconds when the summary was computed server-side.

---

## Authentication requirements

### Does not require auth

- `GetMarkets`
- `GetMarketDescriptors`
- `GetExpiries`
- `GetTokens`
- `GetTokenCaps`
- `GetIndicativePrices`
- `GetEarnSummary`
- `GetTokenMarketsInfo`
- `GetSubscriptions`
- `AddMints`, `RemoveMints`, `AddChannels`, `RemoveChannels`
- `ResumeAuth`, `StartAuth`, `AuthChallenge` (these *perform* auth)

### Requires auth

- `GetPositions`
- `GetMyActiveRfqs`
- `GetOrderStatus`
- `RfqRequest`, `AcceptQuote`, `CancelRfq`, `SubmitSignedSponsoredTx`
- `Subscribe`, `Unsubscribe`
- `RedeemInvite`, `ClaimReferralCode`, `GetMyReferralInfo`

### Requires auth + invite

- `RfqRequest` (server returns `InviteRequired` error if taker has not redeemed an invite)

---

## Errors

Two error message types: `Error` (connection-level) and `RequestError` (request-correlated).
See `ws-common.md` "Error format" for the envelope.

Parsing rule for taker integrations:

1. Handle `RequestError` where you pass `request_id` (e.g. `GetPositions`, `Subscribe`).
2. Handle `Error` for connection-level failures (auth, WS errors, unknown request).
3. For both, switch on `ServerError.type`.
4. If `type == "Generic"`, switch on `data.code`.
5. Keep fallback handling for unknown `Generic` codes.

Common typed `ServerError` variants (PascalCase on the wire):
- `RfqNotFound`, `RfqNotActive`
- `QuoteNotFound`, `QuoteExpired`
- `SignatureTimeout`
- `InvalidPositionType`, `InvalidMarket`
- `OracleNotReady`, `OraclePriceNotReady`, `OraclePriceStale`
- `MarketMetadataIncomplete`, `TokenMetadataIncomplete`
- `Cap` (position, notional, or balance cap exceeded)
- `RateLimit` (data is a plain string reason code)
- `KernelNotAvailable`
- `ServerShuttingDown`
- `Unauthenticated`, `Unauthorized`
- `InviteRequired` (taker not registered; must redeem invite)
- `Invite` (data is `InviteErrorReason` string, e.g. `"invalid_code"`)
- `Claim` (data is `ClaimErrorReason` string, e.g. `"code_taken"`)

Common `Generic` variant `code` values in taker flows:
- `missing_size_rule_for_underlying_mint` — no configured size rule for RFQ market underlying mint
- `invalid_quantity_size_rule` — RFQ `quantity` violates `min/max/step` constraint
- `trading_paused` — backend or on-chain pause currently blocks new trading actions
- `session_expired` — `ResumeAuth` with invalid/expired/revoked session
- `already_authenticated` — `ResumeAuth` on an already authenticated connection
- `hello_required`, `hello_timeout`, `hello_already_sent`
- `parse_error`, `too_many_parse_errors`, `message_too_large`
- `internal_error`

### RateLimitReason (in typed `RateLimit` error)

| Value | Meaning |
|-------|---------|
| `too_many_active_rfqs_total` | Platform-wide RFQ limit reached |
| `too_many_active_rfqs_per_taker` | Per-taker RFQ limit reached |
| `too_many_sessions_per_user` | Too many concurrent sessions |

---

## Related

- [Taker quickstart](../quickstart/taker-quickstart.md) — narrative flow + raw sponsored-tx signing
- [Taker wire examples](../quickstart/taker-wire-examples.md) — complete JSON session + branch scenarios
- [Taker TS SDK quickstart](../quickstart/web-client-ts-sdk.md)
- [Protocol flow](protocol-flow.md) — trade lifecycle, economics, settlement, risk
- [WS common conventions](ws-common.md) — units, envelopes, timeouts
- [Capacity limits](caps.md) — OI and notional caps
- [Sandbox / Devnet](sandbox.md) — endpoints, faucets, program addresses
- [FAQ](faq.md)
