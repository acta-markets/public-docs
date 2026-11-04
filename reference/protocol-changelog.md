# Protocol changelog

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

