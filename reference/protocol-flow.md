# Protocol Flow

Acta is an RFQ options venue on Solana. Positions are fully collateralized against on-chain escrow accounts. There is no continuous margin engine, no leverage, and no early exercise.

Five products trade on the venue. Covered calls and cash-secured puts settle physically. Call spreads, put spreads and cash puts settle in cash in the quote mint.

API payloads are in [Maker API](maker-api.md), [Taker API](taker-api.md), and [WS common conventions](ws-common.md).

## Actors

| Actor | Role | Access |
| --- | --- | --- |
| **Taker** | Writes the option or spread, posts collateral, opens the RFQ. | Open. Any wallet; no registration, no KYC. Only rate-limited (active RFQ count). |
| **Maker** | Buys the option or spread, pays the premium. For physical positions, may fund the settlement leg. | Permissioned. On-chain `RegisterMaker` is admin-gated and timelocked. Subject to caps. |
| **Keeper** (hot authority) | Co-signs the open instructions, calls `FinalizeMarket`, settles positions. | Protocol-operated. |
| **Oracle setter and settlement attestor** | Compute the expiry fixing and publish it to the oracle accounts. | Protocol-operated. See [Settlement fixing](#settlement-fixing). |
| **Liquidator** | Closes ITM physical positions the maker never funded; fronts the settlement to the taker. | Permissionless. Anyone can run it. |

The maker is the buyer and the taker is the writer. Every RFQ, quote and position carries `side`; the venue accepts only `sell` (the taker writes). A `buy` side is rejected with `taker_buy_unsupported`. The `is_taker_buy` byte in the order-id preimage is `0`.

## Products

`terms` is a tagged union. Each variant fixes the product and its strikes.

| `terms.type` | Strike fields | Market side | Settlement | Taker collateral |
| --- | --- | --- | --- | --- |
| `covered_call` | `strike` | call | Physical | `quantity` underlying atoms |
| `cash_secured_put` | `strike` | put | Physical | `floor(strike × quantity)` quote atoms |
| `call_spread` | `lower_strike`, `upper_strike` | call | Cash, quote mint | `C = ceil(width × quantity)` quote atoms |
| `put_spread` | `lower_strike`, `upper_strike` | put | Cash, quote mint | `C = ceil(width × quantity)` quote atoms |
| `cash_put` | `strike` | put | Cash, quote mint | `C = ceil(strike × quantity)` quote atoms |

`width = upper_strike − lower_strike`. A `cash_put` with strike `K` is executed on chain as a put spread with legs `(0, K)`.

A market is either a call market or a put market (`is_put`). Market rows list in `products` only the products of their own side: put markets list `cash_secured_put`, `put_spread`, `cash_put`; call markets list `covered_call`, `call_spread`. A market with none of the configured products is not listed. Strikes and `strike_tick` are scaled by `1e9`.

## Position lifecycles

### Physical: covered call and cash-secured put

The on-chain position account has two live states, `open` and `funded`. `settled` and `liquidated` are terminal outcomes; the account is closed when they happen.

| From | Instruction | Signer | Conditions | To |
| --- | --- | --- | --- | --- |
| (none) | `OpenPosition` | taker, maker Ed25519 quote, keeper | Protocol not paused; market not expired | `open` |
| `open` | `DepositFundsToPosition` | maker owner | None on the market's expiry or finalization | `funded` |
| `open` | `SettlePosition` | any signer | Market finalized; position OTM | `settled` |
| `funded` | `SettlePosition` | any signer | Market finalized | `settled` |
| `open` | `LiquidatePosition` | any signer | Market finalized; position ITM | `liquidated` |

A `funded` position cannot be liquidated. An `open` ITM position cannot be settled (`PositionNotFunded`). An OTM position cannot be liquidated (`PositionNotItm`).

### Cash: call spread, put spread, cash put

The lifecycle is `open` → `settled`. There is no `funded` state and no liquidation. `DepositFundsToPosition` and `LiquidatePosition` reject spread positions. The maker owes nothing after the premium is paid at open. `SettlePosition` is permissionless and needs only a finalized market.

`PositionUpdated` rows for these products never carry `funded` or `liquidated`.

## What an RFQ does

There is no order book. The taker drives a sealed auction, makers stream signed quotes, and the taker picks the winner.

- The taker sends `RfqRequest` (market, `terms`, `side`, quantity, timeout). The backend broadcasts it to eligible makers.
- Makers reply with signed `Quote`s. Each quote is a `(terms, side, price, valid_until)` against the fixed request. Makers cannot quote partial size or change the terms.
- The taker selects the winner by sending `AcceptQuote { maker, order_id }`. Whatever `order_id` the taker names wins. The server's "best price" (`QuoteBestStatus` / `QuoteOutbid`) is advisory only and never selects the winner.
- Winner-take-all: one quote is locked for the full RFQ quantity. No partial fills, no multiple winners. Every live quote reserves full capacity against caps (open interest, notional, position count) for its validity window; the reservation is released on cancel, expiry, or rejection, and converts into position exposure on fill. See [caps.md](caps.md#how-quotes-consume-capacity).
- A maker's live quote is keyed by `(rfq_id, terms)`. A new quote with the same key replaces the previous one, and `QuoteAcknowledged.replaced_order_id` names it.
- The advisory "best" ranking (used only for `QuoteBestStatus` / `best_price`) is: higher premium wins, ties broken by earlier `received_at`, then by smaller `order_id`.

Indicative pricing: after connecting, makers may receive `IndicativePricesRequest` and reply with non-binding reference prices per strike. Takers see these while browsing, before opening an RFQ.

## Trade lifecycle

```
market traded
  -> RFQ active            (taker requests quotes)
  -> quotes streaming      (makers submit signed quotes)
  -> quote accepted        (taker picks order_id; sponsored tx built)
  -> position open         (atomic risk transfer on-chain)
  -> position funded?      physical only, optional (maker deposits settlement leg)
  -> market finalized      (settlement price written after expiry)
  -> settled / liquidated  (liquidated: physical, ITM, unfunded)
```

### 1. Market

A market carries underlying mint, quote mint, expiry timestamp, `is_put` and the oracle accounts of both mints for that expiry. Strikes belong to orders and positions, not the market identity. Backend tradability also applies the pre-expiry cutoff, the disable state and the finalization state; market expiry alone is not an admission check. After expiry, finalization is required before any settlement or liquidation. See [Expiries and the trading cutoff](#expiries-and-the-trading-cutoff).

### 2. RFQ

The taker sends `RfqRequest` with market, `terms`, `side` and quantity. `quantity` is always in underlying atomic units. Before broadcasting, the backend checks: market is active, taker can trade, trading is not paused, size and tick rules pass, caps do not block. A maker blocked by caps may receive `RfqSkipped` instead of `RfqBroadcast`.

### 3. Quote

Makers receive `RfqBroadcast`, choose a premium, build a 32-byte `order_id`, sign it, and send `Quote`. The quoted `price` is the gross premium per 1 underlying unit, scaled by `1e9`. The `order_id` commits to market, product, strikes, quantity, gross price, validity, maker, taker, and nonce. Physical products use the 182-byte preimage; spreads and cash puts use the 190-byte preimage that also commits the upper strike. Backend checks: order-id and maker signature match, quote still valid, maker registered, maker has enough deposited quote balance for the premium, maker and platform caps pass.

For spreads and cash puts the gross price per unit `G` must satisfy `0 < G < width`.

### 4. Accept and open

The taker sends `AcceptQuote { maker, order_id }`. The backend returns a sponsored transaction; the taker signs it and returns it with `SubmitSignedSponsoredTx`. On chain, the open instruction (`OpenPosition` for physical products, `OpenSpread` for spreads and cash puts) requires the taker's signature, the maker's Ed25519 signature over `order_id` in the preceding instruction, and the keeper co-signature. It fails with `ProtocolPaused` while the protocol is paused. If the checks pass, the position opens atomically; risk transfers here (see [Risk model](#risk-model)).

## Timing and deadlines

Defaults below are the server's configuration defaults; all are configurable.

<!-- generated:rfq-timing-defaults -->
| Parameter | Default | Meaning |
| --- | --- | --- |
| RFQ window | taker-chosen | `expires_at = created_at + request.timeout`. The auction deadline. |
| `settlement_buffer` | 50 s | The trailing window reserved for settlement confirmation. A quote's effective trading cutoff is `valid_until - settlement_buffer`; quotes whose `valid_until` is closer than `settlement_buffer + quote_refresh_lead` are rejected. |
| `quote_refresh_lead` | 10 s | At `effective_expiry - quote_refresh_lead` the server fires `QuoteRefreshRequested` and freezes the quote (unacceptable until re-quoted). |
| `signature_timeout` | 30 s | Max time the taker has to sign the sponsored tx after `AcceptQuote`. The signature deadline is the minimum of `now + signature_timeout`, the quote's effective expiry, and the RFQ's `expires_at`. |
| `submitted_watchdog_timeout` | 120 s | After this the server flags a submitted trade that has not resolved. The RFQ stays locked to the selected quote until the trade confirms on-chain or the server knows the transaction was never sent. |
| `closed_rfq_ttl` | 300 s | How long a closed RFQ is retained before purge. |
| `max_quotes_per_rfq` | 50 | Cap on quotes per RFQ across all makers. |
| `max_rfqs_per_taker / max_active_rfqs` | 50 / 1000 | Concurrency limits (`0` disables the corresponding limit). |
| `max_active_rfqs_per_ip` | 50 | Unfinished RFQs per source IP, counted across wallets and reconnects (`0` disables it). A breach rejects `RfqRequest` with a resource-limit error. |
<!-- /generated:rfq-timing-defaults -->

`rfq.expires_at` is the auction deadline; `quote.valid_until` is the on-chain order validity. They are distinct clocks. `valid_until` may not exceed the market expiry.

On a signature timeout or tx-build failure, the locked (winning) quote is discarded and only the still-valid losing quotes are restored; the RFQ reverts to active if it has not yet expired.

### Expiries and the trading cutoff

An expiry is the `expiry_ts` of a market. Markets are created on chain by the hot wallet with any `expiry_ts` in the future. For each expiry the market needs one oracle account per mint, created before the market. Neither the contract nor the backend enforces an expiry grid or a creation cadence, so the set of expiries is whatever markets exist. Read it from `GET /api/v1/markets` or the `GetExpiries` query, and re-read on `ReferenceChanged`.

New RFQs stop `pre_expiry_cutoff_seconds` before expiry. `GET /api/v1/venue` returns this default (the server's built-in default is 10,800 s). An operator can set an explicit absolute cutoff on a single market, which replaces the default for that market. `/venue` does not report such overrides. A disabled or finalized market is not tradable regardless of the cutoff. The cutoff gates new RFQs and admission only; it does not close existing positions.

## Economics

### Units

`PRICE_SCALE = 1_000_000_000`.

| Field | Meaning |
| --- | --- |
| `price` | Gross premium per 1 underlying unit, 1e9 scale |
| `strike`, `lower_strike`, `upper_strike`, `strike_tick` | Quote per 1 underlying unit, 1e9 scale |
| `quantity` | Underlying atomic units |
| `total_premium` | Net premium paid to the taker, quote atomic units |

Quote-side atomic amount, rounded down or up:

```
scaled_quote_amount(x, quantity)      = floor(x * quantity * 10^quote_decimals / (PRICE_SCALE * 10^underlying_decimals))
scaled_quote_amount_ceil(x, quantity) = ceil( x * quantity * 10^quote_decimals / (PRICE_SCALE * 10^underlying_decimals))
```

`scaled_quote_amount(price, quantity)` is the gross premium. `scaled_quote_amount(strike, quantity)` is the notional of a physical position: put collateral and call settlement amount. Spread collateral and payoff use the ceiling form.

### Cash flow at open

On a successful open:

1. Maker PDA pays the net premium to the taker in the quote token.
2. Maker PDA pays the protocol fee to the protocol fee account.
3. Taker collateral moves into the position escrow.
4. Position state becomes `open`.

### Collateral

| Product | Taker collateral (locked at open) | Maker settlement asset (physical, if ITM) |
| --- | --- | --- |
| Covered call | `quantity` underlying atomic units | `scaled_quote_amount(strike, quantity)` quote atomic units |
| Cash-secured put | `scaled_quote_amount(strike, quantity)` quote atomic units | `quantity` underlying atomic units |
| Call spread, put spread | `C = scaled_quote_amount_ceil(width, quantity)` quote atomic units | None |
| Cash put | `C = scaled_quote_amount_ceil(strike, quantity)` quote atomic units | None |

Taker collateral is fully locked at open. For physical products the maker settlement leg is **not** locked at open: the maker pays the premium and may deposit the settlement asset later with `DepositFundsToPosition` (`open` → `funded`). Spreads and cash puts have no maker leg. The maker needs a deposited program quote balance (via `DepositPremium`) for the premium debit; idle balance is retrieved with `WithdrawPremium`.

### Fees

The maker signs the gross price. The contract nets the fee during open:

```
gross_premium = scaled_quote_amount(price, quantity)

price_after_premium_fee_bps =
  price - floor(price * protocol_fee_bps_premium / 10_000)
premium_fee =
  gross_premium - scaled_quote_amount(price_after_premium_fee_bps, quantity)

exposure = strike          (covered call, cash-secured put, cash put)
         = width           (call spread, put spread)
volume_fee = floor(scaled_quote_amount(exposure, quantity) * protocol_fee_bps_volume / 10_000)

fee_total   = min(premium_fee, volume_fee)
net_premium = gross_premium - fee_total
```

The taker receives `net_premium`; `Position.total_premium` and the wire field `total_premium` hold the same net value in quote atoms. The maker pays `gross_premium`, so its cost is `scaled_quote_amount(price, quantity)` and the fee is that cost minus `total_premium`. `protocol_fee_bps_premium` and `protocol_fee_bps_volume` are on-chain config set by governance (each ≤ 10000 bps) and are published by `GET /api/v1/venue`. `net_price` in WS payloads is a display estimate only; for accounting use the formula above or `total_premium`.

### Spread payoff

For a spread or cash put with legs `(L, U)` at settlement price `S`:

```
call spread: intrinsic = max(S - L, 0)
put spread:  intrinsic = max(U - S, 0)      (cash put: L = 0, U = K)

maker_payoff = scaled_quote_amount_ceil(min(intrinsic, width), quantity)
taker_return = collateral_balance - maker_payoff
```

`maker_payoff` is at most `C`. The taker receives the rest of the escrow, including any amount donated to it. The buyer's profit is `maker_payoff - gross_premium`, and the writer's loss is bounded by `C - net_premium`.

## Settlement and payoff

After expiry the hot wallet publishes one scalar price per oracle account (one for the underlying mint and one for the quote mint) and then calls `FinalizeMarket`:

```
settlement_price = floor(underlying_price * PRICE_SCALE / quote_price)
```

`FinalizeMarket` needs both oracle prices published at or after expiry. The market stores `settlement_price`, which the market rows return as `null` until then. Settlement prices are determined as described in [Settlement fixing](#settlement-fixing).

At-the-money counts as OTM.

| Product | OTM | ITM |
| --- | --- | --- |
| Covered call | `settlement_price <= strike` | `settlement_price > strike` |
| Cash-secured put | `settlement_price >= strike` | `settlement_price < strike` |
| Call spread | `settlement_price <= lower_strike` (payoff 0) | `settlement_price > lower_strike` |
| Put spread, cash put | `settlement_price >= upper_strike` (payoff 0) | `settlement_price < upper_strike` |

### Physical outcomes

| State | Moneyness | Token movement |
| --- | --- | --- |
| `open` | OTM | Taker gets collateral back. |
| `funded` | OTM | Taker gets collateral back; maker gets the settlement deposit back. |
| `funded` | ITM | Taker collateral goes to the maker; the maker settlement deposit goes to the taker (a swap at the strike). |
| `open` | ITM | `SettlePosition` fails. The position must be liquidated. |

| | Covered call | Cash-secured put |
| --- | --- | --- |
| Taker posts (collateral) | Underlying | Quote |
| Taker receives (premium) | Quote (net) | Quote (net) |
| Maker pays (premium) | Quote | Quote |
| Maker funds (settlement, if ITM) | Quote | Underlying |
| ITM: taker receives | Quote (maker's settlement) | Underlying (maker's settlement) |
| ITM: maker receives | Taker's underlying collateral | Taker's quote collateral |
| OTM: each side keeps | Taker keeps collateral; maker keeps settlement deposit | Same |

### Cash outcomes

`SettlePosition` pays `maker_payoff` to the maker and `taker_return` to the taker, both in the quote mint, and closes the position. If one owner is on both sides, both amounts go to distinct quote accounts or one shared account. The final `state` is `settled { settlement: cash { settlement_price, maker_payoff, taker_return } }`.

## Funding deadline (physical positions)

The contract sets no deadline for `DepositFundsToPosition`. It accepts a deposit while the position is `open`, before or after expiry and before or after market finalization. The only thing that ends the window is a terminal instruction.

- **Before finalization.** Funding is always safe.
- **After finalization, position OTM.** Funding has no effect on the outcome. `SettlePosition` returns both sides' assets.
- **After finalization, position ITM and `open`.** `LiquidatePosition` is permissionless. Anyone can call it once the market is finalized. The first of the maker's `DepositFundsToPosition` and a liquidation to land wins. If the deposit lands first, the position is `funded` and `LiquidatePosition` fails; settlement then swaps at the strike.
- **If the liquidation lands first.** The liquidator transfers the settlement amount to the taker, receives the whole escrowed collateral, and the position closes as `liquidated`. The maker keeps nothing and owes nothing further. The premium paid at open is not returned.

The keeper does not liquidate. Its settler settles OTM and funded positions, refuses an ITM `open` position ("must be liquidated, not settled") and logs a count of `open_itm_not_fundable` positions per tick. Whether a third-party liquidator runs on a deployment is not guaranteed by the protocol. A maker that wants the swap must fund before finalization.

Funding a position held by a vault has additional vault rules; see the [vault operator guide](../quickstart/vault-operator-guide.md#funding-bought-options).

## Settlement fixing

The oracle price of each mint at expiry `E` is a time-weighted average of Pyth prices over the 30 minutes ending at `E`. It is computed off chain from Pyth Benchmarks history. The chain checks who signed the number, not how it was computed.

### Policy

The policy is `acta-pyth-benchmarks-v1`, calculation version 1, and applies to oracle type 0. It is compiled into the services and is not operator configuration.

| Parameter | Value |
| --- | --- |
| Window | `[E - 1800, E]` seconds |
| Anchor | The last observation at or before `E - 1800`, read from history starting at `E - 1860` |
| Source | Pyth Benchmarks history for the oracle's feed id; ordinary `price`, not `ema_price` |
| Feed ids | The feed id stored in the oracle account, taken from the mint's oracle source. `GET /api/v1/tokens` lists them as `oracle_feed_ids` |
| Scale | Normalized to `1e9`, integer half-up rounding |
| Confidence filter | Observations with `ceil(conf / price * 1_000_000) > 100_000` ppm (10%) are discarded |
| Other filters | Non-positive prices and observations after `E` are discarded |
| Duplicates | Identical observations at one `publish_time` are merged; different values at one `publish_time` fail the fixing |
| Maximum gap | 60 s between consecutive kept observations, from the anchor to `E - 1800`, and from the last observation to `E` |
| Calculation | Piecewise-constant TWAP: each observation's price holds until the next one; sum of `price × seconds` over the 1800 s window, divided by 1800, rounded half-up |
| Result | Must be non-zero |

If no anchor exists, a gap exceeds 60 s, or two observations conflict, no price is produced and the oracle is not updated. The fixing is never filled from live Hermes prices.

The settlement attestor fetches history in 60 s slots. A slot is fetched only after its end plus a finalization delay (60 s in the shipped configuration), so the last slot is read no earlier than `E + 60 s`. A decision for an `(oracle, expiry)` pair, once made, is stored with its policy hash and is not recomputed under a different policy.

Mints with an on-chain price route are fixed by evaluating the route at expiry with this TWAP in place of the Pyth input. The on-chain message is the same.

### Who submits and what the chain checks

1. After `E`, the oracle setter (hot wallet) computes the TWAP from Benchmarks history.
2. The settlement attestor, a separate service with its own key, computes it independently. The setter submits only if the attestor's price, observation digest and evidence match its own and both signatures verify.
3. The hot wallet submits `UpdateOraclePriceAttested`, preceded in the same transaction by an Ed25519 instruction signed by the attestor. The message binds the domain, program, global config, oracle, price, expiry, `issued_at`, `valid_until` and calculation version. The contract requires `GlobalConfig.settlement_attestor` to be the signer, the message to match exactly, `issued_at >= expiry`, a validity window of at most 300 s and `now` inside it.
4. The contract rejects a zero price, a price above its maximum, and a second update of the same oracle (`OracleAlreadyUpdated`). It does not recompute the TWAP and does not verify any Pyth account.
5. `FinalizeMarket` (hot wallet) combines the two oracle prices into `settlement_price`.

When no attestor is configured, the hot wallet publishes directly with `UpdateOraclePrice`. When one is configured, direct hot publication is rejected and only the cold authority keeps a direct path. The cold authority can also overwrite a published price with `CorrectOraclePrice` until the markets on that oracle are finalized. See [Governance and security](governance.md).

## Risk model

Every position is fully collateralized at open in its own escrow. There is no leverage, no maintenance margin, no mark-to-market, no margin call, and no early exercise. Positions open before expiry and resolve only after market finalization.

Custody is per-position. Taker collateral and, for physical products, the maker's settlement deposit each sit in escrow accounts owned by the position PDA; the maker's premium balance sits in the maker PDA. The protocol never holds principal risk; it custodies escrow and routes settlement. The counterparty is a specific maker, not a pool or the protocol: each position names one taker and one maker.

Spreads and cash puts carry no counterparty risk after open. The maker's entire exposure is the premium it paid, and the payoff is paid out of the taker's escrow at settlement.

For physical products, taker collateral is locked at open. An ITM payout requires the maker to fund settlement or a liquidator to supply the settlement asset. The contract provides both paths; it does not guarantee when a willing, funded liquidator will execute the latter. If the option expires ITM and the maker never funded, the position stays `open` after finalization; a liquidator pays the taker the settlement asset, takes the taker's collateral, and closes the position as `liquidated`. The contract creates no additional claim against the maker for an unfunded settlement leg.

Liquidation is post-finalization only: an ITM, unfunded physical position, not a price-threshold or maintenance call. It is permissionless but not automatic. The taker is paid when the liquidation transaction lands.

Backend caps are a separate risk control layered on top: platform caps (token open interest per underlying, quote notional per quote mint, market open interest) apply to everyone; maker caps (open position count, notional per underlying, deposited premium balance) apply to makers. See [Capacity limits](caps.md).

## State machine and events

Terminal states:

- RFQ: expired, cancelled, or filled.
- Position: `settled` or `liquidated`.

Events worth tracking end to end:

| Event | When |
| --- | --- |
| `QuoteFilled` / `OrderConfirmed` | Position opened on chain; carries `position_pda` and `tx_signature`. `QuoteFilled` has no fee or net-premium fields; read the money from the position row. |
| `RfqClosed` | Auction closed; on a fill, `winner = { maker, price, tx_signature }`. |
| `ChainEvent(position_opened)` | Position opened, via chain events. |
| `PositionUpdated { position, update_type }` | Full position row; `update_type` is `created`, `funded`, `liquidated` or `settled`. Apply it only if its `revision` is newer than the stored one. |
| `MarketFinalized` | Market expired, settlement price set. |
| `ChainEvent(position_settled)` | Assets distributed. |
| `ChainEvent(position_liquidated)` | Unfunded ITM physical position liquidated. |

`RegisterMaker`, `DepositPremium`, `WithdrawPremium`, and `DepositFundsToPosition` are Solana on-chain instructions, not WebSocket messages; build and submit them as transactions signed by the relevant owner wallet. The layouts of the maker's deposit instructions are in [Sandbox](sandbox.md#on-chain-maker-instructions). The full IDL ships in `@acta-markets/ts-sdk` as `ACTA_IDL`.

## Related docs

- [Maker API](maker-api.md) / [Taker API](taker-api.md) — full WS message catalogues
- [WS common conventions](ws-common.md) — encodings, units, time hierarchy, collateral formulas
- [Capacity limits](caps.md) — cap layers and monitoring
- [Governance and security](governance.md) — authority split, permissionless settlement/liquidation
- [Sandbox / Devnet](sandbox.md) — onboarding, endpoints, program addresses
