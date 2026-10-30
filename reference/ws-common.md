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
| `strike`, `lower_strike`, `upper_strike` | `u64`, quote per 1 underlying unit, 1e9 scale |
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
active set: markets that are not finalized, not disabled and before their trading cutoff.
For closed and finalized markets use `GET /api/v1/markets?active=false`.

### TokenInfo

One object per mint, on both transports. `role` is a tagged union.

```typescript
type TokenInfo = {
  mint: string;
  symbol: string;
  decimals: number;
  oracle_feed_ids: string[];       // Pyth feed ids, hex without 0x; [] when none is known
  role:
    | { type: "quote" }
    | {
        type: "underlying";
        size_rule: {               // underlying atomic units
          min_size: number;
          max_size: number;
          step: number;
        };
        strike_tick: number;       // strike ladder spacing, 1e9 scale
        quote_mints: string[];     // quote mints of its active markets
      };
};
```

A valid `quantity` satisfies `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`.
Only tokens of active markets are listed.

### MarketInfo

Used by `Snapshot.markets`, `Markets` and HTTP `/api/v1/markets`. Markets name tokens by mint.

```typescript
type MarketInfo = {
  pda: string;
  underlying: string;                // underlying mint
  quote: string;                     // quote mint
  expiry_ts: number;
  is_put: boolean;
  products: Product[];               // non-empty; see below
  status: "active" | "closed" | "finalized";
  settlement_price: number | null;   // 1e9 scale; null until the market is settled
};

type Product =
  | "covered_call" | "cash_secured_put" | "call_spread" | "put_spread" | "cash_put";
```

`products` lists the products enabled for the pair that fit the market's side. A put market
(`is_put: true`) lists a subset of `cash_secured_put`, `put_spread` and `cash_put`. A call market
lists a subset of `covered_call` and `call_spread`. A market with no product on its side is not
listed.

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

## Quantity and collateral by product

`quantity` on the wire is always in underlying atomic units, regardless of position type.
The products differ in what the collateral is:

**Covered call** — user deposits underlying (e.g. SOL).
`quantity` maps directly to the deposit: `quantity = userInput * 10^underlying_decimals`.

