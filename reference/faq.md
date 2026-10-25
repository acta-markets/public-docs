# FAQ

Integration questions and protocol-level errors.

## Questions

### What happens to active quotes when the maker disconnects?

With cancel-on-disconnect, the server removes the session's active and retained non-winning quotes after detecting the disconnect. The client requests `cancel_on_disconnect` in `Hello.features`; the server confirms it in `Welcome.enabled_features`. The managed Rust quote SDK requests it by default. Selected/executing obligations survive. Without it, resting quotes can remain fillable while offline.

Resume restores server-side subscriptions; fresh authentication requires establishing them again. The managed SDK reconciles its desired target on either path. After reconnect, apply `GetMyQuotes { scope: "live" }` and use `GetOrderStatus` for unresolved orders. An empty quote or position list is not proof of nonexecution. See [cancellation semantics](maker-api.md#cancelallquotes-maker---server).

Events in flight during disconnect can be delivered again after recovery, for example a `QuoteAcknowledged` or `QuoteFilled` already processed before the disconnect. Treat lifecycle events as idempotent by `order_id`.

### Can multiple maker bots share a single key?

No. Each maker pubkey gets one quote WebSocket on `/maker`. A second replaces the first with `Error` (`generic.code = "session_replaced"`); cancel-on-disconnect can cancel its quotes. Up to four `/maker/data` sessions may share the owner without replacing the quote session. After `GetMmSummary` recovery they receive `PositionUpdated`, `MakerCapsUpdated` and `TradeExecuted`. A fifth data session receives `AuthError` (`reason = "session_limit_exceeded"`). Use data sessions for monitoring, funding and standby; open the quote session only when taking over. Use separate maker keys for parallel quoting bots.

### Why was a quote rejected?

`QuoteRejected.reason` identifies the cause. Common values:

| Reason | Meaning |
|---|---|
| `invalid_strike` | The strike is not in the RFQ's `order_options` set. |
| `order_id_mismatch` | The submitted `order_id` does not equal `SHA-256(preimage182)`. See [Troubleshooting](#troubleshooting). |
| `quote_expiry_too_short` | `valid_until < now + min_quote_validity_seconds` (`GET /api/v1/venue`, currently 100). |
| `cap_exceeded` | A position-count, notional, or balance cap was breached. See [`caps.md`](caps.md). |
| `rfq_not_active` | The RFQ expired or filled before the quote arrived. |

Fix the cause before retrying. Re-sending the same payload will fail the same way.

### Why are some RFQs not delivered?

The server pre-filters RFQs against the maker's caps before broadcast. When a pre-filter triggers, the maker receives `RfqSkipped` with a `reason` field (typical values include `token_oi_cap_exceeded` and `maker_insufficient_balance`) in place of `RfqBroadcast`. Current cap headroom can be inspected via `GetMyCaps`. Cap mechanics are documented in [`caps.md`](caps.md).

### What is the appropriate value for `valid_until`?

The hard floor is `now + min_quote_validity_seconds` (from `Welcome` or `GET /api/v1/venue`, currently 100); lower values are rejected with `quote_expiry_too_short`. The hard ceiling is the market's `expiry_ts`; later values are rejected with `market_expired`. The server reserves the trailing `settlement_buffer_seconds` (currently 90) as the settlement buffer, so the trading cutoff is `valid_until` minus that buffer. Size the quote against the auction you are quoting into: `rfq.expires_at + min_quote_validity_seconds` covers the whole auction plus the settlement buffer and the refresh lead. Add a few seconds of slack for clock skew, and stay at or below market expiry. Longer windows leave stale quotes live without helping fills, because the taker cannot accept after `rfq.expires_at`. Track server clock offset from `Welcome.server_time_unix_ms` and `Pong.server_time_unix_ms`.

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

The server emits WebSocket-protocol-level pings every 30 seconds and closes idle connections after a 90-second timeout. The Rust and TypeScript SDKs respond to protocol pings automatically. Raw WebSocket clients must respond to protocol pings; those pongs maintain liveness. Application-layer `Ping` is optional and returns `Pong.server_time_unix_ms` for clock-offset estimation.

If drops persist with a correctly responding client, check the network path. Corporate proxies and firewalls often terminate TCP connections they consider idle.
