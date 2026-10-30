# Acta Taker Quickstart

JSON-layer path for a taker: auth, discover markets, open an RFQ for one of five products, accept a quote, sign the sponsored transaction, and track the resulting position. This is the language-neutral protocol walkthrough. A TypeScript SDK wraps the same flow in [`web-client-ts-sdk.md`](web-client-ts-sdk.md).

The message catalogue is in [`../reference/taker-api.md`](../reference/taker-api.md). Units and envelopes are in [`../reference/ws-common.md`](../reference/ws-common.md). Trade mechanics and settlement are in [`../reference/protocol-flow.md`](../reference/protocol-flow.md). Onboarding is in [`../reference/sandbox.md`](../reference/sandbox.md).

A taker needs a Solana wallet keypair. There is no registration and no on-chain setup step: any wallet can open RFQs (rate-limited, and invite-gated on closed mainnet; see [Invite gating](#invite-gating-closed-mainnet)).

The wire is closed. Every field is present on every frame, absent facts are `null`, and state-dependent shapes are tagged unions. The server rejects missing fields, unknown message types and unknown enum values.

---

## Endpoint

```
wss://devnet-api.acta.markets/taker
wss://beta-api.acta.markets/taker
```

## Connection and authentication

The first message from the client is `Hello`. The server accepts only `protocol_version` `1.0.0`. Any other value gets an `Error` with `unsupported_protocol_version` and the server closes the connection with code `4006`.

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "on_disconnect": "keep_quotes",
    "client_name": "my-app",
    "client_version": "0.1.0"
  }
}
```

`protocol_version`, `on_disconnect`, `client_name` and `client_version` are all required, and `Hello` accepts no other fields. `on_disconnect` is `cancel_quotes` or `keep_quotes`; it sets how a maker's quotes are handled on disconnect.

The server responds with `Welcome`. `server_time_unix_ms` is for clock sync, and `min_quote_validity_seconds` and `settlement_buffer_seconds` are the venue's quote window. Taker auth is lazy: you may authenticate immediately or defer until the first authenticated action. Discovery (`GetMarkets`, `GetMarketDescriptors`, `GetTokenCaps`, `GetIndicativePrices` and others) works before auth; see "Authentication requirements" in [`../reference/taker-api.md`](../reference/taker-api.md).

### Fresh sign

Send `StartAuth` with your wallet pubkey. The server replies with `AuthRequest` carrying a challenge string. The taker challenge includes a `Wallet:` line:

```
Acta RFQ Authentication

Sign this message to authenticate your wallet.

Wallet: {base58_pubkey}
Nonce: {hex_32_random_bytes}
Issued At: {RFC3339_timestamp}
```

Before signing, validate the complete challenge with the [canonical auth rules](../reference/ws-common.md#what-to-sign): the exact domain, wallet, nonce, timestamp and final newline. Reject arbitrary endpoint text. Then sign the original UTF-8 bytes with your wallet key (Ed25519, no prefix or hashing), base58-encode the 64-byte signature, and reply:

```json
{
  "type": "AuthChallenge",
  "data": {
    "challenge": "<from AuthRequest>",
    "signature": "<base58 ed25519 signature>",
    "pubkey": "<taker wallet pubkey, base58>"
  }
}
```

The signature is verified directly against `pubkey`. Auth must finish within 15 seconds of the challenge. The default allows three failed attempts and closes the connection on the fourth. On success the server sends `AuthSuccess { session_id, expires_at, maker_pda }` (`maker_pda` is `null` for a taker) followed by `Snapshot`.

### Session resume

`AuthSuccess.expires_at` is a Unix timestamp in seconds. Persist `session_id` and `expires_at`. On reconnect, if `now < expires_at`, skip the wallet signature:

```json
{ "type": "ResumeAuth", "data": { "session_id": "<saved session_id>" } }
```

A valid session returns `AuthSuccess`. An invalid, expired or revoked one returns `AuthError { reason: "session_expired", message }`, and you fall back to the fresh-sign flow. Sessions have a 24h default TTL (sliding, capped at 7 days) and survive disconnects. A failed `ResumeAuth` counts toward the three-attempt limit.

## Discovery

Fetch market metadata before opening an RFQ. `GetMarketDescriptors` returns the descriptor: `products`, `size_rule`, decimals, oracle PDAs and symbols.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "<uuid>" } }
{ "type": "GetTokens",            "data": { "request_id": "<uuid>" } }
{ "type": "GetExpiries",          "data": { "request_id": "<uuid>", "underlying_mint": { "type": "all" }, "quote_mint": { "type": "all" }, "is_put": { "type": "all" } } }
```