**Cash-secured put** — user deposits quote (e.g. USDC).
The collateral is `floor(strike * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`, in quote atoms.
To turn a budget in quote atoms into a quantity, round down, then snap to the [size rule](taker-api.md#marketdescriptors-payload):

```
raw      = floor(quoteAtoms * 1e9 * 10^underlying_decimals / (strike * 10^quote_decimals))
quantity = min_size + floor((raw - min_size) / step) * step
```

Stop if `raw < min_size`, and cap `quantity` at `max_size`. Rounding up would need more collateral than the budget, and a quantity off the step is rejected with `invalid_quantity_size_rule`. For 100 USDC at strike 136 with 9 and 6 decimals, `min_size` 10000000 and `step` 10000000: `raw` is 735294117, `quantity` is 730000000 and the collateral is 99280000.

And to display the deposit back in quote terms:

```
quoteAmount = quantity * strike / 1e9 / 10^underlying_decimals
```

`underlying_decimals` comes from `MarketDescriptorInfo`.

**Call spread, put spread, cash put** — collateral is quote. Spreads and cash puts settle in cash
in the quote mint. The width is `upper_strike - lower_strike` for a spread and `strike` for a cash
put (a put spread with legs `(0, strike)`). The collateral in quote atoms is
`ceil(width * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))`. The gross price
`G` per underlying unit satisfies `0 < G < width`. To size from a quote budget, use the formulas
above with `width` in place of `strike`.

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
`quoteAmountToQuantity` takes whole quote tokens, rounds to the nearest atom and ignores the size
rule, so snap its result before sending.

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

Rust SDK (`acta-maker-sdk` 0.6.0) and TS SDK (`@acta-markets/ts-sdk` 0.3.0) validate authentication messages before invoking signers in their managed clients. Raw Rust clients use `validate_maker_auth_challenge`, or `validate_vault_auth_challenge(challenge, operator, vault)` for a vault session; raw TS clients use `validateAuthChallenge`, which checks the grammar of the maker and taker formats and rejects the vault format, and `assertVaultAuthChallenge(challenge, operator, vault)` for a vault session. This does not establish endpoint trust or prevent relay of a genuine server challenge.

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

**Maker**: `pubkey` is the `maker_owner` pubkey registered on-chain. Sign the challenge with the maker's registered `quote_signing` key, the same key that signs quotes. The server looks up `maker_owner` and verifies the authentication signature against its `quote_signing` key.

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
| `4003` | `rate_limited` | Per-connection or per-wallet message rate exceeded, or the inbound queue is full. See [Rate limits and admission](#rate-limits-and-admission) |
| `4004` | `session_replaced` | A newer connection authenticated as the same maker or taker session |
| `4005` | `hello_timeout` | `Hello` not received in time |
| `4006` | `unsupported_protocol_version` | `Hello.protocol_version` is not `1.0.0` |

A rejected WebSocket upgrade returns an HTTP status with the same JSON body as the HTTP API,
`{"error": "...", "code": "..."}`: `403 origin_not_allowed`, `429 rate_limited` (too many
connections from your IP), `503 too_many_connections`, `503 server_overloaded`,
`503 server_shutting_down`.

### Rate limits and admission

Limits are server configuration; the values below are the defaults. A limit is a token bucket:
`per_sec` tokens refill each second up to `burst`. A per-type limit rejects the message with
`rate_limited` and keeps the connection. The per-connection and per-wallet message limits close it.

| Limit | Scope | Default | On exceed |
|---|---|---|---|
| Message rate | Every connection | 30/s, burst 60 | `rate_limited` error, then close `4003` |
| Wallet message rate | All connections of one taker wallet | 50/s, burst 100 | `rate_limited` error, then close `4003` |
| RFQ messages (`RfqRequest`, `AcceptQuote`, `CancelRfq`, `SwapQuote`, `SwapSubmit`) | Taker and Vault connections | 10/s, burst 20 | `rate_limited` error for that message; the connection stays open |
| Quote messages (`Quote`, `ReplaceQuote`, `BatchQuotes`, `CancelQuote`, `CancelAllQuotes`) | Maker and Vault connections | 50/s, burst 100 | The message is rejected and the connection stays open. `Quote` and `ReplaceQuote` get `QuoteRejected` with reason `rate_limited`; `BatchQuotes` gets `BatchQuotesAck` with every result rejected the same way; cancels get a `rate_limited` error |
| Query messages (`Get*`, `CheckQuote`, subscription messages) | Every connection | 20/s, burst 40 | `rate_limited` error for that message; the connection stays open |

`BatchQuotes` spends one token per quote in the batch, and `SwapQuote` spends five. Every other
message spends one. Quote messages are counted only by the quote bucket, not by the connection
message rate, unless the quote bucket is disabled or the batch exceeds `max_batch_quotes`.

Admission limits cap work in flight across the server:

| Limit | Default | On exceed |
|---|---|---|
| Concurrent query handlers | 128 | `server_overloaded` for that message |
| Concurrent sponsored-transaction submissions | 64 | `server_overloaded` for that message |
| Connection command queue (messages received but not yet processed) | n/a | uncorrelated `server_overloaded`, then close `4003` |

Other defaults:

| Setting | Default |
|---|---|
| Max message size | 32 KiB |
| `max_batch_quotes` | 50 |
| Parse error budget | 3 consecutive parse errors |
| Max connections, server total | 7500 |
| Max connections per IP | 50 |
| Max auth attempts | 3 |
| Auth deadline | 15 s |
| Idle timeout | 90 s |
| Hello timeout | 5 s |

### Logout

```json
{ "type": "Logout" }
```

Unit variant, no `data` field. Server responds with `LogoutSuccess` and closes the connection.

```json
{ "type": "LogoutSuccess", "data": {} }
```

## Fee model

The protocol charges a fee per trade. Fees are set per quote mint in basis points (bps). The
on-chain config stores `protocol_fee_bps_premium` and `protocol_fee_bps_volume`; the venue
publishes both at `GET /api/v1/venue`.

- `price` in the `Quote` message is the gross premium per 1 underlying unit (1e9 scale). The maker
  signs it, and it goes into the order id preimage as `gross_price`.
- On chain, gross premium is scaled to quote-token atomic units. The premium fee is
  `protocol_fee_bps_premium` of the premium. The volume fee is `protocol_fee_bps_volume` of the
  exposure: the strike notional for physical products, the width notional for spreads and cash puts.
  The charged fee is `min(premium_fee, volume_fee)`.
- `net_price` in `QuoteReceived` is `Price | null` and is for display only. The authoritative
  amount is the net premium after the fee and token-decimal scaling.
- `total_premium` in position rows is the net premium in quote atoms after the protocol fee. It is
  the authoritative money figure. `QuoteFilled` carries no fee or net fields.
- The fee does not change what the maker signs. The contract applies it when the position opens.

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
- Explicit `null` for non-makers (admins, takers) and for makers not yet registered on-chain.

### AuthError

```json
{
  "type": "AuthError",
  "data": {
    "reason": "session_expired",
    "message": "Session expired"
  }
}
```

`reason` is a `snake_case` code from the [WS error code catalog](#ws-error-codes) (frames
marked `AuthError`). `message` is a required string.

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

Endpoint values are `maker`, `maker_data`, `taker`, and `vault`.

### RequestError: request-correlated errors

Sent when the server can correlate the failure to a specific client request. The `request_id`
echoes the client's `request_id` from the originating message:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "1e3a5c7e-9b0d-4f26-8a4c-6e8a0c2e4f93",
    "error": { "type": "RfqNotFound" },
    "delivery": null
  }
}
```

The inner `error` has the same `ServerError` shape as `Error.data`.

`delivery` is always present. It is `null` except on failures of `SubmitSignedSponsoredTx` and `SwapSubmit`:

```json
{
  "type": "RequestError",
  "data": {
    "request_id": "1e3a5c7e-9b0d-4f26-8a4c-6e8a0c2e4f93",
    "error": { "type": "Generic", "data": { "code": "vault_swap_quote_expired", "message": "Vault swap quote expired" } },
    "delivery": "not_forwarded"
  }
}
```

`delivery` describes this one request only:

- `not_forwarded`: this request was rejected before it was handed to the keeper (routing gates,
  validation, a pre-enqueue rejection).
- `unknown`: this request may have reached the keeper pipeline (timeouts, queue errors,
  `kernel_timeout`, `order_already_submitted`). Resolve it with `GetOrderStatus` (orders) or
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

### ServerError catalog

This is the closed set of `type` values. A variant without `data` has no `data` key: `{ "type": "RfqNotFound" }`. Enum values are lowercase snake_case strings.

| `type` | `data` | Meaning |
|---|---|---|
| `Unauthenticated` | `{ action }` | The action needs an authenticated session |
| `Unauthorized` | `{ role, action }` | Only `role` may do this |
| `RfqNotFound` | none | The RFQ does not exist |
| `RfqNotActive` | none | The RFQ is not accepting quotes |
| `RfqAlreadyLocked` | none | Another order holds the RFQ |
| `InvalidState` | `{ state }` | The RFQ is in the wrong state for this command |
| `QuoteLocked` | `{ reason }` | The quote cannot be changed |
| `QuoteNotFound` | none | The quote does not exist or is already gone |
| `QuoteExpired` | none | The quote has expired |
| `QuoteExpiryTooShort` | `{ min_seconds }` | `valid_until` is less than `min_seconds` from now |
| `InvalidStrike` | none | The strike is not allowed on this RFQ |
| `InvalidValidUntil` | none | `valid_until` is invalid |
| `OrderIdMismatch` | none | `order_id` does not match the order |
| `UnknownOrder` | none | The server does not know this `order_id` |
| `SignatureTimeout` | none | The signature window closed; select another quote |
| `OracleNotReady` | none | The underlying oracle is not available yet; retry |
| `OraclePriceStale` | `{ age_seconds }` | The oracle price is `age_seconds` old |
| `OraclePriceNotReady` | none | The oracle has no usable price yet |
| `InvalidPositionType` | none | The product does not fit the market's side |
| `InvalidMarket` | `{ pubkey }` | `pubkey` is not a valid market |
| `MarketMetadataIncomplete` | `{ details }` | Active markets lack oracle PDAs or decimals |
| `TokenMetadataIncomplete` | `{ details }` | Active markets lack token decimals |
| `RateLimit` | a `RateLimitReason` string | A resource limit was hit |
| `Cap` | a `CapError` object | A cap was hit. See [`caps.md`](caps.md) |
| `InternalError` | none | Unexpected server failure |
| `KernelNotAvailable` | none | The matching engine is not available; retry |
| `DbDisabled` | `{ feature }` | This read needs a database and the server has none |
| `ServerShuttingDown` | none | The server is draining; reconnect |
| `InviteRequired` | none | The taker wallet must redeem an invite |
| `Invite` | an `InviteErrorReason` string | The invite was refused |
| `Claim` | a `ClaimErrorReason` string | The referral code claim was refused |
| `WrongEndpoint` | `{ endpoint, allowed_endpoints }` | The message is not accepted on this endpoint |
| `Generic` | `{ code, message }` | `code` is in the [WS error code catalog](#ws-error-codes) |

| Field or string | Values |
|---|---|
| `action` | `submit_quotes`, `cancel_quotes`, `create_rfqs`, `accept_quotes`, `submit_signed_tx`, `cancel_rfqs`, `access_rfq`, `request_positions`, `subscribe`, `unsubscribe`, `read_maker_data` |
| `role` | `maker`, `taker`, `owner` |
| `state` | `not_active`, `not_pending_signature`, `cannot_be_cancelled` |
| `reason` | `rfq_locked`, `order_submitted` |
| `min_seconds`, `age_seconds` | Integer seconds |
| `feature` | `maker_positions`, `maker_markets`, `market_descriptors`, `expiries`, `tokens`, `mm_summary`, `my_quotes`, `my_trades` |
| `endpoint`, `allowed_endpoints` | `maker`, `maker_data`, `taker`, `vault` |
| `InviteErrorReason` | `invalid_code`, `code_exhausted`, `code_expired`, `code_disabled`, `code_owner_inactive`, `code_owner_blacklisted`, `already_registered`, `internal_error` |
| `ClaimErrorReason` | `not_registered`, `invalid_format`, `code_taken`, `reserved`, `internal_error` |

`RateLimitReason` is in the [Maker](maker-api.md#ratelimitreason-in-typed-rate_limit-error) and [Taker](taker-api.md#ratelimitreason-in-typed-ratelimit-error) references. `CapError` is in [CapError](maker-api.md#caperror).

```json
{ "type": "Error", "data": { "type": "QuoteLocked", "data": { "reason": "rfq_locked" } } }
```

```json
{ "type": "Error", "data": { "type": "QuoteExpiryTooShort", "data": { "min_seconds": 60 } } }
```

```json
{ "type": "Error", "data": { "type": "OraclePriceStale", "data": { "age_seconds": 120 } } }
```

```json
{ "type": "Error", "data": { "type": "DbDisabled", "data": { "feature": "mm_summary" } } }
```

JavaScript integrators must preserve integer precision: a wire `u64` can exceed `Number.MAX_SAFE_INTEGER`. TS SDK represents most WS amounts as `number`; for an oversized amount literal it reports `unsafe_integer` and drops the whole frame, so the matching event never fires; recover state with a read (`GetMyActiveRfqs`, `GetOrderStatus`). Inbound nonces have separate lossless handling. For full-range raw JSON amounts, use lossless parsing and bigint arithmetic; `BigInt(JSON.parse(...).amount)` cannot repair precision already lost. This does not change the wire encoding or the `1e9` scale.

## WS error codes

The closed set of `Generic.code` values (in `Error.data` and `RequestError.data.error`) and
`AuthError.reason` values. The server sends nothing outside this table; typed `ServerError`
variants (`RfqNotFound`, `Unauthenticated`, `WrongEndpoint`, …) are not codes. A client that
receives a code outside this set fails to decode the frame.

| Code | Frames | Meaning |
|---|---|---|
| `parse_error` | Generic | The frame is not valid JSON or not a known message |
| `too_many_parse_errors` | Generic | Consecutive parse errors exhausted the budget; the connection closes |
| `hello_required` | Generic | The first message was not `Hello`; the connection closes |
| `hello_timeout` | Generic | No `Hello` in time; followed by close `4005` |
| `hello_already_sent` | Generic | A second `Hello` on one connection |
| `rate_limited` | Generic | Connection or wallet rate limits close with `4003`; a per-type limit rejects only the message |
| `server_overloaded` | Generic | The admission limit for this message class is full; retry with backoff. A full connection command queue sends it uncorrelated and closes with `4003` |
| `server_busy` | Generic, AuthError | A bounded internal queue (auth, RFQ creation) is full; retry |
| `internal_error` | Generic | Unexpected server failure; the message is sanitized |
| `temporarily_unavailable` | Generic, AuthError | A dependency (database, matching engine, registry) is not ready; retry with backoff |
| `clock_error` | Generic, AuthError | The server clock is before the Unix epoch |
| `time_overflow` | Generic | The requested deadline cannot be represented |
| `trading_paused` | Generic | Venue trading is paused; order-path messages are rejected |
| `batch_quotes_too_large` | Generic | `BatchQuotes` exceeds `max_batch_quotes` |
| `session_replaced` | Generic | A newer connection took over this session; do not reconnect automatically |
| `session_expired` | Generic, AuthError | The session or resume credential expired; authenticate again |
| `session_limit_exceeded` | AuthError | Too many concurrent sessions for this identity |
| `session_conflict` | Generic | The server no longer binds this session to the identity |
| `blacklisted` | Generic, AuthError | The identity is blacklisted; terminal |
| `maker_unregistered` | Generic | The maker left the registry while connected; the connection closes |
| `maker_not_registered` | Generic, AuthError | The pubkey is not a registered maker |
| `auth_timeout` | AuthError | Authentication did not finish in time; followed by close `4001` |
| `too_many_auth_attempts` | AuthError | Auth attempt budget exhausted; the connection closes |
| `already_authenticated` | AuthError | Auth started on an authenticated connection |
| `challenge_not_issued` | AuthError | `AuthChallenge` without a preceding `AuthRequest` |
| `challenge_mismatch` | AuthError | The signed challenge differs from the issued one |
| `pubkey_mismatch` | AuthError | `AuthChallenge.pubkey` differs from the `StartAuth` pubkey |
| `invalid_signature` | AuthError | The challenge signature does not verify |
| `operator_mismatch` | AuthError | The Vault challenge was signed by another operator |
| `vault_auth_not_started` | AuthError | Vault challenge answered without `StartVaultAuth` |
| `vault_auth_required` | Generic | A Vault message on a session without Vault authority |
| `vault_resume_not_supported` | AuthError | `ResumeAuth` on `/vault` |
| `vault_operator_unauthorized` | AuthError | The operator holds no current grant for the Vault |
| `vault_authorization_unavailable` | AuthError | Vault authority could not be read; retry |
| `vault_session_invalidated` | Generic | The Vault grant changed or was revoked; the session closes |
| `invalid_vault_subscription` | Generic | The subscription is not allowed for a Vault session |
| `invalid_products` | Generic | `Subscribe.products` is empty |
| `subscription_scope_too_large` | Generic | The subscription mint filter exceeds its limit |
| `history_projection_incomplete` | Generic | A history read lacks listener metadata; retrying does not help until reconciliation |
| `invalid_limit` | Generic | Page `limit` outside the accepted range for the query |
| `invalid_timeout` | Generic | `timeout_seconds` outside the venue's `min_rfq_timeout_seconds` to `max_rfq_timeout_seconds` |
| `market_not_found` | Generic | The market is not in the registry |
| `invalid_program_id` | Generic | The server program id is misconfigured |
| `unsupported_protocol_version` | Generic | `Hello.protocol_version` is not `1.0.0`; followed by close `4006` |
| `invalid_strike_notional` | Generic | Strike × quantity is zero or not representable on chain |
| `insufficient_collateral` | Generic | Wallet taker cannot fund the RFQ collateral. `RfqRequest` is rejected before maker broadcast; `message` includes `required` and `available` in collateral atoms |
| `invalid_quantity_size_rule` | Generic | The quantity is outside the size rule |
| `product_not_allowed` | Generic | The pair policy is absent or does not allow the requested product |
| `missing_size_rule_for_underlying_mint` | Generic | No size rule is configured for the underlying |
| `invalid_underlying_mint_key` | Generic | The tick configuration key is not a valid mint |
| `missing_tick_for_underlying_mint` | Generic | No strike tick is configured for the underlying |
| `ladder_strike_source_unavailable` | Generic | The strike ladder source failed |
| `invalid_strikes` | Generic | The requested strike set is invalid |
| `taker_buy_unsupported` | Generic | `side` is `buy`; only taker `sell` is supported |
| `spread_invalid_strikes` | Generic | Zero, reversed or otherwise invalid spread strikes |
| `spread_tick_violation` | Generic | A spread strike is off the underlying mint's strike tick |
| `spread_premium_invalid` | Generic | Price is at or above the width, the gross transfer is zero, or the cash arithmetic is unrepresentable |
| `duplicate_rfq_id` | Generic | The RFQ id already exists |
| `rfq_closed` | Generic | The RFQ is closed |
| `market_finalized` | Generic | The market is finalized |
| `market_disabled` | Generic | The market is disabled |
| `market_expired` | Generic | The market expired |
| `market_pre_expiry` | Generic | The market is in its pre-expiry window |
| `quote_identity_mismatch` | Generic | The quote `terms` or `side` differ from the RFQ |
| `quote_refresh_required` | Generic | The quote must be refreshed; the maker got `QuoteRefreshRequested` |
| `duplicate_quote` | Generic | Duplicate quote `order_id` |
| `resource_limit` | Generic | A per-identity resource limit was reached |
| `kernel_timeout` | Generic | The matching engine did not answer in time; a submission's outcome is unknown |
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
Position rows carry a required `state`: `open`, `funded`, `liquidated` or
`settled { settlement: physical | cash }`.

- `open`: position created, collateral locked, premium paid to taker.
- `funded`: maker deposited the settlement asset via `DepositFundsToPosition`. Physical products only.
- `settled`: market finalized, assets distributed based on the ITM/OTM outcome. `settlement` is
  `physical` for covered calls and cash-secured puts, `cash` for spreads and cash puts.
- `liquidated`: an unfunded ITM position was closed by a liquidation transaction. Physical products only.

Spreads and cash puts settle in cash in the quote mint. Their lifecycle is `open` then `settled`.
`PositionUpdated` with `update_type` `funded` or `liquidated` does not occur for them.

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
| `/maker` | `RfqBroadcast`, `QuoteAcknowledged`, `QuoteRefreshRequested`, `MarketCreated`, `MarketFinalized`, `VaultCatalogChanged`, `VaultProjectionAdvanced` | `TradeExecuted`, `ChainEvent`, `StatsUpdate`, `RfqSkipped`, `IndicativePricesRequest` |
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

quote.valid_until                        # Cryptographic expiry (on-chain enforcement)
  └─ effective_expiry                    # = valid_until - settlement_buffer (server-side trading cutoff)
       └─ refresh point                  # = effective_expiry - quote_refresh_lead (quote freezes)

rfq.signature_deadline
  └─ tx_submit_timeout
```

