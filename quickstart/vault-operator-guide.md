# Vault operator guide

For the governance and delegate keys that run an Acta vault: what the vault enforces, what you must do each cycle, and how to trade through the SDKs. Read-only vault data is on the [HTTP API](../reference/http-api.md#vaults).

## What a vault is

A vault is a share-accounted pool for one fixed token pair:

- **main** mint: deposits, withdrawals, equity and share price are in main.
- **second** mint: the other token of the pair.

The vault trades any Acta market whose `(underlying, quote)` equals `{main, second}` as a set. A USDC-main vault and a SOL-main vault both trade the SOL/USDC markets, as writer or holder, in all five products: covered call, cash-secured put, call spread, put spread and cash put (see [Protocol flow](../reference/protocol-flow.md#products)). The market and the product decide every payment mint. The vault pays or receives each amount from the leg that holds that mint.

| Product | Premium | Writer collateral | Holder funding |
|---|---|---|---|
| Covered call | quote | underlying | quote, before finalization, to exercise |
| Cash-secured put | quote | quote | underlying, before finalization, to exercise |
| Call spread, put spread, cash put | quote | quote | none |

Spreads and cash puts settle in cash in the quote mint. The holder vault receives `maker_payoff` and the writer vault receives `taker_return` at settlement. Neither side is funded or liquidated.

Capital moves in cycles. Every position of one cycle has the same expiry. Deposits and withdrawals are queued in escrow and processed at a share price fixed after the cycle; escrowed deposits never fund trading.

## Roles and keys

| Key | Set by | Can do |
|---|---|---|
| Governance | Acta cold at `CreateVault` (rotatable through a timelocked update) | Everything a delegate can, plus delegate rotation, timelocked updates (cap, fees, capabilities, policy, governance), recovery swaps, Closing |
| Primary delegate | Governance | Only the permission bits granted, until `expires_at` (0 = no expiry) |
| Acta cold / guardian | Protocol | Freeze (cold or guardian), unfreeze and emergency unwind (cold only), capital fallback delay |

Delegate permission bits:

| Bit | Name | Allows |
|---|---|---|
| `1` | `OPEN_TAKER` | Open positions as writer (the vault sells: RFQ as taker), including spreads |
| `2` | `SIGN_MAKER_QUOTES` | Quote RFQs as holder (the vault buys), including spreads |
| `4` | `FUND_MAKER` | Fund a physical (covered call or cash-secured put) position the vault holds |
| `8` | `REBALANCE` | Rebalance swaps |
| `16` | `PROCESS_CAPITAL` | Price equity, process requests, finalize the cycle before the fallback opens |
| `32` | `SET_DEPOSIT_WINDOW` | Open or close the Idle deposit window |

Governance rotates the delegate only in Idle, and not in Closing, EmergencyUnwind or WithdrawOnly. Governance can disable the delegate at any time. A governance key quotes only if it also holds an active primary-delegate grant with `SIGN_MAKER_QUOTES`.

Vault capabilities (`capabilities` on `/api/v1/vaults/{pda}`) limit what the vault may trade, checked at open:

| Bit | Name | Required when |
|---|---|---|
| `1` | `CAN_SELL_OPTIONS` | The vault is the writer |
| `2` | `CAN_BUY_OPTIONS` | The vault is the holder |
| `4` | `CAN_USE_SECOND_MINT_STRATEGY` | The position commits second-mint balance. Physical: writer collateral in second, or holder premium or future funding in second. Spread or cash put: the quote mint is the vault's second mint, because premium, collateral and payoff all use the quote leg. Also any main-to-second rebalance |

Receiving premium into second needs no bit. Because funding is checked at open, a position never becomes unfundable later for lack of a capability. The capability and permission checks are the same for the standalone vault opens and for a trade between two vaults; each vault is checked for its own side.

## Lifecycle

| Phase | Meaning |
|---|---|
| `idle` | No positions. The deposit window may be open |
| `active` | Positions open, all with the cycle's expiry |
| `settling` | Some positions of the cycle settled or liquidated, others still open |
| `processing_capital` | All positions closed; equity and requests are processed |

1. **Deposit window (Idle).** `SetAcceptingDeposits(true)` needs `SET_DEPOSIT_WINDOW` and the manager stake floor; it is allowed only in Idle with safety `normal`. Close the window before the first open: a vault opens from Idle only when the window is closed and no priced batch is waiting. Idle requests can be priced and processed after the window closes (`ProcessVaultEquity`, then `ProcessDeposits` / `ProcessWithdrawals`).
2. **Active.** The first open fixes the cycle expiry; an open with another expiry fails with `VaultExpiryMismatch`. Opens also check `policy.max_open_positions`, the manager stake floor, capabilities and safety `normal`. Deposit and withdrawal requests are accepted until expiry.
3. **Settlement.** After the market is finalized, each position settles in its own transaction; settlement is permissionless. An unfunded ITM physical position is liquidated instead. Spreads and cash puts only settle. The last close moves the vault to `processing_capital`.
4. **ProcessingCapital.** In order:
   - optional second-to-main rebalance, so main covers withdrawals and the `min_main_liquidity_bps` floor;
   - `ProcessVaultEquity`: values both legs at their [price routes](#prices), assesses fees and fixes the boundary price;
   - `ProcessDeposits` / `ProcessWithdrawals`, at most 3 users per instruction;
   - `FinalizeVaultCycle`: needs empty queues and returns the vault to `idle`.

Each capital step needs governance or `PROCESS_CAPITAL` until `active_expiry + capital_fallback_delay_secs`; after that anyone may send it. The delay defaults to 43,200 s (12 h), may range 0–259,200 s, and only Acta cold changes it, through the protocol timelock. In Idle the same delay runs from each request's `created_at`. In Idle, a non-operator can price equity only with a withdrawal request older than the delay; a depositor can cancel instead. The fallback lets others run these steps. It does not swap: rebalance needs `REBALANCE`, recovery needs governance.

The SDKs include a reference crank: Rust `chain::vault::next_step` with the crate's `vault_capital` example (`cargo run --example vault_capital --features chain-rpc`), TS `flows.nextCapitalStep` / `flows.planWithdrawals`.

### Closing

Governance calls `StartVaultShutdown` from safety `normal`. Closing is irreversible. It blocks new positions and share minting; withdrawal requests and cancellations stay open in every phase, and existing positions run to settlement. After the last position closes, governance converts second to main (swap mode `closing`; blocked while the protocol is paused). `FinalizeVaultShutdown` assesses fees once and enters `withdraw_only`. It needs zero positions and only dust in second. Governance may call it at once; after `closing_started_at + capital_fallback_delay_secs` anyone may. Management fees stop accruing at `closing_started_at`. Deposit refunds are public in Closing and WithdrawOnly.

### Freeze and emergency

Acta cold or the guardian can freeze a vault. Frozen blocks deposits, new positions and rebalances. Settlement, funding of open physical holder positions, cancels, withdrawals, governance recovery and capital processing continue. Pending deposits can only be refunded. Only cold can unfreeze.

Only cold can start `EmergencyUnwind`, which is irreversible. It allows deposit refunds, settlement and liquidation, second-to-main conversion and final accounting, then ends in `withdraw_only`. LP payouts stay locked until then, and **funding is blocked**, so unfunded ITM holder positions are lost to liquidation.

## Trading through the SDK

One WebSocket connection on `/vault` (same host as `/maker` and `/taker`) serves one vault. Auth is `StartVaultAuth { operator, vault_pda }`, a challenge signed by the operator key, then `VaultAuthSuccess { vault_pda, operator, operator_kind, permissions }`, where `operator_kind` is `governance` or `delegate` and `permissions` is a u32 bitmask of the delegate permission bits (TS `VaultDelegatePermission` / `hasVaultPermission`). The TS `ActaWsClient` emits `vaultAuthenticated` for it, not `authenticated`. Every reconnect re-authenticates; there is no `ResumeAuth`. The only subscribable channel is `vaults`, on by default; `VaultProjectionAdvanced` on it means this vault's row changed: reload `/api/v1/vaults/{pda}`, and reload it after every reconnect too, since missed notifications are not replayed. To follow the set of vaults, use a maker or taker session subscribed to `vaults`: it receives `VaultCatalogChanged { vault_pda }` when a vault is created, re-delegated or closed (not when a delegate is disabled). It is a hint that can be lost with a dropped connection, so re-list `GET /api/v1/vaults` on it and after every connect or reconnect.

- **Selling** (vault is writer, `OPEN_TAKER`): send `RfqRequest` with `terms` and `side: "sell"`, then `AcceptQuote`, as a taker. The server returns `SponsoredTxToSign`. The keeper (the deployment's `GlobalConfig.hot_authority`) pays and signs slot 0; the operator signs slot 1. The vault pays the collateral from the matching leg and receives the net premium in its quote leg.
- **Buying** (vault is holder, `SIGN_MAKER_QUOTES`): answer `RfqBroadcast` with a quote whose `maker_owner` is the vault PDA, signed by the operator key. The quote carries the RFQ's `terms` and `side`. The external taker signs the transaction; the vault pays the gross premium (net premium plus protocol fee) from its quote-mint leg.
- **Vault to vault:** a vault can sell to another vault's quote. Writer and holder must be two distinct vaults; the contract rejects a trade of a vault with itself. Each vault is checked for its own permission, capability and `max_open_positions`. Both vaults count the position in their cycle, so both must be in a cycle with that expiry.
- **Swaps** (`REBALANCE`; `recovery` and `closing` need governance and accept only `direction: "second_to_main"`): `SwapQuote` returns a server-built transaction. The swap runs as 1–4 hops, each a CPI into a program that Acta cold registered as a swap venue: Jupiter v6, plus Exponent's Generic SY and CLMM programs for PT purchases, on mainnet; a mock router on devnet. Rebalance works in `active` before expiry in either direction while the vault has at least one open position, and in `processing_capital` before equity only second-to-main.

A swap quote is exact-in: `amount` is the input in raw atoms of the source leg, as a decimal string. `slippage_bps` is optional (default 50). `vault_pda` must be the authenticated vault.

```json
{
  "type": "SwapQuote",
  "data": {
    "request_id": "3b0f6a0e-8c41-4d2a-9f57-2e6c1d9a7b13",
    "vault_pda": "VaultPdaBase58",
    "mode": "rebalance",
    "direction": "main_to_second",
    "amount": "250000000",
    "slippage_bps": 50
  }
}
```

```json
{
  "type": "SwapQuoteResult",
  "data": {
    "request_id": "3b0f6a0e-8c41-4d2a-9f57-2e6c1d9a7b13",
    "execution_id": "<64 hex>",
    "mode": "rebalance",
    "tx_base64": "<base64 v0 VersionedTransaction>",
    "last_valid_block_height": 312456789,
    "in_amount": "250000000",
    "quoted_out_amount": "1503221",
    "minimum_out_amount": "1495705",
    "valid_until": "1710000060",
    "staging": null
  }
}
```

Amounts and `valid_until` (Unix seconds) are decimal strings; `last_valid_block_height` is a number, the block height the transaction's blockhash is valid through. The transaction carries the vault's [price section](#prices). `execution_id` names this swap in every later message. A failed quote returns a `RequestError` with a `Generic` code from the [WS error code catalog](../reference/ws-common.md#ws-error-codes) (usually a `vault_*` or `vault_swap_*` row, also general codes such as `invalid_amount`; `vault_swap_oracle_stale` is retryable within seconds, `vault_swap_route_unavailable` and `vault_swap_service_unavailable` later).

Sign it and send `SwapSubmit`:

```json
{
  "type": "SwapSubmit",
  "data": {
    "request_id": "9d4e2c71-5a3b-4f80-b6e2-0c7a1f9d3e58",
    "execution_id": "<64 hex>",
    "signed_tx_base64": "<signed, base64>"
  }
}
```

```json
{ "type": "SwapSubmitResult", "data": { "request_id": "9d4e2c71-5a3b-4f80-b6e2-0c7a1f9d3e58", "execution_id": "<64 hex>" } }
```

`SwapSubmitResult` confirms durable queueing, not execution. A failed `SwapSubmit` carries `delivery` for that request: `not_forwarded` means this request was rejected before it was queued, `unknown` means it may have been queued. It says nothing about your earlier submissions of the same transaction: treat a swap as not executed only if every `SwapSubmit` you sent for it returned `not_forwarded`. Resolve `unknown`, a lost `SwapSubmitResult` or a restart with `GetSwapStatus` (TS `vault.getVaultSwapStatus({ execution_id })`, Rust `client.swap_status(execution_id)`):

```json
{ "type": "GetSwapStatus", "data": { "request_id": "c1a7f3e9-2b64-4d58-8e0f-6a9b3c2d1e47", "execution_id": "<64 hex>" } }
```

```json
{
  "type": "SwapStatus",
  "data": {
    "request_id": "c1a7f3e9-2b64-4d58-8e0f-6a9b3c2d1e47",
    "execution_id": "<64 hex>",
    "state": { "type": "finalized", "signature": "5eyk...base58sig", "slot": 312456801 }
  }
}
```

`state.type` is `pending`, `finalized` (with `signature` and `slot`, a number), `failed`, `expired` or `unknown`. `unknown` means the server has no resolved handoff of this swap for this vault: none exists, or a failed handoff may still be retried or already be executing. It is not proof that nothing executed. Every session of the vault also gets `SwapSettled` when the swap becomes terminal:

```json
{
  "type": "SwapSettled",
  "data": {
    "vault_pda": "VaultPdaBase58",
    "execution_id": "<64 hex>",
    "state": { "type": "failed" }
  }
}
```

A push missed during a reconnect is not replayed, so poll `GetSwapStatus` after reconnecting.

#### Two-transaction swaps

A Solana transaction can lock at most 64 accounts. Some paths do not fit in one transaction, such as buying an Exponent PT with USDC (USDC → hyloSOL → SY → PT). The server then splits the swap in two, and the quote carries `staging`:

- `"stage"`: this transaction swaps the input leg into the vault's account of an intermediate mint, the staging mint (for example hyloSOL). The vault records the staged amount.
- `"unstage"`: this transaction swaps the staged balance into the output leg.

`staging` is `null` for a one-transaction swap. After a `stage` is finalized, send the same `SwapQuote` again with a new `request_id`: as long as the vault has something staged, every swap quote is the `unstage` of the whole staged amount into the requested output leg. `in_amount` is the staged amount, not your original `amount`. Each half is a full swap: it has its own `execution_id`, status and `SwapSettled`, its own minimum output, and its own oracle drift check.

While something is staged, the vault's `staging` field (HTTP) is set. Equity processing, cycle finalization, final withdrawal, starting Closing, recovery, manager top-ups, fee updates and closing the vault all wait until it is unstaged. A delegate can unstage into main while the vault is `normal`. From 300 s after the first stage, governance can unstage into main in any safety mode. The server splits a swap only in `rebalance` mode.

Respect the venue limits: quantity on `size_rule`, every strike (both legs of a spread) on `strike_tick`, 1e9-scaled (`GET /api/v1/tokens`), quote validity, RFQ timeouts, per-taker RFQ limit and trading cutoff (`GET /api/v1/venue`). `GET /api/v1/markets` lists on each market the `products` allowed on its side. Vault sessions receive `ReferenceChanged` like maker and taker sessions (Rust `VaultOperatorClient::subscribe_reference_changes()`); on it, re-read those routes with `If-None-Match`.

### Checked signing

Never sign a served transaction blind. The checked helpers rebuild the expected instructions from what you agreed to and sign only on an exact match. They compare fee payer, signers, compute budget within your bounds, every account with its flags, instruction data, ATA creates, the maker's Ed25519 quote check and the order id. For swaps they also check:

- **Price section.** Each leg must be the vault's `PriceRoute` followed by that route's accounts, with Pyth feeds read on one of your shards.
- **Hops.** 1–4 hops, each through its venue PDA, chaining the input leg through the vault's own transit accounts to the output leg. Hop accounts carry no signers and no payer, and make no fixed, price, venue or program account writable. Each hop has at most 64 CPI accounts; the path has at most 128 account slots and 1,024 data bytes.
- **Setup.** Allowed setup is only idempotent creation of the vault's transit accounts and a stake-pool balance update for a pool one of the vault's routes reads.
- **Staging.** A `stage` must use a staging mint whose route you listed.

A swap whose `minimum_out_amount` is below your `min_output_floor` is rejected even if it fits the quote's slippage. The floor does not apply to a `stage`. Compute the floor from your own oracle price, not from the server's quote.

| | Rust (`acta-maker-sdk`, feature `vault`) | TypeScript (`@acta-markets/ts-sdk/ws`) |
|---|---|---|
| Position open | `VaultOperatorClient::verify_and_sign_sponsored_tx` | `verifyAndSignSponsoredTx` |
| Swap | `VaultOperatorClient::verify_and_sign_swap` | `verifyAndSignSwap` |
| Verify only | `chain::vault::verify_vault_open` / `verify_vault_swap` | `flows.verifyVaultOpen` / `flows.verifyVaultSwap` from `/chain` |
| Price routes | `PriceRoute::from_account` at `VaultAddresses::{main,second}_price_route` | `fetchVaultPriceRoutes` / `decodePriceRouteAccount` + `findPriceRoutePda` from `/chain` |

Open intents: `Sale` / `"sale"` (vault writes for an external maker), `BetweenVaults` / `"betweenVaults"` (vault writes against another vault's quote), `Fill` / `"fill"` (an external taker writes against the vault's quote). Every intent carries `keeper`: the deployment's `GlobalConfig.hot_authority`, read from chain (Rust `ChainClient::hot_authority()`, TS the `hotAuthority` field of the decoded GlobalConfig). Do not take it from the venue: it pays and signs slot 0. `terms` are the RFQ and quote terms: Rust `VaultSaleTerms::Physical(VaultTradeTerms)` or `VaultSaleTerms::Spread(VaultSpreadTerms)`, TS `VaultTradeTerms` with `kind: "physical"` or `kind: "spread"`. Both carry `side`, which must be `sell`. `maker_quote_signer` is the maker's `quote_signing` key (`GET /api/v1/makers` or its on-chain Maker account).

| Product | Rust terms | Instruction rebuilt |
|---|---|---|
| Covered call, cash-secured put | `VaultSaleTerms::Physical` | `OpenPositionForVaultAsTaker` / `AsMaker` / `OpenPositionBetweenVaults` |
| Call spread, put spread | `VaultSaleTerms::Spread` with `kind: SpreadType::CallSpread` / `PutSpread` | `OpenSpreadForVaultAsTaker` (78) / `AsMaker` (79) / `OpenSpreadBetweenVaults` (80) |
| Cash put | `VaultSaleTerms::Spread`, `PutSpread`, `lower_strike: 0`, `upper_strike: K` | same as put spread |

`VaultManager::open_spread_as_taker` (vault sells), `open_spread_as_maker` (vault buys) and `open_spread_with_vault_holder` (vault sells to another vault) build the instructions. `VaultSpreadTerms` is validated before building: ordered strikes, a positive lower strike for a call spread, `0 < price < width`, and `valid_until` no later than the market expiry.

Rust, selling:

```rust
use acta_maker_sdk::chain::vault::{
    ComputeBudgetBounds, VaultAccount, VaultManager, VaultOpenIntent, VaultSaleTerms, VaultTradeTerms,
};
use acta_maker_sdk::TakerSide;

let vault = VaultManager::from_vault(program_id, operator_pubkey, &VaultAccount::parse(&vault_data)?)?;
let bounds = ComputeBudgetBounds { max_unit_limit: 400_000, max_unit_price_micro_lamports: 50_000 };
let keeper = chain.hot_authority().await?;

// On ServerMessage::SponsoredTxToSign(to_sign) for the RFQ this vault accepted:
let terms = VaultSaleTerms::Physical(VaultTradeTerms {
    underlying_mint, quote_mint, expiry_ts, is_put, side: TakerSide::Sell,
    strike, quantity, price: quote_price, valid_until: quote_valid_until, nonce: quote_nonce,
});
let signed = client.verify_and_sign_sponsored_tx(
    to_sign,
    &VaultOpenIntent::Sale { keeper, writer: &vault, maker_owner, maker_quote_signer, terms },
    &lookup_tables, // ChainClient::address_lookup_tables for the served v0 message
    bounds,
)?;
client.submit_signed_sponsored_tx(signed).await?;
```

For a call spread, put spread or cash put, build `VaultSaleTerms::Spread(VaultSpreadTerms { underlying_mint, quote_mint, expiry_ts, kind, side: TakerSide::Sell, lower_strike, upper_strike, quantity, price, valid_until, nonce })` and pass it in the same intent.

`client.sell_rfq(&chain, &sale, connection_epoch, &gate, &mut desk)` runs the whole sale: it sends `RfqRequest`, offers each `QuoteReceived` with matching `terms` to your `SaleDesk::take`, sends `AcceptQuote` for the first taken quote, rebuilds the signed terms from that quote, verifies and signs the served transaction against the counterparty, and submits it. `VaultSale { market_pda, market, terms: OptionTerms, quantity, rfq_timeout, window, bounds }` accepts any `OptionTerms`; it rejects a market whose side does not match the product and checks spread strikes before the request. `SaleOutcome::Submitted` means the participant-signed transaction reached the keeper; `SaleError::in_flight()` returns the signed sale when a failed submit may still execute.

For swaps, `client.swap_quote(request)` returns the quote. Pass it with `VaultSwapIntent { vault, request, quote, price_routes: PairPriceRoutes { main, second, staging, shards }, min_output_floor, account }` to `verify_and_sign_swap`, then send it with `swap_submit_in_epoch(payload, epoch, gate)`. `epoch` is the connection epoch `wait_until_ready()` returned; the swap is sent only on that connection, and only if `gate` admits it (`Ungated` admits every send). `account` is the `VaultAccount` read before the request; the second leg and the staging slot are checked against it. `main` and `second` are the vault's decoded `PriceRoute`s. `staging` lists the routes of the mints you accept as a staging mint; pass `&[]` to refuse two-transaction swaps. `shards` lists the Pyth push-oracle shards you accept. The crate's `vault_operator` example (`cargo run --example vault_operator --features vault`) shows verified fill and swap signing.

TypeScript:

```ts
import { fetchAddressesForLookupTables } from "@solana/kit";
import { flows } from "@acta-markets/ts-sdk/chain";
import { ActaWsClient, WalletAuthProvider, verifyAndSignSponsoredTx, verifyAndSignSwap } from "@acta-markets/ts-sdk/ws";

const vault = new ActaWsClient({ url, role: "vault", vaultPda });
vault.connectAndAuthenticate(new WalletAuthProvider({ publicKeyBase58: operator, signMessage }));
const bounds = { maxUnitLimit: 400_000, maxUnitPriceMicroLamports: 50_000 };

vault.on("sponsoredTxToSign", async (orderIdHex, txBase64) => {
  const tables = await fetchAddressesForLookupTables(flows.servedLookupTableKeys(txBase64), rpc);
  const signed = await verifyAndSignSponsoredTx({
    orderIdHex, txBase64, tables, bounds, signer: wallet,
    intent: { kind: "sale", keeper, writer: planner, makerOwner, makerQuoteSigner, terms },
  });
  await vault.submitSignedSponsoredTx({ orderIdHex, txBase64: signed });
});

vault.on("swapQuoteResult", async (quote) => {
  const submit = await verifyAndSignSwap({
    intent: { vault: planner, request, quote, priceRoutes, minOutputFloor, staging: vaultAccount },
    tables: await fetchAddressesForLookupTables(flows.servedLookupTableKeys(quote.tx_base64), rpc),
    bounds,
    signer: wallet,
  });
  vault.submitVaultSwap(submit);
});
```

`keeper` is the `hotAuthority` of the decoded GlobalConfig; `terms` is `{ kind: "physical", ..., side }` or `{ kind: "spread", positionType, lowerStrike, upperStrike, ..., side }`. `staging` is the decoded `Vault` account read before the request; a staging half is checked against its `stagedMint`, `stagedAmount` and `stagedGeneration`. `planner` is `await flows.VaultManagerPlanner.fromVault({ vault, actor, programAddress })`; `request` is the `SwapQuoteData` sent: the arguments to `vault.requestVaultSwapQuote(...)` plus its returned `request_id` and the `vault_pda`; `priceRoutes` is `{ ...(await fetchVaultPriceRoutes(rpc, planner)), staging, shards }`. `fetchVaultPriceRoutes` comes from `@acta-markets/ts-sdk/chain`. `staging` lists the decoded routes of the mints you accept as a staging mint (`[]` refuses two-transaction swaps). `shards` lists the push-oracle shards the deployment posts to (default `[0]`). If the served price section differs from the routes on one of those shards, verification fails with `{ kind: "priceSection", leg }`.

### Building swaps yourself

A manager can also build a swap without the Acta server. In Rust, use `VaultManager::swap(kind, &VaultSwapPlan { price_section, amount_in, minimum_out, valid_until, hops })` with `VaultSwapKind::{Rebalance, Recovery, Closing, Staging}`. In TS, use `planner.swaps.spotRebalance(plan)`, `recoveryConvertToMain`, `closingConvertToMain` or `stagingSwap`. A Jupiter route from `GET /swap/v2/build` goes in as one hop with its accounts and data unchanged. The hop's program must be a registered venue (`["swap_venue", program]`). On the wire each hop is:

```text
account_count u8 | data_len u16 | amount_offset u16 (0xFFFF = none) | [scale_num u64 | scale_den u64] | data
```

The accounts are `venue PDA, program, source, destination`, followed by the hop's own accounts. A hop after the first may set `amount_offset`: the program then writes `floor(((received + 1) × num − 1) / den)` as a u64 at that byte offset of the hop's data, where `received` is how much the hop's source got earlier in the same swap. With `num == den` it writes `received` itself; a hop into Exponent's Generic SY `mint` uses `num` = SY index and `den` = 1e12. `amount_in` is the most the input leg may spend; the minimum output and drift apply to what was actually spent and received.

## Funding bought options

This section applies to covered calls and cash-secured puts only. A vault that holds a call spread, put spread or cash put has nothing to fund: the position is collateralized in full by the writer and the vault receives its payoff at settlement. `VaultFundMakerPosition` on such a position fails.

When the vault holds a physical option, the option is not funded at open. To exercise an ITM option, the vault must fund it (`FUND_MAKER`): it pays strike × quantity in quote for a call, or the quantity in underlying for a put, from the matching leg, and receives the collateral at settlement. **Fund before expiry.** After the market is finalized, anyone can liquidate an unfunded ITM position. The liquidator pays the writer and takes the collateral, and the vault keeps nothing: it loses the intrinsic value. The contract sets no funding deadline. Funding is accepted after finalization until a settlement or liquidation lands, but then you are racing liquidators. Keep enough of the funding mint for every physical holder position you may need to exercise.

`ChainClient::unfunded_positions(vault)` returns every `open` position whose maker is the vault. That list includes spreads and cash puts, which stay `open` until settlement and cannot be funded. Skip entries whose `terms` is a spread or cash put before building a fund instruction.

Builders: TS `planner.trading.fundMakerPosition(position)`; Rust `VaultManager::fund_maker_position` in `acta-maker-sdk` (feature `chain`). The builder picks the funding mint from the market side and the vault ATA by that mint. The actor signs (`FUND_MAKER` or governance) and pays; `VaultFundMakerPosition` takes no arguments and moves the amount the position needs.

```rust
use acta_maker_sdk::chain::vault::{VaultAccount, VaultManager, VaultPositionRef};
use solana_sdk::transaction::Transaction;

let vault = VaultManager::from_vault(program_id, operator.pubkey(), &VaultAccount::parse(&vault_data)?)?;
let fund = vault.fund_maker_position(&VaultPositionRef {
    underlying_mint, quote_mint, expiry_ts, is_put,
    order_id, // the held position's order id
})?;
let tx = Transaction::new_signed_with_payer(&[fund], Some(&operator.pubkey()), &[&operator], recent_blockhash);
```

## Prices

Each vault mint is priced by its on-chain `PriceRoute` (`["price_route", mint]`), registered by Acta cold through the timelock. A route is a Pyth push feed, optionally multiplied by one on-chain rate:

- SOL, USDC: the mint's Pyth USD feed.
- An LST: the stake pool's rate (SOL per LST) × SOL/USD.
- An Exponent PT: the PT's value in SOL × SOL/USD. The value in SOL is `(1 + apy)^−τ × min(1, backing)`, where τ is the time to maturity, `apy` is set by Acta governance in the route, and backing is the SOL value per PT held in the Exponent vault. This is a policy valuation, not the market price.

`ProcessVaultEquity`, `FinalizeVaultShutdown`, `EmergencyFinalizeWithdrawOnly`, manager top-ups and every swap carry a price section: each leg's route followed by the accounts the route reads. The two finalizations read prices only when the vault holds some of the second asset. A Pyth update must:

- carry the route's feed id and be fully verified;
- be no older than the route's `max_age` (60 s by default) and at most 5 s in the future;
- have confidence at most 5% of price.

The two legs' newest Pyth publish times must be within the smaller of the two routes' `max_skew` (10 s by default). A stake-pool rate may lag the current epoch by at most the route's epoch lag (1 epoch for a PT route). Pyth's sponsored mainnet feeds update about once a minute, so a sponsored account can be just over 60 s old. If a step fails on staleness, retry after the next update, or push your own update to the feed. Build the section with Rust `VaultManager::price_section(main, second, shard)` or TS `resolveVaultPriceSection(rpc, planner, shard)`.

## Devnet differences

- **Swaps** go through a pool-backed mock router (`BtHXpaAvhaKYk9pR4vNpuMZLTR8XeXnZ4iTy2Q1NzuCz`), registered as the devnet swap venue, not through Jupiter, which has no devnet deployment. The checked helpers need no router setting: they verify the served venue PDA.
- **Price routes** are registered with wider Pyth tolerances, because devnet feeds publish less often and on independent cadences: up to 900 s staleness and 900 s between the two legs' publish times.

## If the operator disappears

If only the delegate is lost, governance can do everything the delegate did, disable the delegate at once, and rotate it in the next Idle.

If both governance and delegate stop acting:

- No new positions or rebalances happen.
- Positions settle permissionlessly after finalization. ITM physical positions the vault holds and never funded are liquidated; the vault loses their intrinsic value. Spreads and cash puts settle in cash with no action from the vault.
- After `expiry + capital_fallback_delay_secs`, anyone can price equity, process requests and finalize the cycle. Pricing still needs main to cover withdrawals and the liquidity floor. A large second balance can only be converted by `REBALANCE` or governance, so it blocks pricing until someone with those rights acts.
- In Idle, withdrawal requests older than the delay can be priced and processed by anyone; depositors can cancel pending deposits.
- Closing cannot start without governance. Acta cold can freeze the vault or start an emergency unwind.
