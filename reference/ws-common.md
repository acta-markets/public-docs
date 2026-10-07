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
| Timestamps | Unix seconds |
| `server_time_unix_ms` | Unix milliseconds |

`price` and `strike` are always 1e9 fixed-point per 1 underlying unit, independent of mint decimals.


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

### What to sign

Validate the entire challenge **before invoking the signer**. Require the exact text and blank lines above, a nonce of exactly 64 lowercase hex characters, a valid UTC timestamp in `YYYY-MM-DDTHH:MM:SSZ` format, and a final `\n` after that timestamp. Reject extra lines, alternate domains, missing final newlines and arbitrary 32-byte strings. A maker challenge must have no `Wallet:` line. A taker challenge must include that line and it must match the wallet being authenticated.

After validation, sign the original UTF-8 bytes without normalization, additional prefix or hashing. The fixed text already supplies the authentication domain. Signing arbitrary endpoint text with the order key can authorize an order instead of authentication.

Rust SDK 0.4.3 and TS SDK 0.1.6 validate authentication messages before invoking signers in their managed clients. Raw Rust clients use `validate_maker_auth_challenge`; raw TS clients use `validateAuthChallenge`, which checks the grammar of both server formats. This does not establish endpoint trust or prevent relay of a genuine server challenge. Rust 0.4.2 and TS 0.1.5 lack these checks.

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

- Max auth attempts per connection: `3`. After 3 failures, connection is closed.
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

`reason` is a `snake_case` code. `message` is optional and may be omitted.

Current `reason` codes:
- `session_expired`, `already_authenticated`
- `auth_timeout`, `too_many_auth_attempts`
- `invalid_pubkey`, `invalid_signature`, `pubkey_mismatch`, `challenge_mismatch`
- `maker_not_registered`

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

### Common `generic.code` values

- `parse_error`
- `message_too_large`
- `too_many_parse_errors`
- `hello_required`
- `hello_timeout`
- `hello_already_sent`
- `session_replaced`
- `session_expired`
- `already_authenticated`
- `trading_paused`
- `internal_error`

This list grows over time; keep a fallback branch for unknown codes.

JavaScript integrators must preserve integer precision: a wire `u64` can exceed `Number.MAX_SAFE_INTEGER`. TS SDK represents most WS amounts as `number`; it reports `unsafe_integer` for oversized amount literals but still dispatches the message. Do not use such rounded values for signing or accounting. Inbound nonces have separate lossless handling. For full-range raw JSON amounts, use lossless parsing and bigint arithmetic; `BigInt(JSON.parse(...).amount)` cannot repair precision already lost. This does not change the wire encoding or the `1e9` scale.

## Correlation semantics

There's no single global correlation id. Correlation is per-request, via the `request_id` field on messages that have a defined response.

- `Subscribe` / `Unsubscribe` carry a mandatory `request_id`. The server echoes it in `SubscribeAck` / `UnsubscribeAck`, and the `subscribed` / `unsubscribed` arrays contain only the diff for this call — not the full subscription list.
- `GetSubscriptions` carries `request_id`; `Subscriptions` echoes it.
- Query-style `Get*` operations require `request_id` and echo it in their responses.
- Correlated failures use `RequestError { request_id, error }`; `Error` has no request ID. Not every handler follows the correlated path: `GetTokenMarketsInfo` can return an uncorrelated error or no response for an empty market set. See [its failure behavior](taker-api.md#tokenmarketsinfo).
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
| `positions` | — (no public producer) | `PositionUpdated` is delivered **owner-direct to the maker's session** by PDA lookup, never via this channel. The channel yields nothing for takers. |

Takers typically subscribe to `chain_events` (position outcomes) plus optionally `trades` / `stats` / `markets`. They do **not** need `rfqs` or `positions`.

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
