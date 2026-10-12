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
- max message size: `32 KiB`
- max connections per IP: `50`

Rate limits (token buckets, per connection unless stated):

| Bucket | Sustained | Burst | Covers | On breach |
|---|---|---|---|---|
| Message | `30 msg/s` | `60` | every client message | connection closed |
| Wallet | `50 msg/s` | `100` | all connections of the same wallet | connection closed |
| RFQ | `10 msg/s` | `20` | `RfqRequest`, `AcceptQuote`, `CancelRfq` | soft `rate_limited`, connection stays open |
| Query | `20 msg/s` | `40` | discovery, position and subscription messages | soft `rate_limited`, connection stays open |

Every breach is reported with `generic.code = "rate_limited"`; only the message and wallet buckets
close the connection. Retry with exponential backoff and jitter.

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
The default allows three failed attempts; the fourth triggers `too_many_auth_attempts` and closes the connection.

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
    "challenge": "Acta RFQ Authentication\n\nSign this message to authenticate your wallet.\n\nWallet: WalletPubkeyBase58\nNonce: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\nIssued At: 2026-09-15T00:00:00Z\n",
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

`expires_at` is a required Unix timestamp (seconds) for both takers and makers. It is the resume credential deadline, not the lifetime of the authenticated socket. Makers use a shorter TTL, and makers can `ResumeAuth` too. The official maker SDK tries the cached session first and falls back to a fresh signed challenge only when the server reports `session_expired`.
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

`underlying_mints` filters by underlying mint and `quote_mints` by quote mint.
When either field is present, `Subscribe` replaces both scopes together; an
omitted peer field becomes the all-mints wildcard. If both are omitted, existing
scopes are unchanged. Use `AddMints`/`RemoveMints` for incremental updates.

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
        "is_put": false,
        "status": "active"
      }
    ]
  }
}
```

`Snapshot.markets` lists the active markets as compact [`MarketInfo`](ws-common.md#marketinfo) (`pda`, `underlying`, `quote`, `expiry_ts`, `is_put`, `status`). This is a different DTO from the `MarketDescriptor` used in `RfqCreated.order_options` and discovery responses. Use `GetMarketDescriptors` for the full market descriptor with `chain_id`, `program_id`, `collateral_mint`, `settlement_mint`, etc.

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
- `Snapshot`, `ReferenceChanged`
- `RfqCreated`, `QuoteReceived`, `QuotesUpdate`
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

`GetPositions.price` is returned only from the maker-signed gross price in the
canonical `OpenPosition` epoch. A GPA-only net-premium observation is not
converted into a synthetic gross price; incomplete rows fail closed with
`gross_price_proof` missing.
Snapshot-to-epoch binding additionally requires finalized audit coverage
through the observation slot; a later audit-gap retraction fails the read closed.

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
    "valid_until": 1710000350,
    "nonce": 42,
    "order_id": "0x...64chars"
  }
}
```

- `price` — gross quote from the maker. Hash-bound via `order_id` — use this for `AcceptQuote` and all order operations.
- `net_price` — display-only net price estimate after protocol fee deduction. Present when the server has loaded the on-chain fee config. The contract's authoritative fee is computed at `OpenPosition` as `min(premium_fee, volume_fee)` after token-decimal scaling. Use `net_price` for UI display and `price` for order operations.

### QuotesUpdate (server -> taker)

The server sends the current executable quote snapshot directly to the owner
of the RFQ whenever its quote set changes. An empty `quotes` array means no
executable quotes remain.

```json
{
  "type": "QuotesUpdate",
  "data": {
    "rfq_id": "uuid",
    "quotes": [
      {
        "rfq_id": "uuid",
        "strike": 160000000000,
        "maker": "MakerOwnerPubkeyBase58",
        "price": 50000000,
        "net_price": 49750000,
        "valid_until": 1710000350,
        "nonce": 42,
        "order_id": "0x...64chars"
      }
    ]
  }
}
```

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

Sent when `SubmitSignedSponsoredTx` is accepted into Core's `Enqueued` state; it is not chain confirmation. An exact `AcceptQuote` retry on a locked order may also receive this liveness ACK while its signing payload is being built, or after enqueueing. Initial `AcceptQuote` locks the quote and starts building `SponsoredTxToSign`.

```json
{
  "type": "OrderAccepted",
  "data": {
    "order_id": "0x...64chars",
    "order_version": 1
  }
}
```

### OrderStatus (server -> taker)

`GetOrderStatus` is an owner-scoped execution lookup, available to authenticated takers and makers. It returns the strongest currently available execution evidence for the supplied `order_id`:

```json
{
  "type": "OrderStatus",
  "data": {
    "request_id": "uuid",
    "order_id": "0x...64chars",
    "state": { "type": "confirmed", "position_pda": "PositionPdaBase58" }
  }
}
```

