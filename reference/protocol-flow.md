# Protocol Flow

Acta is an RFQ options venue on Solana. Positions are fully collateralized against on-chain escrow accounts and settle physically after expiry. There is no continuous margin engine, no leverage, and no early exercise.

API payloads are in [Maker API](maker-api.md), [Taker API](taker-api.md), and [WS common conventions](ws-common.md).

## Actors

| Actor | Role | Access |
| --- | --- | --- |
| **Taker** | Writes the option (covered call or cash-secured put), posts collateral, opens the RFQ. | Open. Any wallet; no registration, no KYC. Only rate-limited (active RFQ count). |
| **Maker** | Buys the option, pays premium, may fund the settlement leg. | Permissioned. On-chain `RegisterMaker` is admin-gated and timelocked. Subject to caps. |
| **Keeper** (hot authority) | Relays fills on-chain, co-signs `OpenPosition`, calls `FinalizeMarket`. | Protocol-operated. |
| **Liquidator** | Closes ITM positions the maker never funded; fronts the settlement to the taker. | Permissionless. Anyone can run it. |
| **Oracle / admin** | Publishes the scalar settlement price after expiry. | Protocol-operated; settlement/liquidation that consume it are permissionless. |

The maker is the option buyer and the taker is the writer. The `is_taker_buy` field in the order-id preimage is fixed at `0`; a taker-buys-the-option market type is reserved but not implemented.

## What an RFQ does

There is no order book. The taker drives a sealed auction, makers stream signed quotes, and the taker picks the winner.

