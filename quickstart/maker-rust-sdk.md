# Acta Rust Maker SDK

[`acta-maker-sdk`](https://crates.io/crates/acta-maker-sdk) is the Rust client for the maker WebSocket protocol. It handles auth, reconnects, request/response correlation, and lifecycle bookkeeping.

For bots, the managed client owns the connection, re-authentication, subscription replay, state reads before readiness, and message receiver. `WsClient` is the raw client for tests, recorded sessions, or custom auth.

For JSON-layer integrations, see [`maker-quickstart.md`](maker-quickstart.md). Messages are in [`../reference/maker-api.md`](../reference/maker-api.md); wire conventions are in [`../reference/ws-common.md`](../reference/ws-common.md). Runnable examples: [github.com/acta-markets/rust-maker-sdk](https://github.com/acta-markets/rust-maker-sdk).

---

## Installation

```toml
[dependencies]
acta-maker-sdk = { version = "0.5.3", features = ["ws-client"] }
```

Version 0.5.3 targets the program deployed to devnet and mainnet on 2026-10-17. Upgrading from 0.5.0/0.5.1: `ManagedWsConfig::new` takes the maker owner, `QuoteBuilder` requires `maker_owner` and `min_valid_until`, and `MakerQuoteClient` quotes through `quote_in_epoch` / `replace_in_epoch` / `batch_quotes_in_epoch` (see the crate's `CHANGELOG.md`). This guide covers ordinary makers; vault operators use the `vault` feature, see the [vault operator guide](vault-operator-guide.md).

Quote-only integrations need `ws-client`. `chain` adds Solana instruction builders (including `chain::vault`); `chain-rpc` adds on-chain reads; `http` adds the read-only HTTP client; `vault` adds the vault operator client and implies `ws-client` and `chain`.

One key signs both the auth challenge and every quote: the `quote_signing` key stored in your on-chain Maker account. Acta registers it with the admin-gated, timelocked `RegisterMaker`; it may equal `maker_owner`.

---

## Connection

This fragment runs inside an async function returning `Result`. Supply `url: String`, `hello: HelloData`, `subscribe_data: SubscribeData`, `maker_owner: [u8; 32]`, and `quote_signing_secret: [u8; 32]`. The signing key must be registered for authentication and quote signing under that maker owner.

```rust
use acta_maker_sdk::{BytesSigner, encode_base58};
use acta_maker_sdk::ws::maker::{MakerDataClient, MakerQuoteClient};
use acta_maker_sdk::ws::managed::{ManagedWsConfig, MakerWsEndpoint};
use std::sync::Arc;

let signer = Arc::new(BytesSigner::from_secret(quote_signing_secret));
let maker_owner_base58 = encode_base58(&maker_owner);

let quote_config =
    ManagedWsConfig::new(url.clone(), hello.clone(), signer.clone(), maker_owner_base58.clone())
        .with_cancel_on_disconnect(true)
        .low_latency()
        .with_initial_subscribe(subscribe_data);

let quote_client = MakerQuoteClient::spawn(quote_config)?;
let mut messages = quote_client.subscribe_messages(); // Consume recovery before quoting.

let data_config = ManagedWsConfig::new(url, hello, signer.clone(), maker_owner_base58)
    .with_endpoint(MakerWsEndpoint::Data);

let data_client = MakerDataClient::spawn(data_config)?;
```

`url` accepts `http(s)://` and `ws(s)://` interchangeably. The default endpoint appends `/maker`; `with_endpoint(MakerWsEndpoint::Data)` appends `/maker/data`. `hello` carries the protocol version (use `WS_PROTOCOL_VERSION`), opt-in `features`, and `client_name` / `client_version` strings.

The signer is an `Arc<dyn SignerLike + Send + Sync>`. It signs the auth challenge with the registered key; the fourth argument is the maker owner, required even when the owner key signs. Use that same owner in `QuoteBuilder::maker_owner`. `ManagedWsConfig::new_async(url, hello, signer, owner)` takes an `AsyncSignerLike` for HSM- or KMS-backed keys. `BytesSigner` holds an ed25519 keypair in memory and zeroes it on drop.

The managed quote client requests cancel-on-disconnect (COD) by default: the server cancels the connection's active and retained non-winning quotes on disconnect. Selected/executing orders remain obligations. To disable COD, set `.with_cancel_on_disconnect(false)` and omit `cancel_on_disconnect` from `HelloData.features`; the setter preserves manually supplied features. The data client does not add this feature automatically. If a required feature is absent from `Welcome.enabled_features`, the session terminates with `FeatureUnsupported`.

`low_latency()` uses 2 s ping/pong deadlines, a 1 s write timeout and a larger command buffer. The inbound ring holds 4096 messages by default. The default gap policy is `EndpointDefault`: reconnect on inbound gaps for Quote, surface them for Data. `low_latency()` selects `Reconnect` on either endpoint. A gap on the quote endpoint therefore reconnects, and COD removes the connection's active and retained non-winning quotes (selected/executing orders survive); keep the receiver draining.

Readiness reads depend on the endpoint: Quote reads `GetActiveRfqs` and `GetMyQuotes { scope: Live }`; Data reads `GetMmSummary`. Quote first reconciles its desired subscription target with the server and waits for acknowledgements. `initial_subscribe` seeds the desired subscription target; its `request_id` is not sent on the wire, and recovery uses fresh request IDs. Dynamic subscription changes update the desired target after acknowledgement.

Resume can restore a server-side mint scope. The SDK explicitly sends both mint lists, including empty lists, to replace it with its local target. `Ready` identifies a connection epoch whose required reads completed; the strategy must consume them and reconcile its own unresolved orders before resuming risk-taking. These reads are not an atomic execution snapshot.

---

## Quoting

After applying recovery, retain that `recovery_epoch: u64` with the strategy state and build quotes from its `RfqBroadcast`. `RfqBinding` verifies the market PDA and terms under the configured Acta program before deriving the preimage and wire message. Use `from_broadcast_for_program` for a different locally configured deployment; never take that program ID from the RFQ:

```rust
use std::time::Duration;

use acta_maker_sdk::{AtomicNonceGenerator, Nonce, Price, QuoteExpiry, RfqBinding};

static NONCE_GEN: AtomicNonceGenerator = AtomicNonceGenerator::new();

// `rfq` is the current borrowed RfqBroadcastMessage; `price` is your premium.
// `min_validity` is `min_quote_validity_seconds` from `GET /api/v1/venue`.
let min_valid_until =
    QuoteExpiry::after(Duration::from_secs(u64::from(min_validity))).ok_or("clock")?;
let quote = RfqBinding::from_broadcast(rfq)?
    .quote()
    .maker_owner(maker_owner)
    .price(Price::new(price))
    .valid_until(valid_until)
    .min_valid_until(min_valid_until)
    .nonce(Nonce::new(NONCE_GEN.next_u64()?))
    .sign(signer.as_ref())?;

quote_client.quote_in_epoch(quote, recovery_epoch).await?;
```

With the `http` feature, `.venue(&venue)` replaces `.min_valid_until(...)`: given the `Venue` from `HttpClient::new(DEVNET_API)?.venue().await?`, it sets the floor to `now + min_quote_validity_seconds` at the current clock and refuses a `valid_until` whose `valid_until − settlement_buffer_seconds` is not after now (`QuoteBuildError::ValidUntilWithinSettlementBuffer`). Fetch the venue at startup; the builder reads the clock when `.venue` is called. `DEVNET_API`, `DEVNET_WS`, `MAINNET_API` and `MAINNET_WS` are the public origins.

Units: `RfqBroadcast.quantity` and `QuoteFilled.quantity` are underlying atoms; `Price` and `Strike` are 1e9-scaled per underlying token, independent of decimals. Mint decimals come only from `GetMarketDescriptors` on `/maker/data` (`underlying_decimals`, `quote_decimals` per `market_pda`); `MarketDecimals::observe(&message)` caches them and `get(market_pda)` returns them. `total_premium_quote_atoms(price, quantity, underlying_decimals, quote_decimals)` is the fill's gross premium (before protocol fees) in quote atoms with the contract's rounding; `Price::as_decimal()`, `Strike::as_decimal()` and `Quantity::to_units(decimals)` are for display.

`QuoteExpiry` stores whole Unix seconds; use `QuoteExpiry::from_unix_seconds(...)` or `QuoteExpiry::after(...)` rather than putting a fractional `SystemTime` into a signed order. `RfqBinding` also supports choosing a permitted strike from `order_options`. See the crate's `managed_quote` example (`cargo run --example managed_quote --features ws-client,http` from the 0.5.3 crate sources; API reference on [docs.rs](https://docs.rs/acta-maker-sdk)) for receiving recovery responses, checking epochs and handling stream gaps.

`quote_in_epoch`, `replace_in_epoch` and `batch_quotes_in_epoch` (and their `try_` forms) return `NotReady` if the connection has changed. Do not replace the applied recovery epoch with the latest transport epoch just before sending.

Constraints:

- `now + min_quote_validity_seconds ≤ valid_until ≤ market.expiry_ts`, with `min_quote_validity_seconds` from `GET /api/v1/venue`. `QuoteBuilder` checks both bounds before signing (`ValidUntilTooSoon`, `ValidUntilAfterMarketExpiry`). The trailing `settlement_buffer_seconds` is reserved for settlement; the effective trading window ends at `valid_until − settlement_buffer_seconds`. Size it as `rfq.expires_at + min_quote_validity_seconds` plus a little slack, capped at market expiry.
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
| `QuoteFilled` | Position opened on-chain. Carries `market_pda`, `expiry_ts`, `position_type`, `position_pda` and `tx_signature`, so it can be attributed without the RFQ. |
| `QuoteCancelled` | Removes only the listed `order_ids`; an empty list removes nothing. It is not a cancellation-command ACK. Reconcile after disconnect. |
| `QuoteExpired` | Emitted only when `quote_expired` was opted into via `Hello`. |
| `RfqAvailableAgain` | Settlement reverted; re-quote with a fresh `order_id`. |
| `RfqClosed` | Closes the auction. Keep unresolved execution obligations separately. |

Use `ReplaceQuote` for repricing. The prior quote is removed only after the replacement validates, so the swap is single-RTT. `CancelQuote` followed by `Quote` creates a gap and adds a round-trip.

```rust
// `quote` is a freshly signed QuoteMessage with a new nonce.
quote_client.replace_in_epoch(quote.into_replacement(old_order_id), recovery_epoch).await?;
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

Protocol-level errors arrive as `ServerMessage::Error` (session-level) or `ServerMessage::RequestError` (per-request, correlated by `request_id`), both carrying one `#[non_exhaustive]` `ServerError`. Codes without a typed variant are `ServerError::Generic { code, message }`; match on `code`. Quote outcomes arrive as `QuoteRejected { reason }`; besides validation reasons, `RateLimited` means the quote bucket is exhausted (the session stays up, retry later) and `TradingPaused` means the venue is paused. The full list is in [`../reference/maker-api.md`](../reference/maker-api.md).

Server-initiated closes carry a close code: `4000 idle_timeout`, `4001 auth_timeout`, `4002 slow_consumer`, `4003 rate_limited`, `4004 session_replaced`, `4005 hello_timeout`. The managed client logs the frame and reports it as `ManagedWsEvent::Disconnected { close: Some(ServerClose { code, reason }) }`, including when an `AuthError` / `Error` with the same code precedes the close; `close_code()` decodes it to `Option<CloseCode>`. `WsClient::server_close()` returns it after `next()` ends. The managed client reconnects after all of them except session replacement, which terminates it. Some server closes carry no code (blacklisted or unregistered maker, expired session, parse-error or auth-attempt limits, server shutdown); then `close` is `None` and the preceding `Error` / `AuthError` usually names the cause. Every opened connection ends with exactly one `Disconnected`, before `Terminated` when the client stops. Codes 4900 (`pong_timeout`) and 4901 (`auth_error`) are the TypeScript client's own drops and never come from the server.

---

## Key rotation

Rotate the quote-signing key with `UpdateMakerQuoteSigning`, signed by the maker owner; the new key needs no SOL:

```rust
use acta_maker_sdk::chain::ix::{UpdateQuoteSigningIxArgs, build_update_quote_signing_ixs};

let ixs = build_update_quote_signing_ixs(
    &program_id,
    &UpdateQuoteSigningIxArgs { maker_owner, new_quote_signing },
);
// Send `ixs` in a transaction signed by `maker_owner` (feature `chain`).
```

Once it lands, restart the quote and data sessions with the new signer; the owner passed to `ManagedWsConfig::new` and `QuoteBuilder::maker_owner` stays the same. Rotate immediately if a quote-signing key may be compromised.

---

## Low-level WsClient

Use the crate's `hello_auth` example (`cargo run --example hello_auth --features ws-client`) for the explicit Hello, challenge and subscription sequence. Supply required request IDs yourself and reconcile after reconnect.

Convenience methods exist for every `ClientMessage` variant. `WsClient` does not reconnect, re-authenticate, or queue outbound messages.

---

## Reference

- [Maker API reference](../reference/maker-api.md) — message catalogue and error variants
- [Maker quickstart (JSON)](maker-quickstart.md) — the same flow at the protocol level
- [Maker wire examples (JSON)](maker-wire-examples.md) — concrete request/response payloads
- [WS common conventions](../reference/ws-common.md) — units, envelopes, error codes
- [Caps reference](../reference/caps.md) — risk and quoting limits
- [FAQ](../reference/faq.md)
