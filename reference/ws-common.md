# Acta WS Common Conventions

## Message envelope

All WS messages use:

```json
{ "type": "MessageType", "data": { ... } }
```

Unit variants may omit `data`, for example:

```json
{ "type": "Logout" }
```

## Encodings and units

| Field | Format |
|-------|--------|
| Pubkeys | base58 |
| Signatures | base58 (64 bytes) |
| `order_id` | 64-char hex (optional `0x` prefix) |
| `price` | `u64`, gross premium per 1 underlying unit, 1e9 scale |
| `total_premium` | `u64`, net premium amount in quote token atomic units |
| `strike` | `u64`, quote per 1 underlying unit, 1e9 scale |
| `quantity` | `u64`, underlying atomic units |
| `size_rule` | min_size, max_size, step in underlying atomic units; always tied to underlying_mint |
| `strike_tick` | `u64`, strike ladder spacing for an underlying, 1e9 scale |
| Timestamps | Unix seconds |
| `server_time_unix_ms` | Unix milliseconds |

`price` and `strike` are always 1e9 fixed-point per 1 underlying unit, independent of mint decimals.


## Reference data

Tokens and markets come from one server snapshot, shared by WS (`GetTokens`, `GetMarkets`,
`GetMarketDescriptors`, `GetExpiries`, `Snapshot.markets`) and HTTP (`/api/v1/tokens`,
`/api/v1/markets`, see [HTTP API](http-api.md#reference-data)). WS discovery always answers from the
active set: markets that are not finalized, not disabled and before their trading cutoff. There is no
`active_only` field; the server ignores it if sent.
For closed and finalized markets use `GET /api/v1/markets?active=false`.

### TokenInfo

One object per mint, on both transports. Underlying-only fields are omitted on quote tokens.

```typescript
type TokenInfo = {
  mint: string;
  symbol: string;
  decimals: number;
  oracle_feed_ids?: string[];      // Pyth feed ids, hex without 0x; omitted when none is known
  size_rule?: {                    // underlyings only; underlying atomic units
    min_size: number;
    max_size: number;
    step: number;
  };
  strike_tick?: number;            // underlyings only; strike ladder spacing, 1e9 scale
  quote_mints?: string[];          // underlyings only; quote mints of its active markets
};
```

A valid `quantity` satisfies `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`.
Only tokens of active markets are listed.

### MarketInfo

Used by `Snapshot.markets`, `Markets` and HTTP `/api/v1/markets`. Markets name tokens by mint.

```typescript
type MarketInfo = {
  pda: string;
  underlying: string;        // underlying mint
  quote: string;             // quote mint
  expiry_ts: number;
  is_put: boolean;
  status: "active" | "closed" | "finalized";
  settlement_price?: number; // 1e9 scale, finalized markets only
};
```

WS responses (`Snapshot`, `Markets`) contain only `active` markets.

### ReferenceChanged

Pushed to every authenticated maker (quote and `/maker/data`), taker and vault operator session, without a subscription, when the token list or
the market list changes, including a market crossing its trading cutoff or expiry. The server checks
for changes every second.

```json
{
  "type": "ReferenceChanged",
  "data": { "tokens_etag": "\"3f2a...\"", "markets_etag": "\"9b1c...\"" }
}
```

The ETags are the HTTP `ETag` values of `/api/v1/tokens` and of unfiltered
`/api/v1/markets?active=false`. On receipt, re-read what you cache (`GetTokens` /
`GetMarketDescriptors` on WS, or the HTTP routes with `If-None-Match`). Vault sessions on `/vault`
re-read the HTTP routes.

### VaultCatalogChanged

Sent on the `vaults` channel to maker and taker sessions after a vault is created, re-delegated
(`RotateVaultDelegate`) or closed. Disabling a delegate (`DisableVaultDelegate`) does not send it.
It carries only the vault address; re-list vaults (`GET /api/v1/vaults`) to pick up the change.
`/vault` operator sessions do not receive it.

It is a hint, not a feed. It is never replayed: if your outbound queue is full the message is
dropped and the server closes the connection, and nothing sent while you were disconnected is
redelivered. Re-list vaults after every connect and reconnect as well as on this message.

```json
{ "type": "VaultCatalogChanged", "data": { "vault_pda": "VaultPdaBase58" } }
```

### VaultProjectionAdvanced

Sent on the `vaults` channel after the server's projection of a vault advances by a committed
on-chain vault event. `/maker` quote sessions and taker sessions subscribed to `vaults` receive it for every vault; a
`/vault` operator session receives it only for its own vault. It is a change notification, not the
new state: reload `GET /api/v1/vaults/{pda}`. `version` increases with each projected vault event; if
you see a gap, or after a reconnect, reload the vault rather than repairing local state. Delivery
follows the same drop-and-disconnect rule as `VaultCatalogChanged`.

```json
{
  "type": "VaultProjectionAdvanced",
  "data": {
    "vault_pda": "VaultPdaBase58",
    "version": 42,
    "slot": 312345678,
    "transaction_index": 3,
    "signature": "TxSignatureBase58",
    "event_index": 1
  }
}
```

`slot`, `transaction_index`, `signature` and `event_index` locate the vault event that produced
`version`.

## Quantity and collateral by position type

`quantity` on the wire is always in underlying atomic units, regardless of position type.
The two position types differ in what the user deposits as collateral:

**Covered call** — user deposits underlying (e.g. SOL).
`quantity` maps directly to the deposit: `quantity = userInput * 10^underlying_decimals`.

**Cash-secured put** — user deposits quote (e.g. USDC).
The collateral relationship is: `collateral = quantity * strike / 1e9`.
Convert quote input to underlying quantity before sending:

```
quantity = round(quoteAmount * 10^underlying_decimals * 1e9 / strike)
```

And to display the deposit back in quote terms:

```
quoteAmount = quantity * strike / 1e9 / 10^underlying_decimals
```

`underlying_decimals` comes from `MarketDescriptorInfo`.

### Size rule display for CSP

`size_rule` values (min_size, max_size, step) are in underlying atomic units.
To show constraints in quote terms for a CSP UI, apply the same conversion:

```
min_quote = min_size * strike / 1e9 / 10^underlying_decimals
max_quote = max_size * strike / 1e9 / 10^underlying_decimals
step_quote = step    * strike / 1e9 / 10^underlying_decimals
```

Example: SOL/USDC CSP, strike $90 (90e9), `size_rule = {min: 1e9, max: 10e9, step: 1e8}`:

| Wire (SOL lamports) | Display (USDC) |
|---------------------|----------------|
| min_size = 1,000,000,000 | 90 USDC |
| max_size = 10,000,000,000 | 900 USDC |
| step = 100,000,000 | 9 USDC |

The TS SDK provides helpers for these conversions: `quoteAmountToQuantity`,
`quantityToQuoteAmount`, `sizeRuleInQuoteTerms` (exported from `@acta-markets/ts-sdk/ws`).

## Time and clock skew

`server_time_unix_ms` is present in:
- `Welcome.server_time_unix_ms` (required)
- `Pong.server_time_unix_ms` (required)

`Snapshot` does not include `server_time_unix_ms` in current protocol.

Client estimate:

```
offset_ms = server_time_unix_ms - local_time_ms
estimated_server_now_ms = local_time_ms + offset_ms
```

Use this estimate for `expires_at`, `valid_until`, and `signature_deadline`.

## Authentication protocol

### Signing algorithm

Ed25519. Signatures are 64 bytes, base58-encoded on the wire.

### Challenge format

After the client sends `Hello` and receives `Welcome`, the server sends `AuthRequest` with a `challenge` field. The challenge is a multiline plaintext string. The format differs by role:

**Taker challenge** (includes `Wallet:` line):

```
Acta RFQ Authentication

Sign this message to authenticate your wallet.

Wallet: {base58_pubkey}
Nonce: {hex_encoded_32_random_bytes}
Issued At: {RFC3339_timestamp}
```

**Maker challenge** (no `Wallet:` line):

```
Acta RFQ Authentication

Sign this message to authenticate your wallet.

Nonce: {hex_encoded_32_random_bytes}
Issued At: {RFC3339_timestamp}
```

Makers do not need a `Wallet:` line. The server identifies the maker from `AuthChallenge.pubkey`, then looks up the registered signing key from the on-chain maker registry.

**Vault operator challenge** (`/vault` sessions): the taker challenge for the operator key, followed by `\nVault: {vault_pda}` with **no** trailing newline. Because the taker challenge already ends in `\n`, a blank line precedes the `Vault:` line:

```
Acta RFQ Authentication

Sign this message to authenticate your wallet.

Wallet: {operator_pubkey}
Nonce: {hex_encoded_32_random_bytes}
Issued At: {RFC3339_timestamp}

Vault: {vault_pda}
```

### What to sign

Validate the entire challenge **before invoking the signer**. Require the exact text and blank lines above, a nonce of exactly 64 lowercase hex characters, a valid UTC timestamp in `YYYY-MM-DDTHH:MM:SSZ` format, and a final `\n` after that timestamp. A maker challenge must have no `Wallet:` line. A taker challenge must include that line and it must match the wallet being authenticated. A vault challenge must be a valid taker challenge for the operator key followed by exactly `\nVault: {vault_pda}` for the vault you requested, and nothing after it. Reject anything that is not one of these three formats for your role: extra lines, alternate domains, missing final newlines and arbitrary 32-byte strings.

After validation, sign the original UTF-8 bytes without normalization, additional prefix or hashing. The fixed text already supplies the authentication domain. Signing arbitrary endpoint text with the order key can authorize an order instead of authentication.

Rust SDK (`acta-maker-sdk` 0.5.2) and TS SDK (`@acta-markets/ts-sdk` 0.2.2) validate authentication messages before invoking signers in their managed clients. Raw Rust clients use `validate_maker_auth_challenge`, or `validate_vault_auth_challenge(challenge, operator, vault)` for a vault session; raw TS clients use `validateAuthChallenge`, which checks the grammar of the maker and taker formats and rejects the vault format, and `assertVaultAuthChallenge(challenge, operator, vault)` for a vault session. This does not establish endpoint trust or prevent relay of a genuine server challenge.

### AuthChallenge (client -> server)

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "Acta RFQ Authentication\n\nSign this message...",
    "signature": "3q7uQqYc3...base58sig",
    "pubkey": "PubkeyBase58"
  }
}
```

**Maker**: `pubkey` is the `maker_owner` pubkey registered on-chain. Sign the challenge with the maker's registered signing key, which is also used for quotes. The server looks up `maker_owner` and verifies the authentication signature against that key.

**Taker**: `pubkey` is the taker's wallet pubkey (same as the `Wallet:` value in the challenge). The signature is verified directly against this key.

### Auth constraints

- Max auth attempts per connection: `3`. The 4th attempt closes the connection.
- Auth deadline after challenge issued: `15s`.

### Ping / Pong

```json
{ "type": "Ping" }
```

`Ping` is a unit variant (no `data` field). Server responds with:

```json
{
  "type": "Pong",
  "data": {
    "server_time_unix_ms": 1710000000000
  }
}
```

Clients should send `Ping` every ~30s to avoid the 90s idle timeout.

### Connection close codes

When the server closes a connection itself, the close frame carries one of these codes, with the
name as the close reason. An `Error` or `AuthError` frame naming the cause usually precedes it.
Other closes (blacklist, expired session, too many parse or auth errors, shutdown) send a close
frame without a code; the preceding error frame says why.

| Code | Reason | Cause |
|---|---|---|
| `4000` | `idle_timeout` | No traffic within the idle timeout |
| `4001` | `auth_timeout` | Authentication not completed within the auth deadline |
| `4002` | `slow_consumer` | A critical message could not be queued for you, or a socket write to you timed out |
| `4003` | `rate_limited` | Connection or wallet message rate exceeded, or you sent faster than the server could process (inbound queue full). Quote-plane messages never trip the message rate; see the quote bucket |
| `4004` | `session_replaced` | A newer connection authenticated as the same maker or taker session |
| `4005` | `hello_timeout` | `Hello` not received in time |

A rejected WebSocket upgrade returns an HTTP status with the same JSON body as the HTTP API,
`{"error": "...", "code": "..."}`: `403 origin_not_allowed`, `429 rate_limited` (too many
connections from your IP), `503 too_many_connections`, `503 server_overloaded`,
`503 server_shutting_down`.

### Logout

```json
{ "type": "Logout" }
```

Unit variant, no `data` field. Server responds with `LogoutSuccess` and closes the connection.

```json
{ "type": "LogoutSuccess", "data": {} }
```

## Fee model

The protocol charges a fee per trade, configured per quote mint in basis points (bps). The on-chain config stores two fee rates: `protocol_fee_bps_premium` and `protocol_fee_bps_volume`.

- `price` in the `Quote` message is the **gross premium per 1 underlying unit** (1e9 scale). This is what the maker signs and what goes into the order_id preimage as `gross_price`.
- On chain, gross premium is first scaled to quote-token atomic units. The premium-side fee is computed from `protocol_fee_bps_premium`; the volume-side cap is computed from strike notional and `protocol_fee_bps_volume`; the charged fee is `min(premium_fee, volume_fee)`.
- `net_price` in WebSocket payloads is a display approximation. The authoritative amount is the on-chain net premium after the min(premium-fee, volume-fee) calculation and token-decimal scaling.
- `total_premium` in position data is the net premium from on-chain state (in quote token atomic units).
- Makers always quote gross. The fee does not affect what the maker signs; it is applied by the contract at position open.

## Auth messages

### AuthSuccess

```json
{
  "type": "AuthSuccess",
  "data": {
    "session_id": "uuid",
    "expires_at": 1710086400,
    "maker_pda": "MakerPdaBase58"
  }
}
```

`expires_at` wire contract:
- Required JSON integer in Unix seconds, for makers and takers. This is the resume credential deadline, not the lifetime of the authenticated socket.
- Used for session resume by both makers and takers.

`maker_pda` wire contract:
- Set to the on-chain maker account PDA (base58) for authenticated **makers** that are registered on-chain.
- `null` (and omitted from the wire on serialization) for non-makers (admins, takers) and for makers not yet registered on-chain.

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

`reason` is a `snake_case` code from the [WS error code catalog](#ws-error-codes) (frames
marked `AuthError`). `message` is optional and may be omitted.

## Error format

Two error message types exist:

### Error — connection-level errors

Sent when no `request_id` is available (the message had no `request_id` field, or the error
occurred before routing):

```json
{ "type": "Error", "data": { "type": "RfqNotFound" } }
```

`data` is a tagged `ServerError` variant (`type` + optional nested `data`).

Endpoint-policy violations use a typed error rather than closing the socket:

```json
{
  "type": "Error",
  "data": {
    "type": "WrongEndpoint",
    "data": {
      "endpoint": "maker_data",
      "allowed_endpoints": ["maker"]
    }
  }
}
```

Endpoint values are `maker`, `maker_data`, and `taker`.

### RequestError — request-correlated errors

Sent when the server can correlate the failure to a specific client request. The `request_id`
echoes the client's `request_id` from the originating message:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "uuid",
    "error": { "type": "RfqNotFound" }
  }
}
```