- The taker sends `RfqRequest` (market, position type, strike, quantity). The backend broadcasts it to eligible makers.
- Makers reply with signed `Quote`s. Each quote is a `(strike, price, valid_until)` against the fixed request — makers cannot quote partial size or change the side.
- The taker selects the winner by sending `AcceptQuote { maker, order_id }`. Whatever `order_id` the taker names wins. The server's "best price" (`QuoteBestStatus` / `QuoteOutbid`) is advisory only and never selects the winner.
- Winner-take-all: one quote is locked for the full RFQ quantity. No partial fills, no multiple winners. Every live quote reserves full capacity against caps (open interest, notional, position count) for its validity window; the reservation is released on cancel, expiry, or rejection, and converts into position exposure on fill. See [caps.md](caps.md#how-quotes-consume-capacity).
- One quote per `(maker, strike)`: a maker's new quote on the same strike replaces the prior one. A maker may quote several distinct strikes.
- The advisory "best" ranking (used only for `QuoteBestStatus` / `best_price`) is: higher premium wins, ties broken by earlier `received_at`, then by smaller `order_id`.

Indicative pricing: after connecting, makers may receive `IndicativePricesRequest` and reply with non-binding reference prices per strike. Takers see these while browsing, before opening an RFQ.

## Trade lifecycle

```
market traded
  -> RFQ active            (taker requests quotes)
  -> quotes streaming      (makers submit signed quotes)
  -> quote accepted        (taker picks order_id; sponsored tx built)
  -> position open         (atomic risk transfer on-chain)
  -> position funded?       optional (maker deposits settlement leg)
  -> market finalized      (oracle settlement price written after expiry)
  -> settled OTM / settled ITM funded / liquidated ITM unfunded
```

### 1. Market

A market carries underlying mint, quote mint, expiry timestamp, put/call side and oracle setup. Strikes belong to its orders/positions, not the market identity. Backend tradability also applies the configured pre-expiry cutoff, disable state and finalization state; market expiry alone is not an admission check. After expiry, oracle finalization is required before any settlement or liquidation.

### 2. RFQ

The taker sends `RfqRequest` with market, position type, strike, and quantity. `quantity` is always in underlying atomic units. Before broadcasting, the backend checks: market is active, taker can trade, trading is not paused, size and tick rules pass, caps do not block. A maker blocked by caps may receive `RfqSkipped` instead of `RfqBroadcast`.

### 3. Quote

Makers receive `RfqBroadcast`, choose a premium, build a 32-byte `order_id`, sign it, and send `Quote`. The quoted `price` is gross premium per 1 underlying unit, scaled by `1e9`. The `order_id` commits to market, position type, strike, quantity, gross price, validity, maker, taker, and nonce. Backend checks: order-id and maker signature match, quote still valid, maker registered, maker has enough deposited quote balance for the premium, maker and platform caps pass.

### 4. Accept and open

The taker sends `AcceptQuote { maker, order_id }`. The backend returns a sponsored transaction; the taker signs it and returns it with `SubmitSignedSponsoredTx`. On chain, `OpenPosition` requires the taker's signature, the maker's Ed25519 signature over `order_id`, and the keeper co-signature. If those pass, the position opens atomically; risk transfers here (see [Risk model](#risk-model)).

## Timing and deadlines

Defaults below are rendered from the server's config defaults by a test in `rfq-server-config`; all are configurable.

<!-- generated:rfq-timing-defaults -->
| Parameter | Default | Meaning |
| --- | --- | --- |
| RFQ window | taker-chosen | `expires_at = created_at + request.timeout`. The auction deadline. |
| `settlement_buffer` | 90 s | The trailing window reserved for settlement confirmation. A quote's effective trading cutoff is `valid_until - settlement_buffer`; quotes whose `valid_until` is closer than `settlement_buffer + quote_refresh_lead` are rejected. |
| `quote_refresh_lead` | 10 s | At `effective_expiry - quote_refresh_lead` the server fires `QuoteRefreshRequested` and freezes the quote (unacceptable until re-quoted). |
| `signature_timeout` | 30 s | Max time the taker has to sign the sponsored tx after `AcceptQuote`. The signature deadline is the minimum of `now + signature_timeout`, the quote's effective expiry, and the RFQ's `expires_at`. |
| `submitted_watchdog_timeout` | 120 s | The watchdog logs a stalled keeper or listener. The RFQ stays Enqueued until the trade confirms on-chain or the server knows the transaction was never sent. |
| `closed_rfq_ttl` | 300 s | How long a closed RFQ is retained before purge. |
| `max_quotes_per_rfq` | 50 | Cap on quotes per RFQ across all makers. |
| `max_rfqs_per_taker / max_active_rfqs` | 50 / 1000 | Concurrency limits (`0` disables the corresponding limit). |
| `max_active_rfqs_per_ip` | 50 | Unfinished RFQs per source IP, counted across wallets and reconnects (`0` disables it). A breach rejects `RfqRequest` with a resource-limit error. |
<!-- /generated:rfq-timing-defaults -->

`rfq.expires_at` is the auction deadline; `quote.valid_until` is the on-chain order validity. They are distinct clocks.

On a signature timeout or tx-build failure, the locked (winning) quote is discarded and only the still-valid losing quotes are restored; the RFQ reverts to active if it has not yet expired.

## Economics

### Units

`PRICE_SCALE = 1_000_000_000`.

| Field | Meaning |
| --- | --- |
| `price` | Gross premium per 1 underlying unit, 1e9 scale |
| `strike` | Quote per 1 underlying unit, 1e9 scale |
| `quantity` | Underlying atomic units |
| `total_premium` | Net premium paid to taker, quote atomic units |

Quote-side atomic amount:

```
scaled_quote_amount(x, quantity) =
  floor(x * quantity * 10^quote_decimals / (PRICE_SCALE * 10^underlying_decimals))
```

Used for both `scaled_quote_amount(price, quantity)` = gross premium and `scaled_quote_amount(strike, quantity)` = notional / put collateral / call settlement amount.

### Cash flow at open

On a successful `OpenPosition`:

1. Maker PDA pays net premium to the taker in the quote token.
2. Maker PDA pays the protocol fee to the protocol fee account.
3. Taker collateral moves into the position escrow.
4. Position status becomes `open`.

### Collateral

| Position type | Taker collateral (locked at open) | Maker settlement asset (if ITM) |
| --- | --- | --- |
| Covered call | `quantity` underlying atomic units | `scaled_quote_amount(strike, quantity)` quote atomic units |
| Cash-secured put | `scaled_quote_amount(strike, quantity)` quote atomic units | `quantity` underlying atomic units |

Taker collateral is fully locked at open. The maker settlement leg is **not** locked at open: the maker pays premium at open and may deposit the settlement asset later with `DepositFundsToPosition` (status `open` → `funded`). The maker also needs a deposited program quote balance (via `DepositPremium`) for the premium debit; idle balance is retrieved with `WithdrawPremium`.

### Fees

The maker signs the gross price. The contract nets fees during open:

```
gross_premium = scaled_quote_amount(price, quantity)

price_after_premium_fee_bps =
  price - floor(price * protocol_fee_bps_premium / 10_000)
premium_fee =
  gross_premium - scaled_quote_amount(price_after_premium_fee_bps, quantity)

notional_quote = scaled_quote_amount(strike, quantity)
volume_fee = floor(notional_quote * protocol_fee_bps_volume / 10_000)

fee_total   = min(premium_fee, volume_fee)
net_premium = gross_premium - fee_total
```

The taker receives `net_premium`; `Position.total_premium` stores the same net value. `protocol_fee_bps_premium` and `protocol_fee_bps_volume` are on-chain config set by governance (each ≤ 10000 bps). `net_price` in WS payloads is a display estimate only; for accounting use the formula above or the on-chain `total_premium`.

## Settlement and payoff

Settlement is physical: it moves tokens at the strike. After expiry the hot authority publishes scalar prices to the Acta oracle accounts (configured via `OracleSource`) and calls `FinalizeMarket`:

```
settlement_price = floor(underlying_price * PRICE_SCALE / quote_price)
```

Price publication has two modes, selected by `GlobalConfig.settlement_attestor`:

- **Direct** (no attestor configured): `UpdateOraclePrice` writes the backend-published scalar.
- **Attested** (attestor configured): the hot authority must submit `UpdateOraclePriceAttested`, immediately preceded in the same transaction by an Ed25519 signature from the configured attestor key over a domain-separated message binding the program, config, oracle, price, expiry and a short validity window; the on-chain program verifies the binding. The published price is computed off-chain from Pyth Benchmarks history under a fixed canonical TWAP policy.

Neither mode verifies a Pyth price-update account on-chain; settlement trusts the Acta oracle (plus, in attested mode, the independent second signature), not an on-chain Pyth proof.

At-the-money counts as OTM.

| Position type | OTM | ITM |
| --- | --- | --- |
| Covered call | `settlement_price <= strike` | `settlement_price > strike` |
| Cash-secured put | `settlement_price >= strike` | `settlement_price < strike` |

Outcomes by status and moneyness:

| Status | Moneyness | Token movement |
| --- | --- | --- |
| `open` | OTM | Taker gets collateral back. |
| `funded` | OTM | Taker gets collateral back; maker gets settlement deposit back. |
| `funded` | ITM | Taker collateral goes to maker; maker settlement deposit goes to taker (a swap at the strike). |
| `open` | ITM | Normal settlement fails; the position must be liquidated. |

Who holds and receives what:

| | Covered call | Cash-secured put |
| --- | --- | --- |
| Taker posts (collateral) | Underlying | Quote |
| Taker receives (premium) | Quote (net) | Quote (net) |
| Maker pays (premium) | Quote | Quote |
| Maker funds (settlement, if ITM) | Quote | Underlying |
| ITM: taker receives | Quote (maker's settlement) | Underlying (maker's settlement) |
| ITM: maker receives | Taker's underlying collateral | Taker's quote collateral |
| OTM: each side keeps | Taker keeps collateral; maker keeps settlement deposit | Same |

## Risk model

Every position is fully collateralized at open in its own escrow. There is no leverage, no maintenance margin, no mark-to-market, no margin call, and no early exercise. Positions open before expiry and resolve only after market finalization.

Custody is per-position. Taker collateral and the maker's settlement deposit each sit in escrow accounts owned by the position PDA; the maker's premium balance sits in the maker PDA. The protocol never holds principal risk; it custodies escrow and routes settlement. The counterparty is a specific maker, not a pool or the protocol: each position names one taker and one maker.

Taker collateral is locked at open. An ITM payout requires the maker to fund settlement or a liquidator to supply the settlement asset. The contract provides both paths; it does not guarantee when a willing, funded liquidator will execute the latter.

The maker pays the premium at open and may fund the settlement leg later. The settlement leg is not pre-funded at open. If the option expires ITM and the maker never called `DepositFundsToPosition`, normal settlement fails and the position stays `open`; a liquidator then pays the taker the settlement asset, takes the taker's collateral, and closes the position as `liquidated`. The maker forfeits the premium already paid, and the protocol enforces no further debt. The contract does not create an additional claim against the maker for an unfunded settlement leg.

Liquidation is post-expiry only: an ITM, unfunded position after finalization, not a price-threshold or maintenance call. It is permissionless but not automatic; the protocol exposes the path and does not run the liquidator. The taker is paid when the liquidation transaction lands.

Backend caps are a separate risk control layered on top: platform caps (token open interest per underlying, quote notional per quote mint, market open interest) apply to everyone; maker caps (open position count, notional per underlying, deposited premium balance) apply to makers. See [Capacity limits](caps.md).

## State machine and events

Terminal states:

- RFQ: expired, cancelled, or filled.
- Position: settled or liquidated.

Events worth tracking end to end:

| Event | When |
| --- | --- |
| `QuoteFilled` / `OrderConfirmed` | Position opened on-chain; carries `position_pda` and `tx_signature`. |
| `RfqClosed` | Auction closed; on a fill, `winner = { maker, price, tx_signature }`. |
| `ChainEvent(position_opened)` | Position opened, via chain events. |
| `MarketFinalized` | Market expired, oracle settlement price set. |
| `ChainEvent(position_settled)` | Assets distributed. |
| `ChainEvent(position_liquidated)` | Unfunded ITM position liquidated. |

`RegisterMaker`, `DepositPremium`, `WithdrawPremium`, and `DepositFundsToPosition` are Solana on-chain instructions, not WebSocket messages; build and submit them as transactions signed by the relevant owner wallet. Contact the Acta team for the program IDL and account layouts.

## Related docs

- [Maker API](maker-api.md) / [Taker API](taker-api.md) — full WS message catalogues
- [WS common conventions](ws-common.md) — encodings, units, time hierarchy, collateral formulas
- [Capacity limits](caps.md) — cap layers and monitoring
- [Governance and security](governance.md) — authority split, permissionless settlement/liquidation
- [Sandbox / Devnet](sandbox.md) — onboarding, endpoints, program addresses
