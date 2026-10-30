# Acta Rust Maker SDK

[`acta-maker-sdk`](https://crates.io/crates/acta-maker-sdk) is the Rust client for the maker WebSocket protocol. It handles auth, reconnects, request/response correlation, and lifecycle bookkeeping.

For bots, the managed client owns the connection, re-authentication, subscription replay, state reads before readiness, and message receiver. `WsClient` is the raw client for tests, recorded sessions, or custom auth.

For JSON-layer integrations, see [`maker-quickstart.md`](maker-quickstart.md). Messages are in [`../reference/maker-api.md`](../reference/maker-api.md); wire conventions are in [`../reference/ws-common.md`](../reference/ws-common.md). Runnable examples: [github.com/acta-markets/rust-maker-sdk](https://github.com/acta-markets/rust-maker-sdk).

---

## Installation

```toml
[dependencies]
acta-maker-sdk = { version = "0.6.0", features = ["ws-client"] }
```

Version 0.6.0 speaks WS protocol `1.0.0` (`WS_PROTOCOL_VERSION`). The wire is closed: every field is present, absent facts are explicit `null`, and decoding rejects unknown message types, error codes, enum values and missing fields. This guide covers ordinary makers; vault operators use the `vault` feature, see the [vault operator guide](vault-operator-guide.md).

Quote-only integrations need `ws-client`. `chain` adds Solana instruction builders (including `chain::vault`); `chain-rpc` adds on-chain reads; `http` adds the read-only HTTP client; `vault` adds the vault operator client and implies `ws-client` and `chain`.

One key signs both the auth challenge and every quote: the `quote_signing` key stored in your on-chain Maker account. Acta registers it with the admin-gated, timelocked `RegisterMaker`; it may equal `maker_owner`.

---

## Connection

This fragment runs inside an async function returning `Result`. Supply `url: String`, `maker_owner: [u8; 32]`, and `quote_signing_secret: [u8; 32]`. The signing key must be registered for authentication and quote signing under that maker owner.

```rust
use acta_maker_sdk::{
    Address32, BytesSigner, PositionType, ProductSet, WS_PROTOCOL_VERSION,
};
use acta_maker_sdk::ws::maker::{MakerDataClient, MakerQuoteClient};
use acta_maker_sdk::ws::managed::{ManagedWsConfig, MakerWsEndpoint};
use acta_maker_sdk::ws::types::{HelloData, OnDisconnect, SubscribeData, WsChannel};
use std::sync::Arc;

let signer = Arc::new(BytesSigner::from_secret(quote_signing_secret));
let owner = Address32::new(maker_owner);

let hello = HelloData {
    protocol_version: WS_PROTOCOL_VERSION.to_string(),
    on_disconnect: OnDisconnect::CancelQuotes,
    client_name: "my-bot".to_string(),
    client_version: env!("CARGO_PKG_VERSION").to_string(),
};

let quote_config = ManagedWsConfig::new(url.clone(), hello.clone(), signer.clone(), owner)
    .low_latency()
    .with_initial_subscribe(SubscribeData {
        products: ProductSet::new(&[
            PositionType::CoveredCall,
            PositionType::CashSecuredPut,
            PositionType::CallSpread,
            PositionType::PutSpread,
            PositionType::CashPut,
        ])?,
        request_id: uuid::Uuid::new_v4(),
        channels: vec![WsChannel::Rfqs],
        underlying_mints: vec![],
        quote_mints: vec![],
    });

let quote_client = MakerQuoteClient::spawn(quote_config)?;
let mut messages = quote_client.subscribe_messages(); // Consume recovery before quoting.

let data_config = ManagedWsConfig::new(url, hello, signer.clone(), owner)
    .with_endpoint(MakerWsEndpoint::Data);

let data_client = MakerDataClient::spawn(data_config)?;
```

`url` accepts `http(s)://` and `ws(s)://` interchangeably. The default endpoint appends `/maker`; `with_endpoint(MakerWsEndpoint::Data)` appends `/maker/data`. `ProductSet::ALL` selects all five products.

`HelloData` has four required fields: `protocol_version` (use `WS_PROTOCOL_VERSION`, exactly `1.0.0`), `on_disconnect`, `client_name` and `client_version`. A different protocol version gets `unsupported_protocol_version` and a close with code 4006; the managed client terminates with `UnsupportedProtocolVersion`. There is no feature negotiation.

`on_disconnect` is `CancelQuotes` or `KeepQuotes`. With `CancelQuotes` the server cancels the connection's active and retained non-winning quotes on disconnect. Selected/executing orders remain obligations. With `KeepQuotes` resting quotes stay live while the maker is offline. Use the same `hello` for the data client; it owns no quotes.

The signer is an `Arc<dyn SignerLike + Send + Sync>`. It signs the auth challenge with the registered key; the fourth argument is the maker owner as an `Address32`, required even when the owner key signs. Use that same owner in `QuoteBuilder::maker_owner`. `ManagedWsConfig::new_async(url, hello, signer, owner)` takes an `AsyncSignerLike` for HSM- or KMS-backed keys. `BytesSigner` holds an ed25519 keypair in memory and zeroes it on drop.

`low_latency()` uses 2 s ping/pong deadlines, a 1 s write timeout and a larger command buffer. The inbound ring holds 4096 messages by default. The default gap policy is `EndpointDefault`: reconnect on inbound gaps for Quote, surface them for Data. `low_latency()` selects `Reconnect` on either endpoint. A gap on the quote endpoint therefore reconnects, and with `CancelQuotes` the disconnect removes the connection's active and retained non-winning quotes (selected/executing orders survive); keep the receiver draining.

Readiness reads depend on the endpoint:

- Quote: `GetSubscriptions`, then `Unsubscribe` for channels the server holds and the client does not want, then `Subscribe` with the desired target and a wait for `SubscribeAck`. After that it reads `GetActiveRfqs` and `GetMyQuotes { scope: Live }`. A Live response with `has_more` set fails the session attempt, and the client retries.
- Data: `GetMmSummary`.

`initial_subscribe` seeds the desired subscription target; its `request_id` is not sent on the wire, and recovery uses fresh request IDs. Dynamic subscription changes update the desired target after acknowledgement.

Resume can restore a server-side mint scope. The SDK sends `Subscribe` with both mint lists, including empty lists, to replace it with its local target. `Ready` identifies a connection epoch whose required reads completed; the strategy must consume them and reconcile its own unresolved orders before resuming risk-taking. These reads are not an atomic execution snapshot.

---

## Quoting

After applying recovery, retain that `recovery_epoch: u64` with the strategy state and build quotes from its `RfqBroadcast`. `RfqBinding` verifies the market PDA and terms under the configured Acta program before deriving the preimage and wire message. `from_broadcast` pins `ACTA_PROGRAM_ID`, which is the same on devnet and mainnet; `program_id` from `GET /api/v1/venue` is authoritative. Use `from_broadcast_for_program` for a different locally configured deployment; never take that program ID from the RFQ:

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

The builder takes the quote's `terms` and `side` from the RFQ. `QuoteBuilder::sign` selects the 182-byte layout for covered calls and cash-secured puts and the 190-byte layout for spreads and cash puts. `RfqBinding` also supports choosing a permitted strike from `order_options` for covered calls and cash-secured puts; spreads and cash puts quote exactly the RFQ's `terms`. A `buy` side is not executable and is rejected with `taker_buy_unsupported`.

### Spreads and cash puts

Spreads and cash puts settle in cash in the quote mint. `SpreadTerms::premium` returns the collateral `C`, the gross premium, the protocol fee, the net premium and `prefunded_quote`, all in quote atoms. It rejects a price outside `0 < price < width` and a gross premium that is zero or not below `C`. The fee rates come from the venue. `prefunded_quote` is `C` minus the net premium: what the open instruction needs beyond the premium it pays first. A wallet taker's RFQ is still [checked against the full `C`](../reference/taker-api.md#rfqrequest-taker---server).

```rust
use acta_maker_sdk::spreads::{SpreadTerms, SpreadType};

// `rfq` is the borrowed RfqBroadcastMessage; `underlying_decimals` and
// `quote_decimals` come from MarketDecimals; `venue` is the http Venue.
let chain = rfq.terms.to_chain(); // a cash put is the put spread (0, K)
let Some(upper) = rfq.terms.upper_strike() else { return Ok(()) }; // physical terms
let terms = SpreadTerms {
    kind: SpreadType::try_from(u8::from(chain.position_type()))?,
    lower_strike: chain.strike().value(),
    upper_strike: upper.value(),
    quantity: rfq.quantity.value(),
    quote_decimals,
    underlying_decimals,
};
let premium = terms.premium(
    price,
    venue.protocol_fee_bps_premium,
    venue.protocol_fee_bps_volume,
)?;
// premium.collateral, .gross_premium, .protocol_fee, .net_premium, .prefunded_quote

let quote = RfqBinding::from_broadcast_for_program(rfq, venue.program_id.into())?
    .quote()
    .maker_owner(maker_owner)
    .price(Price::new(price))
    .valid_until(valid_until)
    .venue(&venue)
    .nonce(Nonce::new(NONCE_GEN.next_u64()?))
    .sign(signer.as_ref())?;
quote_client.quote_in_epoch(quote, recovery_epoch).await?;
```

The runnable version is the crate's `spread_quote` example (`cargo run --example spread_quote --features ws-client,http`). It subscribes to `CallSpread`, `PutSpread` and `CashPut`. Its price is a placeholder fraction of the width; use your own model. `SpreadTerms::payoff` and `settle` compute the cash result at a settlement price.

For covered calls and cash-secured puts, `physical_premium(price, strike, quantity, underlying_decimals, quote_decimals, premium_bps, volume_bps)` returns the gross premium, protocol fee and net premium with the contract's rounding.

### Units

`RfqBroadcast.quantity` and `QuoteFilled.quantity` are underlying atoms; `Price` and `Strike` are 1e9-scaled per underlying token, independent of decimals. Mint decimals come only from `GetMarketDescriptors` on `/maker/data` (`underlying_decimals`, `quote_decimals` per `market_pda`); `MarketDecimals::observe(&message)` caches them and `get(market_pda)` returns them. `total_premium_quote_atoms(price, quantity, underlying_decimals, quote_decimals)` is the fill's gross premium (before protocol fees) in quote atoms with the contract's rounding; `Price::as_decimal()`, `Strike::as_decimal()` and `Quantity::to_units(decimals)` are for display.

`QuoteExpiry` stores whole Unix seconds; use `QuoteExpiry::from_unix_seconds(...)` or `QuoteExpiry::after(...)` rather than putting a fractional `SystemTime` into a signed order. See the crate's `managed_quote` example (`cargo run --example managed_quote --features ws-client,http` from the 0.6.0 crate sources; API reference on [docs.rs](https://docs.rs/acta-maker-sdk)) for receiving recovery responses, checking epochs and handling stream gaps.

`quote_in_epoch`, `replace_in_epoch` and `batch_quotes_in_epoch` (and their `try_` forms) return `NotReady` if the connection has changed. Do not replace the applied recovery epoch with the latest transport epoch just before sending.

Constraints:

- `now + min_quote_validity_seconds ≤ valid_until ≤ market.expiry_ts`, with `min_quote_validity_seconds` from `GET /api/v1/venue`. `QuoteBuilder` checks both bounds before signing (`ValidUntilTooSoon`, `ValidUntilAfterMarketExpiry`). The trailing `settlement_buffer_seconds` is reserved for settlement; the effective trading window ends at `valid_until − settlement_buffer_seconds`. Size it as `rfq.expires_at + min_quote_validity_seconds` plus a little slack, capped at market expiry.
- The taker side is `sell`: the taker is always the option writer.
- Use a fresh nonce and order ID for a new quote; the server validates the hash-bound order ID, not independent nonce uniqueness. `AtomicNonceGenerator` is safe to declare as a `static`.

---

## Quote lifecycle

Lifecycle events arrive through the managed message receiver with a connection epoch and sequence. Apply order IDs and RFQ/order versions to the corresponding local state.

| Event | Meaning |
|---|---|
| `QuoteAcknowledged` | Server accepted the quote. `replaced_order_id` is `Some` on a replace. |
| `QuoteRejected` | Refused; see `reason`. Bare retries fail identically. |
| `QuoteBestStatus` / `QuoteOutbid` | Current book position. |
| `QuoteRefreshRequested` | Settlement-buffer cutoff approaching. Resubmit with `valid_until ≥ min_valid_until`. Identified by `(rfq_id, terms)`. |
| `QuoteSelected` | Quote locked; awaiting the taker signature. |
| `QuoteFilled` | Position opened on-chain. Carries `market_pda`, `expiry_ts`, `terms`, `side`, `position_pda` and `tx_signature`, so it can be attributed without the RFQ. It has no fee or net-premium field. |
| `QuoteCancelled` | Removes only the listed `order_ids`; an empty list removes nothing. It is not a cancellation-command ACK. Reconcile after disconnect. |
| `QuoteExpired` | Always sent when a quote expires. `reason` is `Expired` or `ExpiringSoon`. |
| `RfqAvailableAgain` | Settlement reverted; re-quote with a fresh `order_id`. |
| `RfqClosed` | Closes the auction. Keep unresolved execution obligations separately. |

Use `ReplaceQuote` for repricing. The prior quote is removed only after the replacement validates, so the swap is single-RTT. `CancelQuote` followed by `Quote` creates a gap and adds a round-trip.

```rust
// `quote` is a freshly signed QuoteMessage with a new nonce.
quote_client.replace_in_epoch(quote.into_replacement(old_order_id), recovery_epoch).await?;
```

The preimage is constructed identically to a fresh `Quote` with a new `nonce` and `order_id`. The server confirms with `QuoteAcknowledged { replaced_order_id: Some(old_order_id) }`. Retain `old_order_id` until the replacement is acknowledged, since events keyed to the old id may still be in flight.

---

## Positions and money

Position rows (`MakerPositionInfo`, and `PositionInfo` in `PositionUpdated`) carry:

- `state`: `Open`, `Funded`, `Liquidated`, or `Settled { settlement }`. `settlement` is `Physical { settlement_price, is_otm }` or `Cash { settlement_price, maker_payoff, taker_return }`.
- `terms` and `side`.
- `total_premium`: the net premium the taker received, in quote atoms after the protocol fee. This is the authoritative figure for the position, not `QuoteFilled`. The maker's cost is the gross premium, and the fee is that cost minus `total_premium`.
- `revision`: `PositionRevision { slot, transaction_index, instruction_index }`. It orders lexicographically. Keep a row only when `incoming.revision > stored.revision` for the same PDA.

`PositionUpdated` carries the full row and an `update_type`, and is coalesced per PDA by revision. The managed data client receives it, with `TradeExecuted` and `MakerCapsUpdated`, through `subscribe_messages()` after summary recovery.

Spreads and cash puts reach only `Open` and `Settled`. They never produce `Funded` or `Liquidated` rows.

---

## Querying state

`handle.send_await(msg, timeout)` correlates request and response via `request_id` and returns `Arc<ServerMessage>`. Supported response shapes:

- `GetMyQuotes`, `GetMyTrades`, `GetMakerPositions`, `GetMmSummary`, `GetOrderStatus` — maker state; prefer the data handle
- `GetActiveRfqs`, `GetMarketsForMaker`, `GetMarketDescriptors` — recovery and instrument metadata; prefer the data handle
- `GetSubscriptions` — quote-plane session view
- `GetTokenCaps`, `GetMyCaps` — risk and per-maker limits

`send_await` also supports `Quote` / `ReplaceQuote` (by `order_id`), batches (by their order IDs), and subscription/cancellation commands (by `request_id`). `CancelQuote` waits for `CancelQuoteAck`; `QuoteCancelled` cannot satisfy it. A correlated protocol error is a response to inspect, not a successful cancellation. Commands with no supported correlation key return `SendAwaitError::NoCorrelationKey`.

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. `AddMints` and `RemoveMints` take both mint lists, and an empty list means no change. Each mutation returns `SubscriptionUpdated` with the current channels and both mint lists.

### Filters and pages

Queries take explicit filters. There are no defaults and no optional fields.

- Single-value filters are `QueryFilter::All {}` or `QueryFilter::Only { value }`.
- Time bounds are `TimeBound::Unbounded {}` or `TimeBound::Bounded { value }`.
- Mint lists and `status: Vec<PositionStatus>` use an empty list for every value.
- Pages take a required `limit` and a `PageCursor::Start {}` or `PageCursor::After { at, id }`. `GetMyQuotes` takes `MakerQuoteScope::Live {}` or `History { limit, cursor }`.

| Request | `limit` range | Cursor `at` | Cursor `id` |
|---|---|---|---|
| `GetMakerPositions` | `1..=500` | `created_at` | PDA |
| `GetMyQuotes` History | `1..=1000` | `created_at` | `order_id` |
| `GetMyTrades` | `1..=200` | `confirmed_at` | trade `id` |

A limit outside the range is rejected with `invalid_limit`; the server does not clamp it. Pages run newest first by (created second, id). `has_more` is `true` while rows remain. `GetMyQuotes { scope: Live }` is complete and unpaged; never substitute History for recovery. `GetMarketsForMaker` supports filters and has no result cap. Do not use any of these as polling paths. The WebSocket query bucket is `20` query tokens per second with a burst of `40`.

`MakerDataClient::all_pages(request, timeout)` pages `GetMakerPositions`, `GetMyTrades` and `GetMyQuotes` History to the end at the server's largest page size. It ignores the request's own `limit` and cursor. Pages after the first are appended.

```rust
use acta_maker_sdk::ws::types::{GetMakerPositionsMessage, PageCursor, QueryFilter, TimeBound};
use std::time::Duration;

let positions = data_client
    .all_pages(
        GetMakerPositionsMessage {
            request_id: uuid::Uuid::new_v4(),
            market: QueryFilter::All {},
            underlying_mint: QueryFilter::All {},
            status: vec![],
            min_expiry_ts: TimeBound::Unbounded {},
            limit: 500,
            cursor: PageCursor::Start {},
        },
        Duration::from_secs(5),
    )
    .await?;
// positions.positions, positions.unrenderable
```

### Recovery after a reconnect

The managed clients run the readiness reads and then report `Ready`. The strategy does the rest:

1. Wait for `Ready` and take the connection epoch. Consume the quote-plane `ActiveRfqs` and `MyQuotes` responses and the data-plane `MmSummary` from the receivers.
2. Apply `MmSummary.positions` by revision. If `positions_has_more` is `true`, page the rest with `all_pages`. The summary holds the newest 500 positions across all statuses.
3. Book money from the rows' `total_premium`.
4. Reconcile each unfinished local order that is not in the Live set. Query `GetOrderStatus` one order at a time, and stop at the first rejection:

```rust
use acta_maker_sdk::ws::types::{GetOrderStatusMessage, OrderExecutionState};

for order_id in unfinished {
    let status = data_client
        .request(
            GetOrderStatusMessage { request_id: uuid::Uuid::new_v4(), order_id },
            Duration::from_secs(5),
        )
        .await?; // a rejected request returns Err; stop and fix the cause
    match &status.state {
        OrderExecutionState::Confirmed { position_pda } => { /* book from the position row */ }
        OrderExecutionState::Pending | OrderExecutionState::Unknown => { /* keep the obligation */ }
    }
}
```

`pending` and `unknown` keep the order as an open obligation. `unknown`, a timeout and an empty Live set do not prove that an order did not execute.

5. Quote again only with a new signed quote in the recovered epoch.

---

## Reconnection

The SDK reconnects by default. The first handshake is: TCP up -> `Hello` -> `Welcome` + server-issued `AuthRequest` -> challenge signed by `quote_signing` -> `AuthChallenge(maker_owner, signature)` -> authenticated -> subscription acknowledgement and recovery reads -> ready. If no challenge is pending, the SDK sends `StartAuth(maker_owner)`, which maker endpoints ignore. After a disconnect it sends `Hello` and tries [`ResumeAuth`](../reference/maker-api.md#resume) with the last session ID; `session_expired` clears that ID and falls back to the full signed-challenge flow on the same connection. Backoff is 250 ms initial, 5 s cap, +/-20% jitter.

`ManagedWsConfig` also bounds the failure modes: configure `connect_timeout`, `max_reconnect_attempts` (`0` means unlimited), and `reconnect_jitter_ratio`. A production supervisor should treat exhaustion or task exit as loss of readiness and restart or alert.

Trading commands are never automatically replayed across a connection boundary. Typed trading calls require `Ready` and are bound to its `connection_epoch`; non-ready calls return `NotReady`. The strategy must recompute quote validity after recovery. Raw pre-ready reads do not provide the same readiness guarantee.

The writer keeps Quote, BatchQuotes, ReplaceQuote, CancelQuote and CancelAllQuotes in one FIFO data lane. Control traffic such as ping and authentication uses a separate lane. This preserves `Quote; Cancel` ordering rather than allowing cancellation to overtake an earlier quote.

`write_timeout` bounds one socket write, not total queueing time or how long the server takes to apply a cancel. `OnDisconnect::CancelQuotes` provides a disconnect fallback, not a bounded-time halt; selected/executing obligations survive it. Queue admission or a successful socket write alone does not prove the server applied a command.

An inbound receiver that falls behind reports `ManagedReceiveError::Gap`; the configured gap policy decides whether the session reconnects. Keep the strategy's unresolved orders across this boundary and query `GetOrderStatus` after losing an ACK. `unknown`, timeout and an empty Live result are not proof of nonexecution.

The server does not replay the missed event stream. Handle duplicate lifecycle observations idempotently using order identity and entity versions.

---

## Indicative pricing

Indicative pricing covers covered calls and cash-secured puts. If the account is enrolled in pre-trade pricing, the server emits `IndicativePricesRequest`; the integrator replies with `IndicativePricesResponse` correlated by `request_id`. Indicative quotes are non-binding and operate under a tighter latency budget than auction quotes.

---

For typed integrations, `MakerQuoteClient` exposes trading operations and `MakerDataClient::request` returns the request's typed payload. Raw `ManagedWsHandle::send_await` returns `Arc<ServerMessage>` and requires matching the response variant.

## Errors

`NotReady` means the typed operation has no ready connection epoch; `SubscriptionPending` means a subscription mutation is already awaiting its acknowledgement. Neither requires replaying queued trading commands.

`WsClientError` covers transport failures; the reconnection loop retries. `ManagedWsError::Closed` indicates the handle's connection task is no longer running; `QueueFull` that the outbound queue or writer lane is full. `SendAwaitError::Timeout` signals the `send_await` deadline elapsed; `Disconnected` indicates the connection went down before the response arrived; `QueueFull` that the request was never queued to the socket. Terminal session states surface through `subscribe_state()` as `Closed { reason }`: `Requested`, `SessionReplaced`, `UnsupportedProtocolVersion`, `AuthenticationRejected`, `ReconnectLimitReached`, `SessionPanicked`.

Protocol-level errors arrive as `ServerMessage::Error` (session-level) or `ServerMessage::RequestError` (per-request, correlated by `request_id`), both carrying one `ServerError`. Error codes are a closed `WsErrorCode` enum: `ServerError::Generic { code, message }` carries the typed code, so match on `code`. A code outside the enum fails decoding. Quote outcomes arrive as `QuoteRejected { reason, message }`; besides validation reasons, `RateLimited` means the quote bucket is exhausted (the session stays up, retry later) and `TradingPaused` means the venue is paused. `QuoteRejectReason`, `CapError` and `PositionUpdateType` are exhaustive, so a `match` without a wildcard arm is checked against the wire. The full list is in [`../reference/maker-api.md`](../reference/maker-api.md).

Server-initiated closes carry a close code: `4000 idle_timeout`, `4001 auth_timeout`, `4002 slow_consumer`, `4003 rate_limited`, `4004 session_replaced`, `4005 hello_timeout`, `4006 unsupported_protocol_version`. The managed client logs the frame and reports it as `ManagedWsEvent::Disconnected { close: Some(ServerClose { code, reason }) }`, including when an `AuthError` / `Error` with the same code precedes the close; `close_code()` decodes it to `Option<CloseCode>`. `WsClient::server_close()` returns it after `next()` ends. The managed client reconnects after all of them except session replacement and an unsupported protocol version, which terminate it. Some server closes carry no code (blacklisted or unregistered maker, expired session, parse-error or auth-attempt limits, server shutdown); then `close` is `None` and the preceding `Error` / `AuthError` usually names the cause. Every opened connection ends with exactly one `Disconnected`, before `Terminated` when the client stops. Codes 4900 (`pong_timeout`) and 4901 (`auth_error`) are the TypeScript client's own drops and never come from the server.

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