The inner `error` has the same `ServerError` shape as `Error.data`.

Failures of `SubmitSignedSponsoredTx` and `SwapSubmit` also carry `delivery`:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "uuid",
    "error": { "type": "Generic", "data": { "code": "vault_swap_quote_expired", "message": "Vault swap quote expired" } },
    "delivery": "not_forwarded"
  }
}
```

`delivery` describes this one request only:

- `not_forwarded`: this request was rejected before it was handed to the keeper (routing gates,
  validation, a pre-enqueue rejection).
- `unknown`: this request may have reached the keeper pipeline (timeouts, queue errors, Core
  timeout, `order_already_submitted`). Resolve it with `GetOrderStatus` (orders) or
  `GetSwapStatus` / `SwapSettled` (Vault swaps).

It says nothing about earlier attempts with the same signed transaction. Treat an order or swap as
not executed only if every attempt you made returned `not_forwarded`; if any attempt timed out,
was lost with the connection, or returned `unknown`, resolve it by status before re-signing.

Prefer `RequestError` handlers over `Error` handlers wherever you pass `request_id` in requests.
Both events fire `error` in the TypeScript SDK (see [Web client SDK](../quickstart/web-client-ts-sdk.md)).

### Parsing rule (applies to both `Error.data` and `RequestError.data.error`)

1. Branch on `type`.
2. If `type != "Generic"`, treat it as a typed variant.
3. If `type == "Generic"`, branch on `data.code`.

Generic code example:

```json
{
  "type": "Error",
  "data": {
    "type": "Generic",
    "data": { "code": "parse_error", "message": "invalid payload" }
  }
}
```

Typed variant examples:

```json
{ "type": "Error", "data": { "type": "RfqNotFound" } }
```

```json
{
  "type": "Error",
  "data": {
    "type": "Unauthenticated",
    "data": { "action": "submit_quotes" }
  }
}
```

Generic codes and `AuthError` reasons are listed in the [WS error code catalog](#ws-error-codes).

JavaScript integrators must preserve integer precision: a wire `u64` can exceed `Number.MAX_SAFE_INTEGER`. TS SDK represents most WS amounts as `number`; it reports `unsafe_integer` for oversized amount literals but still dispatches the message. Do not use such rounded values for signing or accounting. Inbound nonces have separate lossless handling. For full-range raw JSON amounts, use lossless parsing and bigint arithmetic; `BigInt(JSON.parse(...).amount)` cannot repair precision already lost. This does not change the wire encoding or the `1e9` scale.

## WS error codes

The closed set of `Generic.code` values (in `Error.data` and `RequestError.data.error`) and
`AuthError.reason` values. The server sends nothing outside this table; typed `ServerError`
variants (`RfqNotFound`, `Unauthenticated`, `WrongEndpoint`, …) are not codes. Keep a fallback
branch for codes added later.

| Code | Frames | Meaning |
|---|---|---|
| `parse_error` | Generic | The frame is not valid JSON or not a known message |
| `too_many_parse_errors` | Generic | Consecutive parse errors exhausted the budget; the connection closes |
| `hello_required` | Generic | The first message was not `Hello`; the connection closes |
| `hello_timeout` | Generic | No `Hello` in time; followed by close `4005` |
| `hello_already_sent` | Generic | A second `Hello` on one connection |
| `rate_limited` | Generic | Connection or wallet rate limits close with `4003`; a per-category limit rejects only the message |
| `server_overloaded` | Generic | The admission queue for this message class is full; retry with backoff. A full connection command queue sends it uncorrelated and closes with `4003` |
| `server_busy` | Generic, AuthError | A bounded internal queue (auth, RFQ creation) is full; retry |
| `internal_error` | Generic | Unexpected server failure; the message is sanitized |
| `temporarily_unavailable` | Generic, AuthError | A dependency (database, Core, registry) is not ready; retry with backoff |
| `clock_error` | Generic, AuthError | The server clock is before the Unix epoch |
| `trading_paused` | Generic | Venue trading is paused; order-path messages are rejected |
| `batch_quotes_too_large` | Generic | `BatchQuotes` exceeds `max_batch_quotes` |
| `session_replaced` | Generic | A newer connection took over this session; do not reconnect automatically |
| `session_expired` | Generic, AuthError | The session or resume credential expired; authenticate again |
| `session_limit_exceeded` | AuthError | Too many concurrent sessions for this identity |
| `session_conflict` | Generic | Core no longer binds this session to the identity |
| `blacklisted` | Generic, AuthError | The identity is blacklisted; terminal |
| `maker_unregistered` | Generic | The maker left the registry while connected; the connection closes |
| `maker_not_registered` | Generic, AuthError | The pubkey is not a registered maker |
| `auth_timeout` | AuthError | Authentication did not finish in time; followed by close `4001` |
| `too_many_auth_attempts` | AuthError | Auth attempt budget exhausted; the connection closes |
| `already_authenticated` | AuthError | Auth started on an authenticated connection |
| `challenge_not_issued` | AuthError | `AuthChallenge` without a preceding `AuthRequest` |
| `challenge_mismatch` | AuthError | The signed challenge differs from the issued one |
| `invalid_pubkey` | Generic, AuthError | A pubkey field is not valid base58 |
| `pubkey_mismatch` | AuthError | `AuthChallenge.pubkey` differs from the `StartAuth` pubkey |
| `invalid_signature` | AuthError | The challenge signature does not verify |
| `invalid_operator` | AuthError | The Vault operator is not a valid pubkey |
| `operator_mismatch` | AuthError | The Vault challenge was signed by another operator |
| `invalid_vault` | Generic, AuthError | `vault_pda` is not a valid address |
| `vault_auth_not_started` | AuthError | Vault challenge answered without `StartVaultAuth` |
| `vault_auth_required` | Generic | A Vault message on a session without Vault authority |
| `vault_resume_not_supported` | AuthError | `ResumeAuth` on `/vault` |
| `vault_operator_unauthorized` | AuthError | The operator holds no current grant for the Vault |
| `vault_authorization_unavailable` | AuthError | Vault authority could not be read; retry |
| `vault_session_invalidated` | Generic | The Vault grant changed or was revoked; the session closes |
| `invalid_vault_subscription` | Generic | The subscription is not allowed for a Vault session |
| `subscription_scope_too_large` | Generic | The subscription mint filter exceeds its limit |
| `history_projection_incomplete` | Generic | A history read lacks listener metadata; retrying does not help until reconciliation |
| `invalid_position_status` | Generic | Unknown position status filter |
| `invalid_cursor` | Generic | Malformed pagination cursor |
| `invalid_query_scope` | Generic | `limit` or `cursor` sent with `scope: live` |
| `market_not_found` | Generic | The market is not in the registry |
| `invalid_program_id` | Generic | The server program id is misconfigured |
| `invalid_strike_notional` | Generic | Strike × quantity is zero or not representable on chain |
| `invalid_quantity_size_rule` | Generic | The quantity is outside the size rule |
| `missing_size_rule_for_underlying_mint` | Generic | No size rule is configured for the underlying |
| `invalid_underlying_mint_key` | Generic | The tick configuration key is not a valid mint |
| `missing_tick_for_underlying_mint` | Generic | No tick is configured for the underlying |
| `ladder_strike_source_unavailable` | Generic | The strike ladder source failed |
| `invalid_strikes` | Generic | The requested strike set is invalid |
| `duplicate_rfq_id` | Generic | The RFQ id already exists |
| `rfq_closed` | Generic | The RFQ is closed |
| `market_finalized` | Generic | The market is finalized |
| `market_disabled` | Generic | The market is disabled |
| `market_expired` | Generic | The market expired |
| `market_pre_expiry` | Generic | The market is in its pre-expiry window |
| `quote_identity_mismatch` | Generic | The quote does not match its order binding |
| `quote_refresh_required` | Generic | The quote must be refreshed; the maker got `QuoteRefreshRequested` |
| `duplicate_quote` | Generic | Duplicate quote `order_id` |
| `resource_limit` | Generic | A per-identity resource limit was reached |
| `kernel_timeout` | Generic | Core did not answer in time; a submission's outcome is unknown |
| `caps_authority_unavailable` | Generic | Quote risk authority changed; request a new quote |
| `vault_not_tradable` | Generic | A Vault counterparty cannot take this position now |
| `invalid_vault_counterparty` | Generic | Writer and holder must be distinct Vaults |
| `tx_build_failed` | Generic | The sponsored transaction could not be built; the message is from a fixed list |
| `order_already_submitted` | Generic | The order was already handed to settlement |
| `sponsored_tx_not_ready` | Generic | The sponsored transaction is not ready yet |
| `invalid_signed_tx` | Generic | The signed transaction does not decode or match |
| `invalid_amount` | Generic | The swap amount is not a positive u64 string |
| `vault_authorization_failed` | Generic | Vault authorization is no longer valid |
| `vault_projection_unavailable` | Generic | Vault state is temporarily unavailable |
| `vault_safety_mode` | Generic | Vault safety mode forbids the action |
| `vault_pending_capital` | Generic | The idle capital window must be closed and unpriced |
| `vault_trading_closed` | Generic | The Vault cycle does not permit trading |
| `vault_mismatch` | Generic | The request Vault differs from the session Vault |
| `vault_governance_required` | Generic | Recovery and closing swaps need governance |
| `vault_swap_invalid_state` | Generic | Vault state does not permit this swap |
| `vault_swap_invalid_amount` | Generic | The swap amount is invalid for the Vault |
| `vault_swap_idempotency_conflict` | Generic | The swap id is already bound to different swap data |
| `vault_swap_quote_expired` | Generic | The swap quote expired; request a new one |
| `vault_swap_signed_tx_rejected` | Generic | The signed swap transaction was rejected |
| `vault_swap_oracle_stale` | Generic | The Pyth price is too old; retry within seconds |
| `vault_swap_route_unavailable` | Generic | No swap route right now |
| `vault_swap_service_unavailable` | Generic | Another transient swap-service failure |

## Correlation semantics

There's no single global correlation id. Correlation is per-request, via the `request_id` field on messages that have a defined response.

- `Subscribe` / `Unsubscribe` carry a mandatory `request_id`. The server echoes it in `SubscribeAck` / `UnsubscribeAck`, and the `subscribed` / `unsubscribed` arrays contain only the diff for this call — not the full subscription list.
- `GetSubscriptions` carries `request_id`; `Subscriptions` echoes it.
- Query-style `Get*` operations require `request_id` and echo it in their responses.
- Correlated failures use `RequestError { request_id, error }`; `Error` has no request ID.
- `RfqRequest`, `AcceptQuote` and `SubmitSignedSponsoredTx` require `request_id` and get exactly one
  reply: `CommandAccepted { request_id }` or `RequestError { request_id, error }`, including for
  routing gates (rate limit, pause, invite gate, wrong endpoint, overload). `CommandAccepted` means
  the server accepted the command, never that anything landed on chain. Lifecycle pushes
  (`RfqCreated`, `OrderAccepted`, `OrderFailed`, …) stay keyed by `rfq_id` / `order_id` and may
  arrive before or after the receipt. `RfqRequest.request_id` is also the idempotency key: a
  repeated `request_id` returns the RFQ it created, echoed in `RfqCreated.request_id`. Not every handler follows the correlated path: `GetTokenMarketsInfo` can return an uncorrelated error or no response for an empty market set. See [its failure behavior](taker-api.md#tokenmarketsinfo).
- Exception: when the server's inbound queue for a connection is full it sends an uncorrelated
  `Error { code: "server_overloaded" }` and closes the socket with `4003 rate_limited`. Every request
  still outstanding on that connection has an unknown outcome; resolve submissions by status after
  reconnecting.
- Indicative pricing uses `request_id` on both `IndicativePricesRequest` (server → maker) and `IndicativePricesResponse` (maker → server).
- Broadcasts (`RfqBroadcast`, `TradeExecuted`, `StatsUpdate`, etc.) don't carry `request_id`.

### SubscribeAck / UnsubscribeAck

```json
{
  "type": "SubscribeAck",
  "data": {
    "request_id": "uuid",
    "subscribed": ["rfqs", "trades"]
  }
}
```

```json
{
  "type": "UnsubscribeAck",
  "data": {
    "request_id": "uuid",
    "unsubscribed": ["stats"]
  }
}
```

`subscribed` / `unsubscribed` are the channels that were actually added or removed —
channels already subscribed (or already absent) are not repeated.

## Lifecycle terms

Use the same terms across all external docs:

### RFQ lifecycle
- **Terminal RFQ event**: `RfqClosed`. Close RFQ state only on this event.
- **Fill-details event (maker)**: `QuoteFilled`. Contains fill details for the winning maker.
- **Order-level confirmation (taker)**: `OrderConfirmed`. Confirms selected order with
  on-chain identifiers (`tx_signature`, `position_pda`).

### Position lifecycle (post-fill)
- `open` — position created, collateral locked, premium paid to taker.
- `funded` — maker deposited settlement asset via `DepositFundsToPosition`.
- `settled` — market finalized, assets distributed based on ITM/OTM outcome.
- `liquidated` — an unfunded ITM position was closed by a liquidation transaction.

Trade lifecycle and payoff: [Protocol flow](protocol-flow.md).

## Supported channels

`WsChannel` values and what each carries. Channels are role-oriented — subscribe only to the ones your role consumes.

| Channel | Producer(s) | Audience |
|---|---|---|
| `rfqs` | `RfqBroadcast`, `IndicativePricesRequest`, quote/order lifecycle | **Makers.** Takers do not subscribe — a taker's own RFQ/quote/order events (`RfqCreated`, `QuoteReceived`, `OrderAccepted`, `SponsoredTxToSign`, `Order*`, `RfqClosed`) are delivered session-addressed regardless of subscription. |
| `trades` | `TradeExecuted` | Public trade tape. Participants also receive their own fills session-addressed. |
| `stats` | `StatsUpdate` | Global venue stats. |
| `chain_events` | `ChainEvent` (position opened / settled / liquidated, market finalized) | Anyone. **This is how a taker tracks position outcomes** (settlement / liquidation). |
| `markets` | `MarketCreated`, `MarketFinalized` | Anyone tracking the tradable market set. |
| `vaults` | `VaultCatalogChanged`, `VaultProjectionAdvanced` | Maker and taker sessions that list or watch vaults: they receive both, `VaultProjectionAdvanced` for every vault. `/vault` operator sessions are subscribed automatically and receive `VaultProjectionAdvanced` for their own vault only, and no `VaultCatalogChanged`. |

There is no `positions` channel: `PositionUpdated` and `MakerCapsUpdated` are delivered **owner-direct to the maker's session**, with no subscription.

Takers typically subscribe to `chain_events` (position outcomes) plus optionally `trades` / `stats` / `markets`. They do **not** need `rfqs`.

### Delivery criticality

When your outbound queue is full, the server handles a pushed message in one of two ways:

- **Critical**: the message is dropped and the connection is closed with `4002 slow_consumer`. Responses and some pushes are retried briefly first; lifecycle pushes fanned out from chain events are not. Reconnect and re-read state.
- **Best-effort**: the message is dropped silently and the connection stays open.

| Audience | Critical | Best-effort |
|---|---|---|
| `/maker` | `RfqBroadcast`, `QuoteReceived`, `QuoteAcknowledged`, `QuoteRefreshRequested`, `OrderAccepted`, `SponsoredTxToSign`, `OrderSubmitted`, `OrderConfirmed`, `OrderFailed`, `MarketCreated`, `MarketFinalized`, `VaultCatalogChanged`, `VaultProjectionAdvanced` | `TradeExecuted`, `ChainEvent`, `StatsUpdate`, `RfqSkipped`, `IndicativePricesRequest` |
| `/taker` | everything except `StatsUpdate` | `StatsUpdate` |
| `/maker/data`, `/vault` | everything | — |

Messages addressed to your session alone (responses, your own lifecycle, `PositionUpdated`, `MakerCapsUpdated`) always follow the critical rule.

`TradeExecuted` is a best-effort live hint and may be lost, delayed, or repeated.
Merge it with role-appropriate authoritative history by `trade.id`; never derive
trade counts or irreversible actions from the number of received frames.

## Timeout hierarchy

```
market.expiry_ts
  └─ rfq.expires_at                     # Auction window (no new quotes or accepts after this)
       └─ quote_refresh_lead

