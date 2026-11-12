# Protocol changelog

## Next (not deployed)

`acta-maker-sdk` 0.10.0 and `@acta-markets/ts-sdk` 0.7.0. Breaking: address and time fields are renamed after what they hold. The protocol string stays
`1.0.0`, so an older client passes `Hello` and then fails on the renamed fields. The program and
the order ID preimage are unchanged; signatures stay valid.

| Was | Now | Where |
|---|---|---|
| `market` (address or filter) | `market_pda` | `RfqRequest`, `PositionInfo`, `TradeInfo`, `ActiveRfqInfo`, `MyActiveRfqInfo`, `MakerQuoteInfo`, indicative prices, `GetPositions`, `GetMakerPositions`, `GetMyTrades`, `CancelAllQuotes`, `PositionOpened`, `MarketCreated`, `MarketFinalized` |
| `market` (descriptor object) | `market_descriptor` | `RfqBroadcast`, `IndicativePricesRequest`, `MarketDescriptorInfo` |
| `market_id` | `market_pda` | `RfqSkipped`, `CheckQuote`, `MarketCapInfo`, cap error `market_oi_cap_exceeded` |
| `maker` (owner wallet) | `maker_owner` | `QuoteReceived`, `QuotesUpdate`, `AcceptQuote`, `RfqClosed.winner` |
| `maker` (maker PDA) | `maker_pda` | `TradeInfo`, `PositionOpened` |
| `position` | `position_pda` | `PositionSettled`, `PositionLiquidated`, HTTP vault positions and exposure |
| `underlying`, `quote` | `underlying_mint`, `quote_mint` | `MarketInfo` (WS and `GET /markets`) |
| `old_order_id` | `replaced_order_id` | `ReplaceQuote` |
| `sent_at_unix_ms` | `sent_at_ms` | `RfqBroadcast` |
| `server_time_unix_ms` | `server_now_at_ms` | `Welcome`, `Pong` |
| `signature_deadline` | `signature_deadline_at` | `QuoteSelected`, `SponsoredTxToSign` |
| `expiry` | `expiry_ts` | HTTP vault cycles and exposure |
| `event_time` | `occurred_at` | HTTP depositor events and vault cycles |
| `from_time`, `to_time` | `window_start_at`, `window_end_at` | HTTP vault performance windows |
| `spot_time`, `price_time` | `spot_published_at`, `price_published_at` | HTTP vault exposure, vault |
| `deposit_window_set_ts`, `last_hwm_reset_ts`, `last_hwm_update_ts`, `last_settlement_ts` | `*_at` | HTTP vault |
| `capital_fallback_delay_secs` | `capital_fallback_delay_seconds` | HTTP vault |
| query `underlying`, `quote`, `expiry` | `underlying_mint`, `quote_mint`, `expiry_ts` | `GET /markets`; the old names return 400 instead of an unfiltered list |

- Error action `wait`: a server-side timer may clear the condition, resend the same request after it. `RfqAlreadyLocked` is `wait`; `RfqNotFound` and `RfqNotActive` are `refresh` (were `stop`).
- `TokenInfo.protocol_fees` is `quote { … }`, `not_quote_token` or `pending` (was `none` for both).
- `TokenMarketsInfo.session`: `continuous`, `open { closes_at }`, `closed { opens_at }` or `halted`, from the underlying's price-feed schedule.
- A settled physical position's `settlement` carries `collateral_atoms` and `settlement_atoms`, the amounts the program moves.

## 2026-11-11: 0.9.1 and 0.6.1

Additive; no client changes required.

