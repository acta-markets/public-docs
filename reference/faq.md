# FAQ

Integration questions and protocol-level errors.

## Questions

### What happens to active quotes when the maker disconnects?

Quotes are not auto-cancelled on disconnect. They remain on the server until `valid_until` or `RfqClosed`. Subscriptions do not persist; resend them after auth. After reconnect, call `GetMyQuotes` to reconcile live quotes.

Events in flight during disconnect can be delivered again after recovery, for example a `QuoteAcknowledged` or `QuoteFilled` already processed before the disconnect. Treat lifecycle events as idempotent by `order_id`.

### Can multiple maker bots share a single key?

No. Only one active WebSocket connection per maker pubkey is allowed. A second connection replaces the first; the displaced session receives `Error` with `generic.code = "session_replaced"` and is closed. Use separate maker keys for parallel bots.

### Why was a quote rejected?

`QuoteRejected.reason` identifies the cause. Common values:

| Reason | Meaning |
|---|---|
| `invalid_strike` | The strike is not in the RFQ's `order_options` set. |
| `order_id_mismatch` | The submitted `order_id` does not equal `SHA-256(preimage182)`. See [Troubleshooting](#troubleshooting). |
| `quote_expiry_too_short` | `valid_until < now + 310s` (the server's settlement buffer floor). |
| `cap_exceeded` | A position-count, notional, or balance cap was breached. See [`caps.md`](caps.md). |
| `rfq_not_active` | The RFQ expired or filled before the quote arrived. |

Fix the cause before retrying. Re-sending the same payload will fail the same way.

### Why are some RFQs not delivered?

The server pre-filters RFQs against the maker's caps before broadcast. When a pre-filter triggers, the maker receives `RfqSkipped` with a `reason` field (typical values include `token_oi_cap_exceeded` and `maker_insufficient_balance`) in place of `RfqBroadcast`. Current cap headroom can be inspected via `GetMyCaps`. Cap mechanics are documented in [`caps.md`](caps.md).

### What is the appropriate value for `valid_until`?

The hard floor is `now + 310s`; lower values are rejected with `quote_expiry_too_short`. The server reserves the trailing 300 seconds as the settlement buffer, so the trading cutoff is `valid_until - 300s`. Use `now + 320..360s` unless you have a reason not to. Longer windows leave stale quotes live without helping fills, because the taker cannot accept after `rfq.expires_at`. Track server clock offset from `Welcome.server_time_unix_ms` and `Pong.server_time_unix_ms`.

## Troubleshooting

### `order_id_mismatch`

The `order_id` must equal the SHA-256 hash of the 182-byte preimage laid out in [`maker-api.md`](maker-api.md) (Quote rules). Common causes:

- **Endianness.** All `u64` fields are written little-endian in the binary preimage.
- **Field offsets.** Offsets are exact (`domain_tag` at byte 0, `chain_id` at byte 4, `program_id` at byte 12, etc.). A single-byte misalignment cascades through the remainder of the preimage.
- **`taker` field.** The `taker` field in the preimage carries the taker's pubkey from `RfqBroadcast`, not the maker's pubkey.
- **`is_taker_buy` flag.** This field is fixed at `0` (false); the taker is always the option writer. Implementations that default the field to `true` must override it explicitly.

The Rust SDK's `compute_order_id()` builds this preimage. Other languages should test against a known-good preimage and `order_id` pair before deployment.

### `rfq_not_active`

The RFQ expired or another maker filled it before your quote arrived. You can only reduce this race: lower submission latency and use `BatchQuotes` when quoting several strikes for one RFQ.

### Persistent disconnects

The server emits WebSocket-protocol-level pings every 30 seconds and closes idle connections after a 90-second timeout. The Rust and TypeScript SDKs respond to protocol pings automatically. Raw WebSocket clients must respond to protocol pings and additionally emit application-layer `Ping` messages approximately every 30 seconds.

If drops persist with a correctly responding client, check the network path. Corporate proxies and firewalls often terminate TCP connections they consider idle.