quote.valid_until                        # Cryptographic expiry (on-chain enforcement)
  └─ effective_expiry                    # = valid_until - settlement_buffer (server-side trading cutoff)

rfq.signature_deadline
  └─ tx_submit_timeout
```

### `rfq.expires_at` vs `quote.valid_until`

`expires_at` is the auction deadline. `valid_until` is the on-chain order validity.

- **`rfq.expires_at`** controls how long makers can submit quotes and the taker can accept. After this time new quoting/acceptance stops; an enqueued order remains unresolved until execution evidence arrives.
- **`quote.valid_until`** is the cryptographic expiry the maker signs into the order. The on-chain program rejects settlement if `valid_until` has passed.

Makers should normally choose `valid_until` beyond `expires_at` to cover settlement; this relationship is not a Core invariant. A taker might accept a quote at second 59 of a 60-second auction. After that, the server builds a sponsored tx, the taker signs it, and the tx confirms on Solana. This can take up to 90 seconds. If `valid_until` equaled `expires_at`, the on-chain order would expire before the tx lands.

### Recommended `valid_until` range

```
min:  now + min_signature_expiry_seconds          (default 100s)
max:  rfq.expires_at + settlement_buffer_seconds  (recommended upper bound)
hard max: market.expiry_ts
```

Setting `valid_until` above the recommended max but no later than
`market.expiry_ts` is accepted, but provides no benefit: after
`rfq.expires_at` the server will not allow accepts, so extra on-chain validity
only increases the maker's exposure window without enabling additional trades.
Values later than `market.expiry_ts` are rejected with
`QuoteRejected.reason = "market_expired"`.

Example: RFQ with `expires_at = now + 60s`, settlement buffer 90s:

```
valid_until range: [now + 100s, now + 150s]
effective_expiry:  [now + 10s,  now + 60s]
```

The maker's quote is tradeable for up to 60 seconds (the full RFQ lifetime)
and the on-chain order is valid long enough for settlement to land.

### Invariants

- `quote.valid_until >= now + min_signature_expiry_seconds` (default 100s; server rejects shorter expiries)
- `quote.valid_until <= market.expiry_ts` (hard server-side bound)
- `effective_expiry = quote.valid_until - settlement_buffer` (default 90s)
- `rfq.expires_at < market.expiry_ts` (recommended client-side validation)
- `signature_deadline <= min(effective_expiry, rfq.expires_at)`

`signature_deadline` appears in `QuoteSelected` (server -> maker; see [maker-api.md](maker-api.md)) and `SponsoredTxToSign` (server -> taker; see [taker-api.md](taker-api.md)).

## Reconciliation safety

Use entity versions where available:
- `rfq_version` for RFQ lifecycle progression
- `order_version` for order lifecycle progression

`order_version` is present on order lifecycle pushes, not on `OrderStatus`. Lifecycle precedence uses ranks `accepted = 1`, `submitted = 2`, `failed = 3`, `confirmed = 4`. Confirmation can replace a locally recorded failure; a transition may skip a rank and replay keeps the same rank.

`GetOrderStatus` instead returns `state: pending | confirmed { position_pda } | unknown`. `unknown` and timeouts do not prove nonexecution. Reconnect reads are not an atomic snapshot; keep unresolved obligations separately from transient RFQ/UI caches.

On client side:
- apply update only if `new_version > current_version`
- ignore stale updates (`new_version < current_version`)
- treat equal versions as idempotent replay