- `GET /api/v1/venue` adds `max_batch_quotes`, the most quotes one `BatchQuotes` message may carry. `acta-maker-sdk` 0.9.1 and `@acta-markets/ts-sdk` 0.6.1 read it.
- Vault deposit and withdraw request pages accept `limit` up to 200, like every other list (was 100).
- Docs: sell indicative requests reach only `rfqs` subscribers, indicatives go stale 60 seconds after the last answered round, the writer bears the protocol fee on a taker buy, and [address fields](ws-common.md#address-fields) lists which name each message uses for market, maker and position addresses.

## 2026-11-11

`acta-maker-sdk` 0.9.0 and `@acta-markets/ts-sdk` 0.6.0. Breaking: absent facts are tagged unions
instead of `null`. The protocol string stays `1.0.0`, so older clients pass `Hello` and then fail on
the changed messages. The program is unchanged.

- `AuthSuccess.role`: `{ type: "taker" }` or `{ type: "maker", maker_pda }` (was nullable `maker_pda`).
- Indicatives: `freshness` is `no_data`, `fresh { updated_at }` or `stale { updated_at }` (was `updated_at: 0` and `is_stale`); each strike has `price: { type: "none" } | { type: "quoted", price }` (was nullable `best_price`).
- `ActiveRfqInfo.best_price`, `MyActiveRfqInfo.best_price`, `QuoteBestStatus.current_best_price` and `QuoteOutbid.current_best_price` are `QuotedPrice`; `MyActiveRfqInfo.lock` is `unlocked` or `locked { order_id }` (was `locked_order_id`).
- `RfqClosed.winner` is `none` or `winner { maker, price, transaction }` with `transaction` `pending` or `sent { signature }`; `your_quote` is `none` or `quoted { order_id, status, price }`.
- `QuoteAcknowledged.replaced` is `none` or `replaced { order_id }` (was `replaced_order_id`).
- `MarketInfo.settlement` is `pending` or `fixed { price }` (was nullable `settlement_price`); `GlobalStats.usd` is `unavailable` or `available { … }`.
- `MakerTradeInfo.position` is `pending` or `confirmed { position_pda }`.
- `SwapQuoteResult.staging` is `"none"`, `"stage"` or `"unstage"`; `RequestError.delivery` gains `not_applicable` instead of `null`.
- Always set now: `RfqCreated.request_id` (equals `rfq_id`), `QuoteFilled.expiry_ts`, `QuoteReceived.net_price` (equals `price` without a fee configuration).
- `PositionInfo` carries `underlying_symbol` and `quote_symbol`; `MakerPositionInfo` is gone, maker reads return `PositionInfo`.
- `null` remains only for `next_cursor` on the last page.

## 2026-11-10

`acta-maker-sdk` 0.8.3 and 0.8.4, `@acta-markets/ts-sdk` 0.5.3 to 0.5.5. The program is unchanged and every change is additive.

- `Welcome.rate_limits` advertises the connection's message budgets: `connection`, `wallet`, `rfqs`, `quotes`, `queries` (each `unlimited` or `bucket` with `per_second` and `burst`) and `liquidity_provider` (`shared` or `not_applicable`). See [Rate limits and admission](ws-common.md#rate-limits-and-admission). The TS client paces its messages by them.
- Every error has an action: `retry`, `reauthenticate`, `refresh`, `reconcile`, `fix_request` or `stop`, listed in the [error catalogs](ws-common.md#error-actions). Rust: `ServerError::action`; TS: `errorAction` and `ActaWsRequestError.action`.
- TS: `createRfqOnce` and `submitOrderOnce` resend a lost reply only when it is safe and reconcile otherwise; `previewFill` prices a fill locally; `grossFromIndicative1e9` turns a sell indicative into the gross price `previewFill` takes. See [Agents and automation](../quickstart/agents.md).
- A repeated `RfqRequest` with an open RFQ's id is rejected with `duplicate_rfq_id`; the docs used to say it returned the RFQ.

## 2026-11-09 (later)

`acta-maker-sdk` 0.8.2 and `@acta-markets/ts-sdk` 0.5.2. The program is unchanged. One breaking change: `/venue` no longer carries protocol fees, so `venue()` in 0.8.1 and 0.5.1 fails; everything else is additive.

- `TokenInfo.session` (`/tokens`, WS `Tokens`): `continuous`, or `pyth_schedule` with the Pyth trading-hours string of the token's feed, so a maker no longer configures TSLA or XAU hours by hand.
- `/venue` reports `max_page_size` (200). The Rust SDK exports `paging::MAX_PAGE_SIZE`.
- Protocol fees moved from `/venue` to each quote token: `TokenInfo.protocol_fees` (`quote` with `premium_bps` and `volume_bps`, or `none`). Quote tokens can have different fees; `/venue` used to answer `503` when they did.
- The indicative response window is 5 seconds (it was 800 ms). The [indicative rules](maker-api.md#indicative-rules) document timing, replacement, partial answers, units, fees and rate limits.
- Rust SDK: `build_maker_withdraw_native_sol_ixs` unwraps exactly the withdrawn amount; other wSOL stays wrapped.
- TS SDK: a `PositionUpdated` push keeps the token symbols of a maker row in the client cache.
- Taker-facing prices (`net_price`, indicatives, `MyActiveRfqs.best_price`, earn APR) apply the protocol fee as the program does: the smaller of the premium fee and the volume fee on the strike or spread width. Spreads and cash puts now carry `net_price`; before, it was `null` for them and the volume cap was ignored.

## 2026-11-09

`acta-maker-sdk` 0.8.1 and `@acta-markets/ts-sdk` 0.5.1. The wire changes below are breaking; the protocol version string stays `1.0.0`, so an older client passes `Hello` and then fails on the changed messages. `acta-maker-sdk` 0.8.0 and `@acta-markets/ts-sdk` 0.5.0 sent `1.1.0` and are refused at `Hello`; use 0.8.1 and 0.5.1. The program is unchanged.

- Indicatives are per taker side. `IndicativePricesRequest`, `IndicativePricesResponse`, `GetIndicativePrices`, `IndicativePrices` and each `TokenMarketIndicatives` carry a required `side`. Makers echo the request's `side`. Buy requests reach only `taker_buy_rfqs` subscribers. `best_price` is the median of maker prices: net of the protocol fee for `sell`, the gross price paid for `buy`.
- `PositionInfo` in `Positions` and `PositionUpdated` carries a required `underlying_price`: `{ type: "available", price, published_at }` or `{ type: "unavailable" }`, the trusted oracle price the server prices RFQs with. `MakerPositionInfo` (`MakerPositions`, `MmSummary`) carries it too and is `PositionInfo` plus token symbols.
- `RfqLocked { rfq_id, rfq_version, locked_at }` tells every other maker on an RFQ that the taker accepted a quote. Their quotes stay as backups until `RfqAvailableAgain` or `RfqClosed`.
- `RfqClosed.reason` gains `server_shutdown`: a graceful stop closes every RFQ whose order has not reached the keeper and cancels its quotes.
- On a buy, `QuoteReceived.net_price`, `QuotesUpdate` quotes and `MyActiveRfqs.best_price` are the gross price the buyer pays; they were shown net of the fee. The TS client keeps the lowest quote as a buy RFQ's best price.
- Query failures are `RequestError` with the request's `request_id`: `GetIndicativePrices`, `GetTokenMarketsInfo`, `GetPositions`, `GetMyActiveRfqs`, `GetMakerPositions`, `GetMyTrades`, `GetMyCaps`, `CheckQuote` and maker market reads sent an uncorrelated `Error` for some failures. `GetTokenMarketsInfo` for an underlying without active markets returns `market_not_found` instead of no response.
- Quoting races (`RfqNotActive`, `QuoteLocked` and the like) are logged at debug, not error; `ServerError::is_quote_race` tells them apart.
- Native SOL custody helpers: `chain::build_maker_deposit_native_sol_ixs` and `chain::build_maker_withdraw_native_sol_ixs`.
- `io_observer` counts HTTP, WebSocket and RPC traffic. HTTP and RPC reuse connections and accept compressed responses.

## 2026-11-05

SDKs `@acta-markets/ts-sdk` 0.4.3 and `acta-maker-sdk` 0.7.3; the program is unchanged.

- A wallet can fund a physical option it bought from a vault (`flags` vault writer + taker buy). The SDK deposit builders refused any vault flag before; only a vault holder is refused, as on chain. Without the deposit such a buy settled as `not_exercised`.
- A funded buy from a vault can settle with a SOL unwrap to the wallet holder. The ts-sdk refused it before; the contract SDK adds `flows::build_settle_position_with_sol_unwrap`, which places the owner after the vault accounts as the program reads it.
- The RFQ close reason `ladder_timeout` is removed. The server never sent it; both SDKs now reject it like any unknown reason.

## 2026-11-04

Program upgrade; SDKs `@acta-markets/ts-sdk` 0.4.2 and `acta-maker-sdk` 0.7.2.

- Vaults write what wallets buy. A vault session receives `buy` RFQs on the `vaults` channel when its delegate holds the new permission `SIGN_WRITER_QUOTES` (`64`), and quotes an ask. The buyer signs `OpenPositionForVaultWriter` (41) or `OpenSpreadForVaultWriter` (78) with holder byte `1` (data length 75 or 83); the vault's delegate signs only the order id. A holder byte `0` is the existing sale to a maker. No price floor applies.
- A vault receives RFQs for any expiry while Idle and only its cycle expiry while Active, on both sides. A buy the vault's last observed collateral leg cannot cover is not sent.
- Names follow position roles; opcodes and bytes are unchanged: `OpenPositionForVaultHolder` (42), `OpenSpreadForVaultHolder` (79), `FundPositionForVaultHolder` (43); delegate bits `OPEN_AS_WRITER` (1), `SIGN_HOLDER_QUOTES` (2), `FUND_HOLDER` (4).
- Program events are version 2. `PositionOpened` (kind 11) is emitted once per open on every path, with writer, holder, `flags` (bit 0 vault writer, 1 vault holder, 2 taker buy), terms, gross and net premium and collateral; `VaultCycleOpened` (kind 48) once per vault side. They replace `OpenPosition`, `OpenSpread`, `OpenTakerBuy`, `OpenTakerBuySpread`, `VaultPositionOpened` and `VaultSpreadOpened`. `SettleSpread` is kind 79, `VaultSpreadClosed` 80, `TakerBuyFunded` 81, `TakerBuyNotExercised` 82. The SDKs do not decode version 1 events.
- Program error `NotPositionHolder` (1111): a deposit signer that is not the position holder.

## 2026-11-02

- Every list request takes required `limit` (at most 200) and a tagged `cursor` (`start`, or `after` with `id`). Every list response has a required nullable `next_cursor`. This replaces timestamp cursors, `has_more`, nested quote-scope paging and dual MM cursors. A larger limit is rejected with `invalid_limit`, a bad cursor with `invalid_cursor`.
- `GetPositionSummary` returns premium totals per quote mint across all products. It is cached for 5 seconds per wallet.
- REST lists take optional `limit` (default 50, at most 200, 100 for vault requests) and optional opaque `cursor`. Omit it for the first page.
- Configuration-sized pushes are capped at 1000 items.
- RFQ admission: new RFQs are rejected before fan-out with `server_overloaded`, `insufficient_collateral` or `temporarily_unavailable`.
- The version stays 1.0.0, so a client built for the previous shapes passes Hello and fails on the changed requests.

## 2026-10-30

- `QuoteRefreshRequested.reason` and `QuoteExpired.reason` are the closed `QuoteExpiryReason` (`expired` | `expiring_soon`). `RfqSkipped` drops its string `reason`; `cap_detail` is required and names the failing cap.
- `total_premium` on position rows is `Balance`: net premium in quote atoms after the protocol fee.
- Market rows list only the pair's products for the market's side; a market with none is not listed.
- SDK wire types drop `#[non_exhaustive]`.

## 2026-10-29

- Required pair product policies are stored in PostgreSQL and edited through audited admin commands; catalog rows carry products and admission rejects disallowed products.
- Adds cash_put {strike: K}, executed as the existing put-spread tag 3 with K1=0/K2=K, including user/vault opens and cash settlement. SOL/USDC enables all five products; call spreads still require a positive lower strike.
- Every position row carries the last applied event revision {slot, transaction_index, instruction_index}; complete lifecycle updates merge only when newer. Migration 064 backfills canonical coordinates and aborts with unresolved PDAs; verified rows require complete coordinates.

- `/api/v1/venue` adds required on-chain `protocol_fee_bps_premium` and `protocol_fee_bps_volume`, refreshed with the config projection.
- PositionInfo and MakerPositionInfo flatten one required `state` (`open`, `funded`, `liquidated`, or `settled` with physical/cash settlement), plus decimals and revision. Orphan rows are excluded in SQL. PositionUpdated coalesces full rows by PDA/revision for maker and taker sessions, independently of caps; caps_snapshot is removed.
- WS protocol `1.0.0` removes feature negotiation. Hello requires `on_disconnect`, `client_name` and `client_version`; Subscribe requires nonempty `products` and both mint lists. Spreads are base protocol, QuoteExpired is unconditional, and the raw Hello version is checked before strict decoding, so any other version receives `unsupported_protocol_version` and close 4006 promptly.
- The wire is closed: every field is always serialized; absent facts are explicit null; state-dependent shapes are tagged unions (`TokenInfo.role`, liability `policy`, `QuoteCheck.verdict`, `EarnAssetSummary.apr`, venue limits, vault capacity). Queries use `QueryFilter`/`TimeBound`, typed status lists, required `limit` (`invalid_limit` outside range) and `PageCursor`; `GetMyQuotes.scope` is tagged. Unknown types, codes and enum values are rejected by both SDKs. Never-populated `stats_delta`/`stats` are removed.
- Wire addresses use validated base58 `Address32` (TypeScript `Address`); invalid addresses reject at decoding. Vault authentication records a typed actor identity.
- RFQ and quote commands carry required `OptionTerms` through Core. Checked commands return `CoreError`; failures do not mutate state, and the runtime maps errors to wire responses.
- Public products and chain tags are distinct; cash_put converts only through canonical terms conversion. Redis RFQ/quote cache v2 requires complete OptionTerms without legacy defaults. Subscribe decodes a validated ProductSet, and pair policy checks share the registry authority.
- `/maker/data` is the read-only data plane: up to four concurrent sessions per maker owner, independent of its quote session.

## 2026-10-28

- Renamed capability `spreads_v2` to `spreads` (order bytes unchanged).
- All position opens use keeper-sponsored v0 transactions with address lookup tables (ALT).