| `state` | Meaning |
|---|---|
| `{ "type": "pending" }` | The venue still has a pending execution obligation. |
| `{ "type": "confirmed", "position_pda": "..." }` | Execution is confirmed; this is the resulting position. |
| `{ "type": "unknown" }` | Available evidence cannot establish the outcome. This is not proof of nonexecution. |

The response has no `status`, `order_version`, `rfq_id` or `tx_signature` fields. Lookup failures return a correlated `RequestError`. Neither an empty position list nor `unknown` authorizes replaying the order.

`order_version` remains on lifecycle pushes: accepted `1`, submitted `2`, failed `3`, confirmed `4`. A later chain confirmation can supersede a local failure.

### OrderSubmitted (server -> taker)

```json
{
  "type": "OrderSubmitted",
  "data": {
    "order_id": "0x...64chars",
    "tx_signature": "5eyk...base58sig",
    "order_version": 2
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
    "order_version": 4
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
    "order_version": 3
  }
}
```

`reason` is a free-form diagnostic string, not an enum. Log it, show it, but never branch on an
exact match — the wording changes with the underlying error. Observed shapes include
`on_chain: <program error>`, `blockhash_expired_before_send`,
`sponsored transaction rejected: <detail>`, `order_expired` and `order_failed`.

No `OrderFailed` reason authorizes a client-side replay. Treat every one as non-retryable and
reconcile execution instead.

Keeper retries retryable submission failures within a five-attempt budget. Exhausting it does not prove nonexecution or guarantee a WS `OrderFailed` or `RfqAvailableAgain`. An uncertain keeper failure leaves Core `Enqueued`; only a locally proven-unforwarded failure can reopen or close the RFQ. Follow `RfqAvailableAgain` / `RfqClosed`, and query `GetOrderStatus` for unresolved execution. Rollback discards the winner; refresh available quotes instead of replaying that order.

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

## Delivery and recovery

Delivery is **best-effort, with loss surfaced as a disconnect**. Everything a taker receives except `StatsUpdate` is critical: your owner-direct pushes (`RfqCreated`, `QuoteReceived`, `QuotesUpdate`, `RfqClosed`, `TradeExecuted`, and the order lifecycle `OrderAccepted`/`SponsoredTxToSign`/`OrderSubmitted`/`OrderConfirmed`/`OrderFailed`) and subscribed broadcasts (`ChainEvent`, market events). Each is delivered reliably or — if it cannot be delivered under backpressure — dropped, after which the server **closes the connection**. A lost critical message therefore always surfaces as a disconnect — never silent staleness. Only `StatsUpdate` may drop silently. (`PositionUpdated` is maker-only and never sent to takers.)

After `AuthSuccess`, re-read `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus` for unresolved orders. These reads have different authorities: live RFQs come from Core; positions are a chain-derived database projection; execution lookup combines pending obligations and execution evidence. They are not one atomic snapshot, and absence from a single response does not prove nonexecution.

A successful `ResumeAuth` completes Core handoff before `AuthSuccess`: ownership transfers from the previous physical connection of that same auth session. A fresh credential for the same wallet does not perform that transfer.

| Recovered state | Client action |
|---|---|
| Same credential, same `rfq_id` and `locked_order_id`, `pending_signature`, signature not sent | Repeat the exact `AcceptQuote` to retrieve the existing signing payload. While building it, the server may return `OrderAccepted`. Keep the original signature deadline. |
| `enqueued`, or `SubmitSignedSponsoredTx` already sent | Reconcile execution with `GetOrderStatus`; do not resubmit or sign again automatically. |
| `unknown`, missing RFQ/position, timeout or failed lookup | Keep the outcome unresolved. Missing evidence does not close the obligation. |

Preserve selected order identity and whether the signature was sent across reconnect. Ignore wallet results that belong to an older connection or selection. An uncorrelated `SignatureTimeout` is not enough to identify the affected RFQ; use the matching `RfqAvailableAgain` / `RfqClosed` or re-read its state.

There is no stream replay cursor. Reconcile entity versions on lifecycle pushes, and apply owner-scoped reads after reconnect. `OrderConfirmed` establishes execution; `RfqClosed` closes the auction, not every possible unresolved execution record.

---

## Discovery

Current taker discovery requests:

```json
{ "type": "GetMarkets", "data": { "request_id": "uuid" } }
{ "type": "GetMarketDescriptors", "data": { "request_id": "uuid" } }
{ "type": "GetExpiries", "data": { "request_id": "uuid" } }
{ "type": "GetTokens", "data": { "request_id": "uuid" } }
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

`GetMarkets`, `GetMarketDescriptors`, `GetExpiries` and `GetTokens` answer from the server's
[reference snapshot](ws-common.md#reference-data): **tradable** markets only — non-finalized,
non-disabled, and before the pre-expiry trading cutoff. There is no `active_only` field; the server
ignores it if sent. Closed and finalized markets are on HTTP `GET /api/v1/markets?active=false`.
Re-read on [`ReferenceChanged`](ws-common.md#referencechanged).

### Discovery strict policy

- Every underlying of an active market needs a configured size rule. If one is missing,
  `GetMarketDescriptors` and `GetTokens` fail as a whole with `missing_size_rule_for_underlying_mint`.
- `GetMarketDescriptors` also fails as a whole (`MarketMetadataIncomplete`) when an active market has
  no oracle PDA. Partial discovery is not supported.

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

For cash-secured puts, the size_rule still constrains the underlying quantity on the wire, not the quote deposit amount. Frontends that collect user input in quote terms (e.g. USDC) must convert before sending `RfqRequest`. See [WS common conventions](ws-common.md) "Quantity and collateral by position type" for conversion formulas and SDK helpers.

`underlying_symbol`, `quote_symbol`, and token `symbol` are mandatory and non-null in discovery payloads.

### Tokens payload

```jsonc
{ "type": "Tokens", "data": { "request_id": "uuid", "tokens": [ /* TokenInfo */ ] } }
```

`tokens` lists each mint once; see [`TokenInfo`](ws-common.md#tokeninfo). Underlyings carry
`size_rule`, `strike_tick` and `quote_mints`; quote tokens do not. The same objects are served by
HTTP `GET /api/v1/tokens`.

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
        "expiry_ts": 1710600000
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
    "underlying_mints": ["MintBase58"]
  }
}
```

In subscription responses, an omitted mint list means that dimension is unfiltered. In requests, omission/null preserves the current filter; an explicit empty list clears it.

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
  current_oi: number;          // current open interest (underlying atomic units)
  max_oi: number;              // limit in underlying atomic units; 0 = configured zero capacity
  utilization: number;         // current_oi / max_oi (not clamped to 1.0)
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
  current_notional: number;    // exposure plus live reservations (quote atomic units)
  max_notional: number;        // authorized limit (quote atomic units)
  utilization: number;
};
```

All amount fields are `u64` represented as JSON numbers. Scale is token-specific
(underlying atomic units for OI, quote token atomic units for notional).

Only entries with a configured budget are returned. If a token or market has no budget, it will not appear in the response and the backend treats that scope as uncapped.

`GetTokenCaps.include_markets` is accepted but ignored; configured market caps are returned when present.

---

## TokenMarketsInfo

`GetTokenMarketsInfo { request_id, underlying_mint }` is available after `Welcome` without authentication. It returns one market-page snapshot from the selected backend:

```json
{
  "type": "TokenMarketsInfo",
  "data": {
    "request_id": "uuid",
    "underlying_symbol": "SOL",
    "underlying_decimals": 9,
    "quote_symbol": "USDC",
    "quote_decimals": 6,
    "size_rule": { "min_size": 100000000, "max_size": 10000000000, "step": 100000000 },
    "reference_price": 150000000000,
    "markets": [{
      "market_pda": "MarketPdaBase58",
      "expiry_ts": 1710600000,
      "is_put": false,
      "indicatives": [{
        "position_type": "covered_call",
        "updated_at": 1710000000,
        "is_stale": false,
        "strikes": [{ "strike": 160000000000, "best_price": 50000000 }]
      }]
    }]
  }
}
```

`reference_price` is the backend-validated underlying/quote spot, scaled by `1e9`. `best_price` is a non-binding indicative display premium per underlying unit, also scaled by `1e9`. The backend applies the configured quote-mint fee adjustment before returning it; if fee configuration is unavailable, it leaves the original indicative price unchanged. This is an estimate, not the exact on-chain net premium, and it may be absent. Size-rule quantities are underlying atomic units. Examples illustrate units, not current prices or limits.

The response does not echo `underlying_mint`: retain it with the request ID. Use spot and premiums from this same response for an APR estimate, and suppress the estimate when the indicative is stale or missing. `updated_at` / `is_stale` describe the maker's indicative premiums, not the oracle spot. The response contains no oracle publish timestamp, confidence or age.

The backend owns Pyth/Hermes access and price validation. A browser does not need a Pyth key. This endpoint supplies a current snapshot, not historical chart data or a binding execution quote.

**Failure behavior:** missing size rules, incomplete metadata or unavailable oracle prices can return uncorrelated `Error` (including `OraclePriceNotReady`). When there are no active markets for the underlying, the handler sends no response. Apply a request deadline, show unavailable data, and discard stale responses by request ID. Do not interpret silence as a successful empty snapshot. This endpoint is an exception to correlated query-error handling.

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
- `cap_filled_pct`: fraction of capacity used, not clamped to 1.0. It may exceed 1.0 after limits are tightened; a zero limit reports saturated utilization 1.0. See [caps](caps.md#monitoring-caps).
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
See [WS common conventions](ws-common.md) "Error format" for the envelope.

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
