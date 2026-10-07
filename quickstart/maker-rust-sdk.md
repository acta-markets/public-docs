# Acta Rust Maker SDK

[`acta-maker-sdk`](https://crates.io/crates/acta-maker-sdk) is the Rust client for the maker WebSocket protocol. It handles auth, reconnects, request/response correlation, and lifecycle bookkeeping.

For bots, the managed client owns the connection, re-authentication, subscription replay, state reads before readiness, and message receiver. `WsClient` is the raw client for tests, recorded sessions, or custom auth.

For JSON-layer integrations, see [`maker-quickstart.md`](maker-quickstart.md). Messages are in [`../reference/maker-api.md`](../reference/maker-api.md); wire conventions are in [`../reference/ws-common.md`](../reference/ws-common.md). Runnable examples: [github.com/acta-markets/rust-maker-sdk](https://github.com/acta-markets/rust-maker-sdk).

---

## Installation

```toml
[dependencies]
acta-maker-sdk = { version = "0.4.3", features = ["ws-client"] }
```

Version 0.4.3 includes canonical auth-challenge validation and market-PDA binding.
These checks are absent from 0.4.2. This guide covers ordinary makers;
vault integrations remain on the separate `feat/vaults` branch.

Quote-only integrations need `ws-client`. `chain` adds Solana instruction builders; `chain-rpc` adds on-chain reads.

---

## Connection

This fragment runs inside an async function returning `Result`. Supply `url: String`, `hello: HelloData`, `subscribe_data: SubscribeData`, `maker_owner: [u8; 32]`, and `quote_signing_secret: [u8; 32]`. The signing key must be registered for authentication and quote signing under that maker owner.

```rust
use acta_maker_sdk::{BytesSigner, encode_base58};
use acta_maker_sdk::ws::managed::{ManagedWsConfig, MakerWsEndpoint, spawn_managed_ws};
use std::sync::Arc;

let signer = Arc::new(BytesSigner::from_secret(quote_signing_secret));
let maker_owner_base58 = encode_base58(&maker_owner);

let quote_config = ManagedWsConfig::new(url.clone(), hello.clone(), signer.clone())
    .with_auth_pubkey(maker_owner_base58.clone())
    .with_cancel_on_disconnect(true)
    .low_latency()
    .with_initial_subscribe(subscribe_data);

let quote_handle = spawn_managed_ws(quote_config)?;
let mut messages = quote_handle.subscribe_messages(); // Consume recovery before quoting.

let data_config = ManagedWsConfig::new(url, hello, signer.clone())
    .with_auth_pubkey(maker_owner_base58)
    .with_endpoint(MakerWsEndpoint::Data);

let data_handle = spawn_managed_ws(data_config)?;
```

`url` accepts `http(s)://` and `ws(s)://` interchangeably. The default endpoint appends `/maker`; `with_endpoint(MakerWsEndpoint::Data)` appends `/maker/data`. `hello` carries the protocol version (use `WS_PROTOCOL_VERSION`), opt-in `features`, and `client_name` / `client_version` strings.

The signer is an `Arc<dyn SignerLike + Send + Sync>`. It signs the auth challenge with the registered key; `.with_auth_pubkey(...)` identifies the maker owner. Use that same owner in `QuoteBuilder::maker_owner`. `ManagedWsConfig::new_async` takes an `AsyncSignerLike` for HSM- or KMS-backed keys. `BytesSigner` holds an ed25519 keypair in memory and zeroes it on drop.

The managed quote client requests cancel-on-disconnect (COD) by default: the server cancels the connection's active and retained non-winning quotes on disconnect. Selected/executing orders remain obligations. To disable COD, set `.with_cancel_on_disconnect(false)` and omit `cancel_on_disconnect` from `HelloData.features`; the setter preserves manually supplied features. The data client does not add this feature automatically. If a required feature is absent from `Welcome.enabled_features`, the session terminates with `FeatureUnsupported`.

`low_latency()` uses 2 s ping/pong deadlines, a 1 s write timeout and larger buffers. The default gap policy is `EndpointDefault`: reconnect on inbound gaps for Quote, surface them for Data. `low_latency()` selects `Reconnect` on either endpoint.

Readiness always includes successful `GetMmSummary`, `GetActiveRfqs` and `GetMyQuotes { scope: Live }`. Quote first reconciles its desired subscription target with the server and waits for acknowledgements. `initial_subscribe` seeds the desired subscription target; its `request_id` is not sent on the wire, and recovery uses fresh request IDs. Dynamic subscription changes update the desired target after acknowledgement.

Resume can restore a server-side mint scope. The SDK explicitly sends both mint lists, including empty lists, to replace it with its local target. `Ready` identifies a connection epoch whose required reads completed; the strategy must consume them and reconcile its own unresolved orders before resuming risk-taking. These reads are not an atomic execution snapshot.

---

## Quoting

After applying recovery, retain that `recovery_epoch: u64` with the strategy state and build quotes from its `RfqBroadcast`. `RfqBinding` verifies the market PDA and terms under the configured Acta program before deriving the preimage and wire message. Use `from_broadcast_for_program` for a different locally configured deployment; never take that program ID from the RFQ:

```rust
use acta_maker_sdk::{AtomicNonceGenerator, Nonce, Price, QuoteExpiry, RfqBinding};
use acta_maker_sdk::ws::types::ClientMessage;

static NONCE_GEN: AtomicNonceGenerator = AtomicNonceGenerator::new();

// `rfq` is the current borrowed RfqBroadcastMessage; `price` is your premium.
// Choose `valid_until: QuoteExpiry` within the market and settlement deadlines.
let quote = RfqBinding::from_broadcast(rfq)?
    .quote()
    .maker_owner(maker_owner)
    .price(Price::new(price))
    .valid_until(valid_until)
    .nonce(Nonce::new(NONCE_GEN.next_u64()?))
    .sign(signer.as_ref())?;

quote_handle.send_in_epoch(ClientMessage::Quote(quote), recovery_epoch).await?;
```

`QuoteExpiry` stores whole Unix seconds; use `QuoteExpiry::from_unix_seconds(...)` or `QuoteExpiry::after(...)` rather than putting a fractional `SystemTime` into a signed order. `RfqBinding` also supports choosing a permitted strike from `order_options`. See the maintained [`managed_quote` example](https://github.com/acta-markets/rust-maker-sdk/blob/main/examples/managed_quote.rs) for receiving recovery responses, checking epochs and handling stream gaps.

`send_in_epoch` rejects if the connection has changed. Do not replace the applied recovery epoch with the latest transport epoch just before sending.

Constraints:

- `now + 100s ≤ valid_until ≤ market.expiry_ts`. The trailing 90s is reserved for settlement; the effective trading window ends at `valid_until − 90s`. Size it as `rfq.expires_at + 100s` plus a little slack, capped at market expiry.
- `is_taker_buy` is fixed at `false`. The taker is always the option writer. Setting `true` produces an `order_id` the server rejects.
- Use a fresh nonce and order ID for a new quote; the server validates the hash-bound order ID, not independent nonce uniqueness. `AtomicNonceGenerator` is safe to declare as a `static`.

---

## Quote lifecycle

Lifecycle events arrive through the managed message receiver with a connection epoch and sequence. Apply order IDs and RFQ/order versions to the corresponding local state.

| Event | Meaning |
|---|---|
| `QuoteAcknowledged` | Server accepted the quote. On a replace, includes `replaced_order_id`. |
| `QuoteRejected` | Refused; see `reason`. Bare retries fail identically. |
| `QuoteBestStatus` / `QuoteOutbid` | Current book position. |
| `QuoteRefreshRequested` | Settlement-buffer cutoff approaching. Resubmit with `valid_until ≥ min_valid_until`. |
| `QuoteSelected` | Quote locked; awaiting the taker signature. |
| `QuoteFilled` | Position opened on-chain. Carries `position_pda` and `tx_signature`. |
| `QuoteCancelled` | Removes only the listed `order_ids`; an empty list removes nothing. It is not a cancellation-command ACK. Reconcile after disconnect. |
| `QuoteExpired` | Emitted only when `quote_expired` was opted into via `Hello`. |
| `RfqAvailableAgain` | Settlement reverted; re-quote with a fresh `order_id`. |
| `RfqClosed` | Closes the auction. Keep unresolved execution obligations separately. |

Use `ReplaceQuote` for repricing. The prior quote is removed only after the replacement validates, so the swap is single-RTT. `CancelQuote` followed by `Quote` creates a gap and adds a round-trip.

```rust
quote_handle.send_in_epoch(ClientMessage::ReplaceQuote(ReplaceQuoteMessage {
    old_order_id,
    rfq_id, strike, price, valid_until, nonce, order_id, signature,
}), recovery_epoch).await?;
```

The preimage is constructed identically to a fresh `Quote` with a new `nonce` and `order_id`. The server confirms with `QuoteAcknowledged { replaced_order_id: Some(old_order_id) }`. Retain `old_order_id` until the replacement is acknowledged, since events keyed to the old id may still be in flight.

---

## Querying state

`handle.send_await(msg, timeout)` correlates request and response via `request_id` and returns `Arc<ServerMessage>`. Supported response shapes:

- `GetMyQuotes`, `GetMyTrades`, `GetMakerPositions`, `GetMmSummary`, `GetOrderStatus` — maker state; prefer the data handle
- `GetActiveRfqs`, `GetMarketsForMaker`, `GetMarketDescriptors` — recovery and instrument metadata; prefer the data handle
- `GetSubscriptions` — quote-plane session view
- `GetTokenCaps`, `GetMyCaps` — risk and per-maker limits

`send_await` also supports `Quote` / `ReplaceQuote` (by `order_id`), batches (by their order IDs), and subscription/cancellation commands (by `request_id`). `CancelQuote` waits for `CancelQuoteAck`; `QuoteCancelled` cannot satisfy it. A correlated protocol error is a response to inspect, not a successful cancellation. Commands with no supported correlation key return `SendAwaitError::NoCorrelationKey`.

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. Each mutation returns `SubscriptionUpdated` with the current state.

Query bounds: `GetMyQuotes { scope: History }` defaults to `200` historical rows and caps at `1000`; `GetMyTrades` defaults to `50` rows and caps at `200`; `GetMakerPositions` defaults to `100` rows, caps at `500`, and sets `has_more` on truncation. All three page with a keyset cursor: pass `cursor` (unix-seconds timestamp of the last row — `created_at` for quotes/positions, `confirmed_at` for trades) together with `cursor_id` (the last row's hex `order_id`, `pda`, or trade `id` respectively) — both fields or neither; ordering is second-granular with the id as tie-break. Stop when `has_more` is `false`. `GetMyQuotes { scope: Live }` is complete and unpaged; never substitute History for recovery. `GetMarketsForMaker` supports filters and has no result cap. Do not use any of these as polling paths. The WebSocket query bucket is `20` query tokens per second with a burst of `40`.

---

## Reconnection

The SDK reconnects by default. The first handshake is: TCP up -> `Hello` -> `Welcome` + server-issued `AuthRequest` -> challenge signed by `quote_signing` -> `AuthChallenge(maker_owner, signature)` -> authenticated -> subscription acknowledgement and recovery reads -> ready. If no challenge is pending, the SDK sends `StartAuth(maker_owner)`, which maker endpoints ignore. After a disconnect it sends `Hello` and tries `ResumeAuth` with the last session ID; `session_expired` clears that ID and falls back to the full signed-challenge flow on the same connection. Backoff is 250 ms initial, 5 s cap, +/-20% jitter.

`ManagedWsConfig` also bounds the failure modes: configure `connect_timeout`, `max_reconnect_attempts` (`0` means unlimited), and `reconnect_jitter_ratio`. A production supervisor should treat exhaustion or task exit as loss of readiness and restart or alert.

Trading commands are never automatically replayed across a connection boundary. Typed trading calls require `Ready` and are bound to its `connection_epoch`; non-ready calls return `NotReady`. The strategy must recompute quote validity after recovery. Raw pre-ready reads do not provide the same readiness guarantee.

The writer keeps Quote, BatchQuotes, ReplaceQuote, CancelQuote and CancelAllQuotes in one FIFO data lane. Control traffic such as ping and authentication uses a separate lane. This preserves `Quote; Cancel` ordering rather than allowing cancellation to overtake an earlier quote.

`write_timeout` bounds one socket write, not total queueing time or Core cancellation latency. COD provides a disconnect fallback, not a bounded-time halt; selected/executing obligations survive it. Queue admission or a successful socket write alone does not prove Core applied a command.

An inbound receiver that falls behind reports `ManagedReceiveError::Gap`; the configured gap policy decides whether the session reconnects. Keep the strategy's unresolved orders across this boundary and query `GetOrderStatus` after losing an ACK. `unknown`, timeout and an empty Live result are not proof of nonexecution.

The server does not replay the missed event stream. Handle duplicate lifecycle observations idempotently using order identity and entity versions.

---

## Indicative pricing

If the account is enrolled in pre-trade pricing, the server emits `IndicativePricesRequest`; the integrator replies with `IndicativePricesResponse` correlated by `request_id`. Indicative quotes are non-binding and operate under a tighter latency budget than auction quotes.

---

For typed integrations, `MakerQuoteClient` exposes trading operations and `MakerDataClient::request` returns the request's typed payload. Raw `ManagedWsHandle::send_await` returns `Arc<ServerMessage>` and requires matching the response variant.

## Errors

`NotReady` means the typed operation has no ready connection epoch; `SubscriptionPending` means a subscription mutation is already awaiting its acknowledgement. Neither requires replaying queued trading commands.

`WsClientError` covers transport failures; the reconnection loop retries. `ManagedWsError::Closed` indicates the handle's connection task is no longer running; `QueueFull` that the outbound queue or writer lane is full. `SendAwaitError::Timeout` signals the `send_await` deadline elapsed; `Disconnected` indicates the connection went down before the response arrived; `QueueFull` that the request was never queued to the socket. Terminal session states surface through `subscribe_state()` as `Closed { reason }`: `SessionReplaced`, `ProtocolVersionMismatch`, `AuthenticationRejected`, `ReconnectLimitReached`, `FeatureUnsupported`, `SessionPanicked`.

Protocol-level errors arrive as `ServerMessage::Error` (session-level) or `ServerMessage::RequestError` (per-request, correlated by `request_id`). Variants — `AuthError`, `QuoteError`, `CapError`, `RfqError` — are exhaustive enums; the full list is in [`../reference/maker-api.md`](../reference/maker-api.md).

---

## Low-level WsClient

Use the repository's [`hello_auth` example](https://github.com/acta-markets/rust-maker-sdk/blob/main/examples/hello_auth.rs) for the explicit Hello, challenge and subscription sequence. Supply required request IDs yourself and reconcile after reconnect.

Convenience methods exist for every `ClientMessage` variant. `WsClient` does not reconnect, re-authenticate, or queue outbound messages.

---

## Reference

- [Maker API reference](../reference/maker-api.md) — message catalogue and error variants
- [Maker quickstart (JSON)](maker-quickstart.md) — the same flow at the protocol level
- [Maker wire examples (JSON)](maker-wire-examples.md) — concrete request/response payloads
- [WS common conventions](../reference/ws-common.md) — units, envelopes, error codes
- [Caps reference](../reference/caps.md) — risk and quoting limits
- [FAQ](../reference/faq.md)
