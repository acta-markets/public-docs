# Acta Taker Quickstart

JSON-layer path for a taker: auth, discover markets, open an RFQ, accept a quote, sign the sponsored transaction, and track the resulting position. This is the language-neutral protocol walkthrough; a TypeScript SDK wraps the same flow in [`web-client-ts-sdk.md`](web-client-ts-sdk.md).

The message catalogue is in [`../reference/taker-api.md`](../reference/taker-api.md); units and envelopes are in [`../reference/ws-common.md`](../reference/ws-common.md); trade mechanics and settlement are in [`../reference/protocol-flow.md`](../reference/protocol-flow.md); onboarding is in [`../reference/sandbox.md`](../reference/sandbox.md).

A taker needs only a Solana wallet keypair. There is no registration and no on-chain setup step: any wallet can open RFQs (rate-limited, and invite-gated on closed mainnet — see [Invite gating](#invite-gating-closed-mainnet)).

---

## Endpoint

```
wss://devnet-api.acta.markets/taker
wss://beta-api.acta.markets/taker
```

## Connection and authentication

The first message from the client is `Hello`. Version compatibility is semver-based: the server accepts any client `protocol_version` that is `>= min_supported_version`; otherwise it closes the connection with `VersionMismatch`.

```json
{
  "type": "Hello",
  "data": {
    "protocol_version": "1.0.0",
    "features": [],
    "client_name": "my-app",
    "client_version": "0.1.0"
  }
}
```

The server responds with `Welcome` (`server_time_unix_ms` is for clock sync). Taker auth is **lazy**: you may authenticate immediately or defer until the first authenticated action. Unauthenticated discovery (`GetMarkets`, `GetMarketDescriptors`, `GetTokenCaps`, `GetIndicativePrices`, …) works before auth; see [`../reference/taker-api.md`](../reference/taker-api.md) "Authentication requirements".

### Fresh sign

Send `StartAuth` with your wallet pubkey; the server replies with `AuthRequest` carrying a challenge string. The taker challenge includes a `Wallet:` line:

```
Acta RFQ Authentication

Sign this message to authenticate your wallet.

Wallet: {base58_pubkey}
Nonce: {hex_32_random_bytes}
Issued At: {RFC3339_timestamp}
```

Before signing, validate the complete challenge using the [canonical auth rules](../reference/ws-common.md#what-to-sign), including the exact domain, wallet, nonce, timestamp and final newline. Reject arbitrary endpoint text. Then sign the original UTF-8 bytes with your wallet key (Ed25519, no extra prefix or hashing), base58-encode the 64-byte signature, and reply:

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

The signature is verified directly against `pubkey`. Auth must finish within 15 seconds of the challenge; the default limit allows three failed attempts and closes the connection on the fourth. On success the server sends `AuthSuccess { session_id, expires_at }` followed by `Snapshot`.

### Session resume

`AuthSuccess.expires_at` is a Unix timestamp (seconds). Persist `session_id` + `expires_at`. On reconnect, if `now < expires_at`, skip the wallet signature:

```json
{ "type": "ResumeAuth", "data": { "session_id": "<saved session_id>" } }
```

A valid session returns `AuthSuccess`; an invalid/expired/revoked one returns `AuthError { reason: "session_expired" }`, at which point fall back to the fresh-sign flow. Sessions have a 24h default TTL (sliding, capped at 7 days) and survive disconnects. A failed `ResumeAuth` counts toward the 3-attempt limit.

## Discovery

Fetch market metadata before opening an RFQ. `GetMarketDescriptors` returns the full descriptor — `size_rule`, decimals, oracle PDAs, symbols — needed to validate quantity and render prices.

```json
{ "type": "GetMarketDescriptors", "data": { "request_id": "<uuid>" } }
{ "type": "GetTokens",            "data": { "request_id": "<uuid>" } }
{ "type": "GetExpiries",          "data": { "request_id": "<uuid>" } }
```

Query-style requests require `request_id`; responses echo it. Discovery returns tradable markets only. Cache descriptors and re-read them when an authenticated session receives `ReferenceChanged`, which also covers a market passing its trading cutoff. Before auth, or without WS, read `GET /api/v1/tokens` and `GET /api/v1/markets` with `If-None-Match` (see [HTTP API](../reference/http-api.md#reference-data)).

> **Size rules.** `size_rule = { min_size, max_size, step }` is in **underlying atomic units**, keyed by underlying mint. A valid `quantity` satisfies `min_size <= quantity <= max_size` and `(quantity - min_size) % step == 0`. If a market's underlying mint has no configured rule, `RfqRequest` is rejected with `missing_size_rule_for_underlying_mint`. `GetMarketDescriptors` and `GetTokens` fail the whole request if any active market is missing a rule — partial discovery is not supported. `GetTokens` returns the same rule on each underlying token.

## The RFQ flow

```
RfqRequest -> RfqCreated -> QuoteReceived (0..N) -> AcceptQuote
  -> SponsoredTxToSign -> [sign] -> SubmitSignedSponsoredTx
  -> OrderAccepted -> OrderSubmitted -> OrderConfirmed -> RfqClosed
```

### 1. Open the RFQ

`quantity` is always in **underlying atomic units**, including for cash-secured puts. For a CSP UI that collects quote input (e.g. USDC), convert first: `quantity = round(quoteAmount * 10^underlying_decimals * 1e9 / strike)` (see [`../reference/ws-common.md`](../reference/ws-common.md) "Quantity and collateral by position type").

```json
{
  "type": "RfqRequest",
  "data": {
    "market": "<market PDA, base58>",
    "position_type": "covered_call",
    "strike": 136000000000,
    "quantity": 5000000000,
    "timeout_seconds": 30,
    "client_request_id": "<optional uuid>"
  }
}
```

`position_type` is `covered_call` or `cash_secured_put`. `client_request_id` is an optional idempotency key: repeating a request with the same key while the RFQ is active returns the same `rfq_id`. The server replies with `RfqCreated { rfq_id, rfq_version, expires_at, created_at, order_options }`, then broadcasts the RFQ to eligible makers.

### 2. Collect quotes

Makers stream `QuoteReceived`. Each quote is firm and hash-bound via `order_id`:

```json
{
  "type": "QuoteReceived",
  "data": {
    "rfq_id": "<uuid>",
    "strike": 136000000000,
    "maker": "<maker owner pubkey, base58>",
    "price": 50000000,
    "net_price": 49750000,
    "valid_until": 1710000350,
    "nonce": 42,
    "order_id": "0x<64 hex>"
  }
}
```

- `price` — gross premium per 1 underlying unit (1e9 scale). Use this and `order_id` for all order operations.
- `net_price` — display-only estimate after protocol fee. Show `net_price`; the authoritative net is computed on-chain at open.

You pick the winner — the server's "best" ranking is advisory only. Winner-take-all: one quote fills the full quantity, no partial fills.

### 3. Accept and sign the sponsored transaction

Name the winning quote by `order_id`:

```json
{
  "type": "AcceptQuote",
  "data": {
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
    "signature_deadline": 1710000040
  }
}
```

`tx_base64` is a **v0 `VersionedTransaction`**. The keeper is the fee payer and co-signer; the taker signs as the collateral authority. Steps (any language):

1. base64-decode `tx_base64` to bytes.
2. Deserialize as a v0 `VersionedTransaction`.
3. Sign with the taker wallet key. Add the signature to the taker's signer slot — **do not** replace the keeper's fee-payer signature or re-order accounts; the message bytes are fixed.
4. Re-serialize the partially-signed transaction and base64-encode it.
5. Submit before `signature_deadline` (Unix seconds):

```json
{
  "type": "SubmitSignedSponsoredTx",
  "data": { "order_id": "0x<64 hex>", "tx_base64": "<signed, base64>" }
}
```

> The wallet must support **versioned (v0) transaction signing**. Browser wallets that only sign legacy transactions will not work; for headless/bot takers, sign the raw message bytes with the keypair directly (see below). If a wallet cannot sign arbitrary bytes at all, WS auth also fails — use a server-side signer.

### Sponsored transaction: raw signing

The TypeScript SDK provides signing helpers; they do not independently verify that a server-prepared transaction matches your trading intent. Before signing, verify the program, market, quote terms, amounts, accounts and all instructions against your accepted order. For a custom byte-level signer, you do **not** need to fully reconstruct a `VersionedTransaction`. `tx_base64` decodes to Solana's wire transaction format:

```
<shortvec(sig_count)> | sig[0..64] | sig[1..64] | … | <message bytes>
```

- `shortvec(sig_count)` is a compact-u16 (little-endian base128 varint) count of signatures. For a 2-signer sponsored tx it is a single byte `0x02`.
- Each signature slot is 64 bytes, ordered to match the message's required signers.
- **Slot 0 = keeper** (fee payer / co-signer) — left zeroed at this stage; the keeper fills it *after* you submit.
- **Slot 1 = taker** — the one slot you fill.

To sign:

1. Read the `shortvec` at offset 0 → `sig_count` and its byte length `n`.
2. `msg_start = n + sig_count * 64`; the **message bytes** are `tx[msg_start..]`.
3. Produce a raw 64-byte Ed25519 signature over those message bytes with the taker key. Signing the message bytes directly *is* a valid transaction signature — no VersionedTransaction reconstruction required.
4. Write the 64 bytes into slot 1: `tx[n + 64 .. n + 128] = signature`.
5. base64-encode the whole buffer and return it in `SubmitSignedSponsoredTx`.

Do not modify the message bytes, account order, or the keeper's slot.

> **The maker's signature is not a signer slot.** The maker's Ed25519 signature over `order_id` rides inside the message as an **Ed25519-program verify instruction** (immediately preceding `open_position`), not as a transaction signature. The only tx signer slot you touch is the taker's (slot 1).

Library shortcuts:
- **Rust** (`solana-sdk`): deserialize with `bincode`, sign `tx.message.serialize()` with the taker `Keypair`, set `tx.signatures[1]`; or operate on the raw bytes as above.
- **Python** (`solders`): `tx = VersionedTransaction.from_bytes(raw)`, sign `bytes(tx.message)`, assign into `tx.signatures[1]`, re-serialize.
- **TypeScript**: `signSponsoredTxBase64({ txBase64, taker, expectedMessageBytes })` from `@acta-markets/ts-sdk/ws` (no `@solana/web3.js`).

The TypeScript helper in SDK 0.1.6 requires an independently constructed expected
message and checks it before signing. Include the accepted order, configured
program/accounts, approved blockhash, instructions and resolved ALT references;
never use bytes copied from the received transaction as the expectation. Older
SDK versions require this check in application code. The explicitly named
`signSponsoredTxBase64Unverified` is for applications that trust server-side
construction.

### 4. Track the order

The server relays keeper progress:

| Event | Meaning |
|---|---|
| `OrderAccepted` | Signed transaction accepted into Core's Enqueued state. An exact AcceptQuote retry may also receive it as a liveness ACK. It does not confirm chain execution. |
| `OrderSubmitted` | Tx sent to Solana. Carries `tx_signature`, `order_version`. |
| `OrderConfirmed` | Position opened on-chain. Carries `position_pda`, `order_version`. |
| `OrderFailed` | Settlement failed. Carries `reason`, `order_version`. |
| `RfqClosed` | Terminal RFQ event. On a fill it follows `OrderConfirmed`. Drop per-RFQ state only here. |

Apply updates only when `order_version` / `rfq_version` increases; ignore stale replays.

### 5. Failure and retry

`OrderFailed.reason` is a free-form diagnostic string, not an enum — log it and surface it, but do
not branch on an exact match. No reason authorizes another trade: the keeper retries its own
submission failures within five attempts, and a failure never proves rollback. Reconcile with
`GetOrderStatus` and wait for `RfqAvailableAgain` or `RfqClosed`; never replay the old order.

On a signature timeout or tx-build failure, only the winning quote is discarded; still-valid losing quotes are restored and the RFQ reverts to active if not expired.

## Cancelling

```json
{ "type": "CancelRfq", "data": { "rfq_id": "<uuid>", "request_id": "<uuid>" } }
```

The server responds with `RfqClosed { reason: "taker_cancelled" }`.

## Position lifecycle

Once open, the position's collateral is locked in its own escrow — no margin, no early exercise. Settlement requires a finalized market. For an ITM position, the maker supplies the settlement asset; if it remains unfunded, a permissionless liquidator can supply that asset and receive the corresponding collateral. Locked collateral does not guarantee a liquidation deadline: completion depends on someone providing the required funding.

| Status | Meaning |
|---|---|
| `open` | Collateral locked, net premium paid to taker. |
| `funded` | Maker deposited the settlement asset (`DepositFundsToPosition`). |
| `settled` | Market finalized after expiry; assets distributed by ITM/OTM outcome. |
| `liquidated` | An unfunded ITM position was closed by a permissionless liquidator, who fronts the settlement to the taker. |

Track position outcomes by subscribing to `chain_events` (the `ChainEvent(position_settled)` / `ChainEvent(position_liquidated)` pushes for your market) and by polling `GetPositions`. Note: `PositionUpdated` and the `positions` channel are **maker-only** — takers receive nothing there. Settlement mechanics and payoff tables are in [`../reference/protocol-flow.md`](../reference/protocol-flow.md).

## Reconnection and recovery

Subscriptions do not persist across disconnects, and the server does not replay missed events. After reconnect: re-auth (resume or fresh), resubscribe, then rehydrate:

```json
{ "type": "GetMyActiveRfqs", "data": { "request_id": "<uuid>" } }
{ "type": "GetOrderStatus",  "data": { "request_id": "<uuid>", "order_id": "0x<64 hex>" } }
{ "type": "GetPositions",    "data": { "request_id": "<uuid>" } }
```

Lifecycle events can be redelivered after recovery; process them idempotently by `order_id` and gate on `*_version`.

## Invite gating (closed mainnet)

On closed mainnet the server sends `RequireInvite` after auth if the wallet is not registered, and `RfqRequest` returns `InviteRequired` until an invite is redeemed:

```json
{ "type": "RedeemInvite", "data": { "request_id": "<uuid>", "code": "abc123" } }
```

The authenticated session proves wallet ownership; no separate signature is needed. After redemption the taker receives a shareable `referral_code`. Devnet is open — no invite required. Error codes (`invalid_code`, `code_exhausted`, `code_expired`, `code_disabled`, …) are in [`../reference/taker-api.md`](../reference/taker-api.md).

## Operational defaults

| Topic | Value |
|---|---|
| Application `Ping` | Optional clock sample, e.g. every 30s; `Pong` carries `server_time_unix_ms`. Transport pongs maintain liveness; the default idle timeout is 90s. |
| Server WS ping | every 30s; raw clients must answer protocol pings. |
| Reconnect backoff | Exponential with jitter, e.g. 1s initial, 30s cap, ±20%. |
| Clock skew | Track `offset = server_time − local_time` from `Welcome` / `Pong`; apply to `expires_at`, `signature_deadline`. |
| Signature deadline | `min(now + 30s, quote effective_expiry, rfq.expires_at)`. Sign promptly after `AcceptQuote`. |
| Rate limits | Concurrent unfinished RFQs: 50 per taker, 50 per source IP, 1000 platform-wide (defaults). `RateLimit` errors carry a reason code. |

Note: `valid_until` (on-chain order validity) is the maker's concern, not the taker's — the taker never sets it.

## Integrating from other languages

The reference docs are the language-neutral wire spec. To build a taker client in any language you need:

1. An Ed25519 keypair and the ability to sign raw bytes (challenge auth + tx signing).
2. A Solana library that can deserialize, partially sign, and re-serialize a **v0 `VersionedTransaction`** — e.g. `solders` / `solana-py` (Python), `solana-sdk` (Rust), `@solana/web3.js` (TS).
3. A WebSocket client that answers protocol pings.

No on-chain RPC is required for the RFQ flow itself — markets, quotes, positions, and the sponsored transaction all arrive over WS. Contact the Acta team for a Rust or Python starter if you are not on TypeScript.

## Reference

- [Taker API reference](../reference/taker-api.md) — message catalogue and error variants
- [Taker wire examples](taker-wire-examples.md) — complete JSON session + branch scenarios
- [Web client SDK (TypeScript)](web-client-ts-sdk.md) — the same flow via `@acta-markets/ts-sdk`
- [Protocol flow](../reference/protocol-flow.md) — trade lifecycle, economics, settlement, risk
- [WS common conventions](../reference/ws-common.md) — units, envelopes, collateral formulas, timeouts
- [Capacity limits](../reference/caps.md) — OI and notional caps
- [Sandbox / Devnet](../reference/sandbox.md) — endpoints, faucets, program addresses
- [FAQ](../reference/faq.md)

## Recovery after reconnect

After `AuthSuccess`, reconcile `GetMyActiveRfqs`, `GetPositions` and `GetOrderStatus`. Only a successful resume of the same credential transfers ownership of an unfinished signature. For the same pending order whose signature was not sent, repeat the exact `AcceptQuote` to retrieve the signing payload. For a submitted/enqueued order, query status without replaying trading commands.

`OrderStatus` carries `{ request_id, order_id, state }`, with `state` equal to `{ "type": "pending" }`, `{ "type": "confirmed", "position_pda": "..." }` or `{ "type": "unknown" }`. It has no `order_version`; versions apply to lifecycle pushes. `unknown`, missing positions and timeouts leave the outcome unresolved. See [Delivery & recovery](../reference/taker-api.md#delivery-and-recovery).
