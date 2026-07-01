# Acta Rust Maker SDK

[`acta-maker-sdk`](https://crates.io/crates/acta-maker-sdk) is the Rust client for the maker WebSocket protocol. It handles auth, reconnects, request/response correlation, and lifecycle bookkeeping.

Use `ManagedWs` for normal bots. It owns the connection, re-authenticates on reconnect, replays subscriptions, and exposes `broadcast::Receiver<Arc<ServerMessage>>`. `WsClient` is the raw client for tests, recorded sessions, or custom auth.

For JSON-layer integrations, see [`maker-quickstart.md`](maker-quickstart.md). Messages are in [`../reference/maker-api.md`](../reference/maker-api.md); wire conventions are in [`../reference/ws-common.md`](../reference/ws-common.md). Runnable examples: [github.com/acta-markets/rust-maker-sdk](https://github.com/acta-markets/rust-maker-sdk).

---

## Installation

```toml
[dependencies]
acta-maker-sdk = { version = "=0.1.0-beta.1", features = ["ws-client"] }
```

Quote-only integrations need `ws-client`. `chain` adds Solana instruction builders; `chain-rpc` adds on-chain reads.

---

## Connection

```rust
let config = ManagedWsConfig::new(url, hello, pubkey_base58, challenge_signer)
    .with_initial_subscribe(subscribe_data)
    .with_resync_messages(vec![GetMyQuotes, GetActiveRfqs]);

let handle = spawn_managed_ws(config);
let mut messages = handle.subscribe_messages();
```

`url` accepts `http(s)://` and `ws(s)://` interchangeably and appends `/maker` if absent. `hello` carries the protocol version (use `WS_PROTOCOL_VERSION`), opt-in `features`, and `client_name` / `client_version` strings.

`challenge_signer: Arc<dyn Fn(&str) -> Result<String, String> + Send + Sync>` runs on every auth attempt. It receives the challenge string and must return a base58 ed25519 signature. Sign with the registered `quote_signing` key; `pubkey_base58` is the `maker_owner`.

`initial_subscribe` is sent after every successful auth. `resync_messages` is replayed after auth; common entries are `GetMyQuotes` and `GetActiveRfqs`.

`BytesSigner` holds an ed25519 keypair in memory and zeroes it on drop. For HSM- or KMS-backed signing, implement `SignerLike` directly.

---

## Quoting

For each `RfqBroadcast`, build the order preimage, compute `order_id`, sign, and send `Quote`.

```rust
use acta_maker_sdk::{
    compute_order_id, sign_order_id_with_signer, AtomicNonceGenerator,
    OrderPreimageArgs, OrderId, Price, Nonce, decode_base58_32, encode_base58,
};

static NONCE_GEN: AtomicNonceGenerator = AtomicNonceGenerator::new();

let mut messages = handle.subscribe_messages();
while let Ok(msg) = messages.recv().await {
    let ServerMessage::RfqBroadcast(rfq) = msg.as_ref() else { continue };

    let valid_until = std::time::SystemTime::now()
        + std::time::Duration::from_secs(350);
    let nonce = NONCE_GEN.next_u64();
    let price: u64 = calculate_premium(&rfq);

    let args = OrderPreimageArgs {
        chain_id: rfq.market.chain_id.value(),
        program_id: decode_base58_32(&rfq.market.program_id)?,
        is_taker_buy: false,
        position_type: rfq.position_type as u8,
        market: decode_base58_32(&rfq.market.market_pda)?,
        strike: rfq.strike.value(),
        quantity: rfq.quantity.value(),
        gross_price: price,
        valid_until: valid_until.duration_since(std::time::UNIX_EPOCH)?.as_secs(),
        maker: signer.pubkey_bytes(),
        taker: decode_base58_32(&rfq.taker)?,
        nonce,
    };

    let order_id = compute_order_id(&args);
    let signature = sign_order_id_with_signer(&order_id, &signer);

    handle.send(ClientMessage::Quote(QuoteMessage {
        rfq_id: rfq.rfq_id,
        strike: rfq.strike,
        price: Price::new(price),
        valid_until,
        nonce: Nonce::new(nonce),
        order_id: OrderId::new(order_id),
        signature: encode_base58(&signature),
    })).await?;
}
```

Constraints:

- `valid_until ≥ now + 310s`. The trailing 300s is reserved for settlement; the effective trading window is `valid_until − 300s`. The recommended range is `now + 320..360s`.
- `is_taker_buy` is fixed at `false`. The taker is always the option writer. Setting `true` produces an `order_id` the server rejects.
- `nonce` must be unique per `(rfq_id, strike)` per submission. `AtomicNonceGenerator` is safe to declare as a `static`.

---

## Quote lifecycle

Lifecycle events arrive on the same `broadcast::Receiver`, keyed by `order_id`.

| Event | Meaning |
|---|---|
| `QuoteAcknowledged` | Server accepted the quote. On a replace, includes `replaced_order_id`. |
| `QuoteRejected` | Refused; see `reason`. Bare retries fail identically. |
| `QuoteBestStatus` / `QuoteOutbid` | Current book position. |
| `QuoteRefreshRequested` | Settlement-buffer cutoff approaching. Resubmit with `valid_until ≥ min_valid_until`. |
| `QuoteSelected` | Taker selected this quote; settlement is in flight. |
| `QuoteFilled` | Position opened on-chain. Carries `position_pda` and `tx_signature`. |
| `QuoteCancelled` | Terminal. `reason ∈ {requested, risk_check, rfq_accepted}`. |
| `QuoteExpired` | Emitted only when `quote_expired` was opted into via `Hello`. |
| `RfqAvailableAgain` | Settlement reverted; re-quote with a fresh `order_id`. |
| `RfqClosed` | Terminal RFQ event. Discard per-RFQ state. |

Use `ReplaceQuote` for repricing. The prior quote is removed only after the replacement validates, so the swap is single-RTT. `CancelQuote` followed by `Quote` creates a gap and adds a round-trip.

```rust
handle.send(ClientMessage::ReplaceQuote(ReplaceQuoteMessage {
    old_order_id,
    rfq_id, strike, price, valid_until, nonce, order_id, signature,
})).await?;
```

The preimage is constructed identically to a fresh `Quote` with a new `nonce` and `order_id`. The server confirms with `QuoteAcknowledged { replaced_order_id: Some(old_order_id) }`. Retain `old_order_id` until the replacement is acknowledged, since events keyed to the old id may still be in flight.

---

## Querying state

`handle.send_await(msg, timeout)` correlates request and response via `request_id` and returns `Arc<ServerMessage>`. It applies to messages with a defined response shape:

- `GetMyQuotes`, `GetMyTrades`, `GetMakerPositions`, `GetMmSummary` — maker state
- `GetSubscriptions`, `GetActiveRfqs` — session view
- `GetMarketsForMaker`, `GetMarketDescriptors` — instrument metadata
- `GetTokenCaps`, `GetMyCaps` — risk and per-maker limits

`Quote`, `Subscribe`, `CancelQuote` are fire-and-forget; their effects are observed via subsequent unsolicited server events. Calling `send_await` on these returns `SendAwaitError::NoExpectedResponse`.

`AddMints`, `RemoveMints`, `AddChannels`, and `RemoveChannels` mutate subscriptions incrementally. Each mutation returns `SubscriptionUpdated` with the current state.

---

## Reconnection

The SDK reconnects by default. After a disconnect: TCP up -> `Hello` -> `StartAuth` -> challenge signed -> `AuthChallenge` -> ready. Backoff is 250 ms initial, 5 s cap, +/-20% jitter.

Outbound calls via `handle.send()` queue during the reconnection window and flush after re-authentication. `try_send` returns `QueueFull` rather than blocking when immediate failure is preferred.

Inbound state recovery is the integrator's responsibility; the server does not replay events missed during the disconnect window. `initial_subscribe` and `resync_messages` cover the usual recovery queries. Custom recovery can subscribe to the `Authenticated` connection event and issue its own queries.

Events in flight during disconnect can be delivered again after recovery, for example a `QuoteAcknowledged` for an already processed `order_id`. Process lifecycle events idempotently by `order_id`.

---

## Indicative pricing

If the account is enrolled in pre-trade pricing, the server emits `IndicativePricesRequest`; the integrator replies with `IndicativePricesResponse` correlated by `request_id`. Indicative quotes are non-binding and operate under a tighter latency budget than auction quotes.

---

## Errors

`WsClientError` covers transport failures; the reconnection loop retries. `ManagedWsError::Closed` indicates the handle's connection task is no longer running. `SendAwaitError::Timeout` signals the `send_await` deadline elapsed; `Disconnected` indicates the connection went down before the response arrived.

Protocol-level errors arrive as `ServerMessage::Error` (session-level) or `ServerMessage::RequestError` (per-request, correlated by `request_id`). Variants — `AuthError`, `QuoteError`, `CapError`, `RfqError` — are exhaustive enums; the full list is in [`../reference/maker-api.md`](../reference/maker-api.md).

---

## Low-level WsClient

`WsClient` is appropriate when `ManagedWs` does not fit (integration tests, custom auth flows, replay tooling).

```rust
let mut client = WsClient::connect(url).await?;
client.send_hello(hello).await?;
let challenge = client.recv_start_auth().await?;
client.auth_challenge(sign(challenge)).await?;
client.subscribe(subscribe_data).await?;
```

Convenience methods exist for every `ClientMessage` variant. `WsClient` does not reconnect, re-authenticate, or queue outbound messages.

---

## Reference

- [Maker API reference](../reference/maker-api.md) — message catalogue and error variants
- [Maker quickstart (JSON)](maker-quickstart.md) — the same flow at the protocol level
- [Maker wire examples (JSON)](maker-wire-examples.md) — concrete request/response payloads
- [WS common conventions](../reference/ws-common.md) — units, envelopes, error codes
- [Caps reference](../reference/caps.md) — risk and quoting limits
- [FAQ](../reference/faq.md)