### `rfq.expires_at` vs `quote.valid_until`

`expires_at` is the auction deadline. `valid_until` is the on-chain order validity.

- **`rfq.expires_at`** controls how long makers can submit quotes and the taker can accept. After this time new quoting/acceptance stops; an enqueued order remains unresolved until execution evidence arrives.
- **`quote.valid_until`** is the cryptographic expiry the maker signs into the order. The on-chain program rejects settlement if `valid_until` has passed.

Choose `valid_until` beyond `expires_at`. A taker can accept in the last second of an auction. The signature deadline is capped at `expires_at`, and the signed transaction confirms on Solana after that. If `valid_until` equaled `expires_at`, the quote would freeze before the auction ended, and the on-chain order could expire before the transaction lands.

### Recommended `valid_until` range

```
hard min:    now + min_quote_validity_seconds             (default 60 s)
recommended: rfq.expires_at + min_quote_validity_seconds  (plus a little slack)
hard max:    market.expiry_ts
```

`min_quote_validity_seconds` comes from `Welcome` or `GET /api/v1/venue`. It covers the settlement
buffer and the refresh lead. At `valid_until - settlement_buffer_seconds - quote_refresh_lead` the
server freezes the quote and sends `QuoteRefreshRequested`. The recommended value keeps that point
at or after `rfq.expires_at`, so the quote stays acceptable for the whole auction.