Query requests require `request_id`, and responses echo it. Discovery returns tradable markets only. Cache descriptors and re-read them when an authenticated session receives `ReferenceChanged`, which also covers a market passing its trading cutoff. Before auth, or without WebSocket, read `GET /api/v1/tokens` and `GET /api/v1/markets` with `If-None-Match` (see [HTTP API](../reference/http-api.md#reference-data)).

### Which products a market offers

Each market row carries `products`, the pair's products that fit the market's side:

| Market | `is_put` | Possible `products` |
|---|---|---|
| Call market | `false` | `covered_call`, `call_spread` |
| Put market | `true` | `cash_secured_put`, `put_spread`, `cash_put` |

A market with no product on its side is not listed. An RFQ for a product that is not in `products` fails with `product_not_allowed`. A put product on a call market fails with `InvalidPositionType`.

> **Size rules.** `size_rule = { min_size, max_size, step }` is in underlying atoms, keyed by underlying mint. A valid `quantity` satisfies `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`. If a market's underlying mint has no configured rule, `RfqRequest` is rejected with `missing_size_rule_for_underlying_mint`. `GetMarketDescriptors` and `GetTokens` fail the whole request if any active market lacks a rule. `GetTokens` returns the same rule on each underlying token.

> **Strike tick.** An underlying token carries `strike_tick`, `1e9`-scaled like strikes. Spread strikes and the cash put strike must be multiples of it.

## The RFQ flow

```
RfqRequest -> RfqCreated -> QuoteReceived (0..N) -> AcceptQuote
  -> SponsoredTxToSign -> [sign] -> SubmitSignedSponsoredTx
  -> OrderAccepted -> OrderSubmitted -> OrderConfirmed -> RfqClosed
```

### 1. Open the RFQ

`terms` is a tagged union that names the product and its strikes. `side` is the taker's side and must be `sell`; `buy` fails with `taker_buy_unsupported`.

| Product | `terms` |
|---|---|
| Covered call | `{ "type": "covered_call", "strike": 136000000000 }` |
| Cash-secured put | `{ "type": "cash_secured_put", "strike": 136000000000 }` |
| Call spread | `{ "type": "call_spread", "lower_strike": 150000000000, "upper_strike": 160000000000 }` |
| Put spread | `{ "type": "put_spread", "lower_strike": 130000000000, "upper_strike": 140000000000 }` |
| Cash put | `{ "type": "cash_put", "strike": 136000000000 }` |

`quantity` is always in underlying atoms, for every product, and must satisfy `size_rule`. For a UI that collects a quote amount (for example USDC), convert it and snap it to the size rule: see [Quantity and collateral by product](../reference/ws-common.md#quantity-and-collateral-by-product). The wallet must hold the [full collateral](../reference/taker-api.md#rfqrequest-taker---server) when it sends the RFQ.

```json
{
  "type": "RfqRequest",
  "data": {
    "request_id": "<uuid>",
    "market": "<market PDA, base58>",
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "quantity": 1000000000,
    "timeout_seconds": 30
  }
}
```

`request_id` is required and is also the idempotency key: repeating a request with the same `request_id` while the RFQ is active returns the same `rfq_id`. The server answers with `CommandAccepted { request_id }` or `RequestError { request_id, error, delivery }`. On success it also sends `RfqCreated { rfq_id, rfq_version, terms, side, request_id, expires_at, created_at, order_options }`, before or after the receipt, and broadcasts the RFQ to eligible makers. `order_options` is the list of `terms` makers may quote. For a spread or cash put it holds exactly your `terms`. For a covered call or cash-secured put it is a strike ladder of the same product.

### 2. Collect quotes

Makers stream `QuoteReceived`. Each quote is firm and hash-bound through `order_id`:

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "<uuid>",
    "terms": { "type": "covered_call", "strike": 136000000000 },
    "side": "sell",
    "maker": "<maker owner pubkey, base58>",
    "price": 50000000,
    "valid_until": 1710000350,
    "nonce": 42,
    "order_id": "0x<64 hex>",
    "net_price": 49750000
  }
}
```

- `terms` is one of the RFQ's `order_options`.
- `price` is the gross premium per underlying unit (`1e9` scale). Use it with `order_id` for all order operations.
- `net_price` is `Price | null`. It is a display estimate after the protocol fee. It is `null` for spreads and cash puts, and when the server has no fee configuration. The authoritative net premium is computed on chain at open and appears as `total_premium` on the position row.

You pick the winner; the server's ranking is advisory. One quote fills the full quantity, with no partial fills.

### Economics of spreads and cash puts

Call spreads, put spreads and cash puts are cash-settled in the quote mint. Their lifecycle is Open -> Settled; they are never funded or liquidated.

- Width `W` is `upper_strike - lower_strike` for a spread and `strike` for a cash put.
- The taker posts collateral `C = ceil(W * quantity * 10^quote_decimals / (1e9 * 10^underlying_decimals))` in quote atoms and receives the net premium.
- The gross price `G` satisfies `0 < G < W`.
- The protocol fee is `min(premium_fee, volume_fee)`. The volume fee applies to the exposure: the width notional for spreads and cash puts, the strike notional for physical products. Read `protocol_fee_bps_premium` and `protocol_fee_bps_volume` from `GET /api/v1/venue`.
- At expiry the maker receives `ceil(min(intrinsic, W) * quantity ...)` in quote atoms, capped at `C`, and the taker receives the rest. `intrinsic` is `max(S - lower_strike, 0)` for a call spread, `max(upper_strike - S, 0)` for a put spread and `max(strike - S, 0)` for a cash put, where `S` is the settlement price.

Worked numbers are in [`../reference/taker-api.md`](../reference/taker-api.md#economics-of-spreads-and-cash-puts).

### 3. Accept and sign the sponsored transaction

Name the winning quote by `order_id`:

```json
{
  "type": "AcceptQuote",
  "data": {
    "request_id": "<uuid>",
    "rfq_id": "<uuid>",
    "maker": "<maker owner pubkey, base58>",
    "order_id": "0x<64 hex>"
  }
}
```

The server locks the quote and builds a transaction to sign:

```json
{
  "type": "SponsoredTxToSign",
  "data": {
    "order_id": "0x<64 hex>",
    "tx_base64": "<base64 v0 VersionedTransaction>",
    "last_valid_block_height": 312456789,
    "signature_deadline": 1710000040
  }
}
```

`last_valid_block_height` is the block height the transaction's blockhash is valid through. `tx_base64` is a v0 `VersionedTransaction` with two signature slots. The keeper is the fee payer and co-signer in slot 0. The taker signs slot 1 as the collateral authority. Steps (any language):

1. Base64-decode `tx_base64` to bytes.
2. Verify the transaction before signing (see below).
3. Sign with the taker wallet key. Add the signature to slot 1. Do not touch slot 0, re-order accounts or change the message.
4. Re-serialize the partially signed transaction and base64-encode it.
5. Submit before `signature_deadline` (Unix seconds):

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": { "request_id": "<uuid>", "order_id": "0x<64 hex>", "tx_base64": "<signed, base64>" }
}
```

`AcceptQuote` and `SubmitSignedSponsoredTx` are answered with `CommandAccepted` or `RequestError` carrying the same `request_id`. A failed submission also says `delivery`: `not_forwarded` (this request was not sent) or `unknown` (it may have been). Treat the order as not executed only if every submission you made returned `not_forwarded`; otherwise check `GetOrderStatus` first.

> The wallet must support versioned (v0) transaction signing. Browser wallets that only sign legacy transactions will not work. For headless and bot takers, sign the raw message bytes with the keypair directly (see below). If a wallet cannot sign arbitrary bytes at all, WebSocket auth also fails; use a server-side signer.

### Sponsored transaction: raw signing

The TypeScript SDK provides a signing helper that verifies the served transaction. Before signing with any other tool, verify the program, market, product terms, quantity, price, accounts and all instructions against your accepted order. The [expected instructions](#expected-instructions) are listed below. For a custom byte-level signer you do not need to reconstruct a `VersionedTransaction`. `tx_base64` decodes to Solana's wire transaction format:

```
<shortvec(sig_count)> | sig[0..64] | sig[1..64] | … | <message bytes>
```

- `shortvec(sig_count)` is a compact-u16 (little-endian base128 varint) count of signatures. For a sponsored transaction it is one byte, `0x02`.
- Each signature slot is 64 bytes, ordered to match the message's required signers.
- Slot 0 is the keeper (fee payer), left zeroed at this stage. The keeper fills it after you submit.
- Slot 1 is the taker. It is the one slot you fill.

Check that account key 1 of the message is your pubkey and that account key 0 is the keeper, the `hot_authority` field of the program's `GlobalConfig` account (PDA seed `global_config`).

To sign:

1. Read the `shortvec` at offset 0 to get `sig_count` and its byte length `n`.
2. `msg_start = n + sig_count * 64`. The message bytes are `tx[msg_start..]`.
3. Produce a raw 64-byte Ed25519 signature over those message bytes with the taker key. Signing the message bytes directly is a valid transaction signature.
4. Write the 64 bytes into slot 1: `tx[n + 64 .. n + 128] = signature`.
5. Base64-encode the whole buffer and return it in `SubmitSignedSponsoredTx`.

Do not modify the message bytes, the account order or slot 0.

> **The maker's signature is not a signer slot.** The maker's Ed25519 signature over `order_id` rides inside the message as an Ed25519-program verify instruction that immediately precedes the open instruction. The only signer slot you touch is the taker's.

Library shortcuts:
- **Rust** (`solana-sdk`): deserialize with `bincode`, sign `tx.message.serialize()` with the taker `Keypair`, set index 1 in `tx.signatures`, or operate on the raw bytes as above.
- **Python** (`solders`): `tx = VersionedTransaction.from_bytes(raw)`, sign `bytes(tx.message)`, assign into index 1 of `tx.signatures`, re-serialize.
- **TypeScript**: `verifyAndSignTakerFill({ txBase64, quote, rfq, keeper, taker, tables, bounds, signer, makerVault })` from `@acta-markets/ts-sdk/ws` (no `@solana/web3.js`).

`verifyAndSignTakerFill` rebuilds the expected transaction from the accepted quote and the RFQ terms and refuses to sign on any mismatch. It checks the program and market PDA, the order ID recomputed from the quote (it must equal `order_id`), the maker's Ed25519 check over it right before the open instruction, the open instruction's accounts and arguments, the keeper fee payer, no extra signers or instructions, and a compute budget within `bounds`. For a spread or cash put it checks the spread open instruction with the quote-mint collateral. `keeper` is the `GlobalConfig.hot_authority` address. Pass the wallet's `signTransaction` as the helper's `signer` instead of calling it on the served transaction directly. Details: [`web-client-ts-sdk.md`](web-client-ts-sdk.md).

### Expected instructions

The keeper is the fee payer. It also pays the rent of the position and of every ATA created here. The taker pays only the wrap transfer. In order:

| # | Program | Instruction | When |
|---|---|---|---|
| 1 | Compute Budget | `SetComputeUnitLimit` | Always |
| 2 | Compute Budget | `SetComputeUnitPrice` | Always |
| 3 | Associated Token | `CreateIdempotent`: the taker's wSOL ATA | Covered call on wSOL, when wSOL falls short |
| 4 | System | `Transfer` from the taker to that ATA: collateral minus existing wSOL | Same |
| 5 | SPL Token | `SyncNative` on that ATA | Same |
| 6 | Associated Token | `CreateIdempotent`: the taker's quote ATA | Always |
| 7 | Associated Token | `CreateIdempotent`: the position's collateral ATA | Always |
| 8 | Ed25519 | Verify the maker's `quote_signing` signature over `order_id` | Always |
| 9 | Acta | The open instruction | Always |

The open instruction depends on the product and the maker:

| Quote | Instruction | Discriminant |
|---|---|---|
| Covered call or cash-secured put, ordinary maker | `OpenPosition` | `7` |
| Spread or cash put, ordinary maker | `OpenSpread` | `77` |
| Covered call or cash-secured put, vault maker | `OpenPositionForVaultAsMaker` | `42` |
| Spread or cash put, vault maker | `OpenSpreadForVaultAsMaker` | `79` |

A vault maker's transaction has no steps 3 to 5.

The data of `OpenPosition` is the discriminant, `position_type` (`u8`), then `strike`, `quantity`, `price`, `valid_until` and `nonce` (each `u64`, little-endian), then `order_id` (32 bytes). `OpenSpread` adds `upper_strike` (`u64`) at the end, and its `strike` is the lower strike; a cash put has `position_type` `3` and `strike` `0`. Compare these fields with the order. The accounts are in the IDL, `ACTA_IDL` in `@acta-markets/ts-sdk`.

### 4. Track the order

The server relays keeper progress:

| Event | Meaning |
|---|---|
| `OrderAccepted` | Signed transaction accepted and queued for execution. An exact `AcceptQuote` retry may also receive it as a liveness ACK. It does not confirm chain execution. |
| `OrderSubmitted` | Transaction sent to Solana. Carries `tx_signature`, `order_version`. |
| `OrderConfirmed` | Position opened on chain. Carries `position_pda`, `order_version`. |
| `OrderFailed` | Execution failed. Carries `reason`, `order_version`. |
| `RfqClosed` | Terminal RFQ event. On a fill it follows `OrderConfirmed`. Drop per-RFQ state only here. |

Apply updates only when `order_version` or `rfq_version` increases, and ignore stale replays.

### 5. Failure and retry

`OrderFailed.reason` is a free-form diagnostic string, not an enum. Log it and show it, but do not branch on an exact match. No reason authorizes another trade. The keeper retries its own submission failures within five attempts, and a failure never proves rollback. Reconcile with `GetOrderStatus` and wait for `RfqAvailableAgain` or `RfqClosed`. Never replay the old order.

On a signature timeout or tx-build failure, only the winning quote is discarded. Still-valid losing quotes are restored, and the RFQ reverts to active if it has not expired.

## Cancelling

```json
{ "type": "CancelRfq", "data": { "rfq_id": "<uuid>", "request_id": "<uuid>" } }
```

Success has no `CommandAccepted`: the RFQ, active or awaiting your signature, closes with `RfqClosed { reason: "taker_cancelled" }`. Cancelling an RFQ of yours that is already closed gets no reply. A failure is a `RequestError` with the same `request_id`: `RfqNotFound`, or `Generic` `order_already_submitted` once your signed transaction is queued (resolve it with `GetOrderStatus`).

## Position lifecycle

Once open, the position's collateral is locked in its own escrow. There is no margin and no early exercise. Settlement requires a finalized market.

`PositionInfo.state` is a tagged union:

| `state` | Meaning |
|---|---|
| `open` | Collateral locked, net premium paid to the taker. |
| `funded` | Physical products only. The maker deposited the settlement asset (`DepositFundsToPosition`). |
| `settled` | Market finalized after expiry and assets distributed. Carries `settlement`. |
| `liquidated` | Physical products only. An unfunded in-the-money position was closed by a permissionless liquidator, who fronts the settlement to the taker. |

For a covered call or cash-secured put that is in the money, the maker supplies the settlement asset. If it remains unfunded, a permissionless liquidator can supply that asset and receive the corresponding collateral. Locked collateral does not guarantee a liquidation deadline: completion depends on someone providing the funding.

A call spread, put spread or cash put goes from `open` straight to `settled`. Its `settlement` is `{ "type": "cash", "settlement_price", "maker_payoff", "taker_return" }` with amounts in quote atoms. A settled physical position has `{ "type": "physical", "settlement_price", "is_otm" }`.

Every position row carries `revision { slot, transaction_index, instruction_index }`, both decimals and `total_premium`, the net premium in quote atoms after the protocol fee. Track positions in two ways:

- The session receives `PositionUpdated { position, update_type }` for your positions. It holds the full row and needs no subscription. Apply a row only if its `revision` is newer than the stored one for that `pda`.
- Call `GetPositions` for an authoritative read. Every filter is required:

```json
{
  "type": "GetPositions",
  "data": {
    "request_id": "<uuid>",
    "market": { "type": "all" },
    "underlying_mint": { "type": "all" },
    "status": [],
    "min_expiry_ts": { "type": "unbounded" }
  }
}
```

Subscribing to `chain_events` additionally delivers `ChainEvent` frames such as `PositionSettled` and `PositionLiquidated`. Settlement mechanics and payoff tables are in [`../reference/protocol-flow.md`](../reference/protocol-flow.md).

## Reconnection and recovery

Subscriptions do not persist across disconnects, and the server does not replay missed events. After reconnect, re-authenticate (resume or fresh), resubscribe, then rehydrate:

```json
{ "type": "GetMyActiveRfqs", "data": { "request_id": "<uuid>" } }
{ "type": "GetOrderStatus",  "data": { "request_id": "<uuid>", "order_id": "0x<64 hex>" } }
{ "type": "GetPositions",    "data": { "request_id": "<uuid>", "market": { "type": "all" }, "underlying_mint": { "type": "all" }, "status": [], "min_expiry_ts": { "type": "unbounded" } } }
```

Lifecycle events can be redelivered after recovery. Process them idempotently by `order_id` and gate on `*_version`.

After `AuthSuccess`, reconcile `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus`. Only a successful resume of the same credential transfers ownership of an unfinished signature. For the same pending order whose signature was not sent, repeat the exact `AcceptQuote` to retrieve the signing payload. For a submitted or enqueued order, query status without replaying trading commands.

`OrderStatus` carries `{ request_id, order_id, state }`, with `state` equal to `{ "type": "pending" }`, `{ "type": "confirmed", "position_pda": "..." }` or `{ "type": "unknown" }`. It has no `order_version`; versions apply to lifecycle pushes. `unknown`, missing positions and timeouts leave the outcome unresolved. See [Delivery and recovery](../reference/taker-api.md#delivery-and-recovery).

## Invite gating (closed mainnet)

On closed mainnet the server sends `RequireInvite` after auth if the wallet is not registered, and `RfqRequest` returns `InviteRequired` until an invite is redeemed:

```json
{ "type": "RedeemInvite", "data": { "request_id": "<uuid>", "code": "abc123" } }
```

The authenticated session proves wallet ownership; no separate signature is needed. After redemption the taker receives a shareable `referral_code`. Devnet is open and needs no invite. Error codes (`invalid_code`, `code_exhausted`, `code_expired`, `code_disabled` and others) are in [`../reference/taker-api.md`](../reference/taker-api.md#invite-error-types).

## Operational defaults

| Topic | Value |
|---|---|
| Application `Ping` | Optional clock sample, for example every 30s. `Pong` carries `server_time_unix_ms`. Transport pongs maintain liveness; the default idle timeout is 90s. |
| Server WebSocket ping | Every 30s. Raw clients must answer protocol pings. |
| Reconnect backoff | Exponential with jitter, for example 1s initial, 30s cap, ±20%. |
| Clock skew | Track `offset = server_time − local_time` from `Welcome` and `Pong`. Apply it to `expires_at` and `signature_deadline`. |
| Signature deadline | `min(now + 30s, quote effective_expiry, rfq.expires_at)`. Sign promptly after `AcceptQuote`. |
| Rate limits | Concurrent unfinished RFQs: 50 per taker, 50 per source IP, 1000 platform-wide (defaults). `RateLimit` errors carry a reason code. |

`valid_until` (on-chain order validity) is the maker's concern. The taker never sets it.

## Integrating from other languages

The reference docs are the language-neutral wire spec. To build a taker client in any language you need:

1. An Ed25519 keypair and the ability to sign raw bytes (challenge auth and transaction signing).
2. A Solana library that can deserialize, partially sign and re-serialize a v0 `VersionedTransaction`, for example `solders` or `solana-py` (Python), `solana-sdk` (Rust), `@solana/web3.js` (TypeScript). Or the raw-byte path above.
3. A WebSocket client that answers protocol pings.
4. A strict JSON codec for the closed wire: required fields, explicit `null`, tagged unions.

The RFQ flow itself needs no RPC: markets, quotes, positions and the sponsored transaction arrive over WebSocket. An RPC read of the `GlobalConfig` account is needed only to learn the keeper address for transaction verification. Contact the Acta team for a Rust or Python starter if you are not on TypeScript.

## Reference

- [Taker API reference](../reference/taker-api.md): message catalogue and error variants
- [Taker wire examples](taker-wire-examples.md): complete JSON session and branch scenarios
- [Web client SDK (TypeScript)](web-client-ts-sdk.md): the same flow via `@acta-markets/ts-sdk`
- [Protocol flow](../reference/protocol-flow.md): trade lifecycle, economics, settlement, risk
- [WS common conventions](../reference/ws-common.md): units, envelopes, collateral formulas, timeouts
- [Capacity limits](../reference/caps.md): OI, notional and liability caps
- [Sandbox / Devnet](../reference/sandbox.md): endpoints, faucets, program addresses
- [FAQ](../reference/faq.md)