Values below the hard min are rejected with `quote_expiry_too_short`. Values later than
`market.expiry_ts` are rejected with `market_expired`. A value far above the recommended one is
accepted, but after `rfq.expires_at` the server allows no accepts, so it only widens the maker's
exposure window.

Example: RFQ with `expires_at = now + 60s`, settlement buffer 50 s, refresh lead 10 s:

```
valid_until:      now + 120s
effective_expiry: now + 70s
refresh point:    now + 60s
```

### Invariants

- `quote.valid_until >= now + min_quote_validity_seconds` (default 60 s; the server rejects shorter expiries)
- `quote.valid_until <= market.expiry_ts` (hard server-side bound)
- `effective_expiry = quote.valid_until - settlement_buffer_seconds` (default 50 s)
- `rfq.expires_at < market.expiry_ts` (recommended client-side validation)
- `signature_deadline = min(accept time + signature_timeout, effective_expiry, rfq.expires_at)`

`signature_deadline` appears in `QuoteSelected` (server -> maker; see [maker-api.md](maker-api.md)) and `SponsoredTxToSign` (server -> taker; see [taker-api.md](taker-api.md)).

## Reconciliation safety

Position rows carry `revision { slot, transaction_index, instruction_index }`. Compare revisions
lexicographically and apply a `PositionUpdated` or query row only if its revision is newer than the
stored one. `PositionUpdated` is `{ position, update_type }` with a full row. Equal revisions mean
equal state.

Use entity versions where available:
- `rfq_version` for RFQ lifecycle progression
- `order_version` for order lifecycle progression

`order_version` is present on order lifecycle pushes, not on `OrderStatus`. Lifecycle precedence uses ranks `accepted = 1`, `submitted = 2`, `failed = 3`, `confirmed = 4`. Confirmation can replace a locally recorded failure; a transition may skip a rank and replay keeps the same rank.

`GetOrderStatus` instead returns `state: pending | confirmed { position_pda } | unknown`. `unknown` and timeouts do not prove nonexecution. Reconnect reads are not an atomic snapshot; keep unresolved obligations separately from transient RFQ/UI caches.

On client side:
- apply update only if `new_version > current_version`
- ignore stale updates (`new_version < current_version`)
- treat equal versions as idempotent replay
