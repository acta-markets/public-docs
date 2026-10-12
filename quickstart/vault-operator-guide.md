# Vault operator guide

For the governance and delegate keys that run an Acta vault: what the vault enforces, what you must do each cycle, and how to trade through the SDKs. Read-only vault data is on the [HTTP API](../reference/http-api.md#vaults).

## What a vault is

A vault is a share-accounted pool for one fixed token pair:

- **main** mint: deposits, withdrawals, equity and share price are in main.
- **second** mint: the other token of the pair.

The vault trades any Acta market whose `(underlying, quote)` equals `{main, second}` as a set. A USDC-main vault and a SOL-main vault both trade the SOL/USDC markets, calls and puts, as writer or holder. The market decides every payment mint: premium in quote, collateral in underlying for a call and quote for a put, holder funding in quote for a call and underlying for a put. The vault pays or receives each amount from the leg that holds that mint.

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
| `1` | `OPEN_TAKER` | Open positions as writer (the vault sells: RFQ as taker) |
| `2` | `SIGN_MAKER_QUOTES` | Quote RFQs as holder (the vault buys) |
| `4` | `FUND_MAKER` | Fund a position the vault holds |
| `8` | `REBALANCE` | Rebalance swaps |
| `16` | `PROCESS_CAPITAL` | Price equity, process requests, finalize the cycle before the fallback opens |
| `32` | `SET_DEPOSIT_WINDOW` | Open or close the Idle deposit window |

Governance rotates the delegate only in Idle, and not in Closing, EmergencyUnwind or WithdrawOnly. Governance can disable the delegate at any time. A governance key quotes only if it also holds an active primary-delegate grant with `SIGN_MAKER_QUOTES`.

Vault capabilities (`capabilities` on `/api/v1/vaults/{pda}`) limit what the vault may trade, checked at open:

| Bit | Name | Required when |
|---|---|---|
| `1` | `CAN_SELL_OPTIONS` | The vault is the writer |
| `2` | `CAN_BUY_OPTIONS` | The vault is the holder |
| `4` | `CAN_USE_SECOND_MINT_STRATEGY` | The position commits second-mint balance: writer collateral in second, or holder premium or future funding in second. Also any main-to-second rebalance |

Receiving premium into second needs no bit. Because funding is checked at open, a position never becomes unfundable later for lack of a capability.

## Lifecycle

| Phase | Meaning |
|---|---|
| `idle` | No positions. The deposit window may be open |
| `active` | Positions open, all with the cycle's expiry |
| `settling` | Some positions of the cycle settled or liquidated, others still open |
| `processing_capital` | All positions closed; equity and requests are processed |

1. **Deposit window (Idle).** `SetAcceptingDeposits(true)` needs `SET_DEPOSIT_WINDOW` and the manager stake floor; it is allowed only in Idle with safety `normal`. Close the window before the first open: a vault opens from Idle only when the window is closed and no priced batch is waiting. Idle requests can be priced and processed after the window closes (`ProcessVaultEquity`, then `ProcessDeposits` / `ProcessWithdrawals`).
2. **Active.** The first open fixes the cycle expiry; an open with another expiry fails with `VaultExpiryMismatch`. Opens also check `policy.max_open_positions`, the manager stake floor, capabilities and safety `normal`. Deposit and withdrawal requests are accepted until expiry.
3. **Settlement.** After the market is finalized, each position settles or is liquidated in its own transaction; settlement is permissionless. The last one moves the vault to `processing_capital`.
4. **ProcessingCapital.** In order:
   - optional second-to-main rebalance, so main covers withdrawals and the `min_main_liquidity_bps` floor;
   - `ProcessVaultEquity`: values both legs with live Pyth prices, assesses fees and fixes the boundary price;
   - `ProcessDeposits` / `ProcessWithdrawals`, at most 3 users per instruction;
   - `FinalizeVaultCycle`: needs empty queues and returns the vault to `idle`.

Each capital step needs governance or `PROCESS_CAPITAL` until `active_expiry + capital_fallback_delay_secs`; after that anyone may send it. The delay defaults to 43,200 s (12 h), may range 0–259,200 s, and only Acta cold changes it, through the protocol timelock. In Idle the same delay runs from each request's `created_at`. In Idle, a non-operator can price equity only with a withdrawal request older than the delay; a depositor can cancel instead. The fallback lets others run these steps. It does not swap: rebalance needs `REBALANCE`, recovery needs governance.

The SDKs include a reference crank: Rust `chain::vault::next_step` with `examples/vault_capital.rs`, TS `flows.nextCapitalStep` / `flows.planWithdrawals`.

### Closing

Governance calls `StartVaultShutdown` from safety `normal`. Closing is irreversible. It blocks new positions and share minting; withdrawal requests and cancellations stay open in every phase, and existing positions run to settlement. After the last position closes, governance converts second to main (swap mode `closing`; blocked while the protocol is paused). `FinalizeVaultShutdown` assesses fees once and enters `withdraw_only`. It needs zero positions and only dust in second. Governance may call it at once; after `closing_started_at + capital_fallback_delay_secs` anyone may. Management fees stop accruing at `closing_started_at`. Deposit refunds are public in Closing and WithdrawOnly.

### Freeze and emergency

Acta cold or the guardian can freeze a vault. Frozen blocks deposits, new positions and rebalances. Settlement, funding of open holder positions, cancels, withdrawals, governance recovery and capital processing continue. Pending deposits can only be refunded. Only cold can unfreeze.

Only cold can start `EmergencyUnwind`, which is irreversible. It allows deposit refunds, settlement and liquidation, second-to-main conversion and final accounting, then ends in `withdraw_only`. LP payouts stay locked until then, and **funding is blocked**, so unfunded ITM holder positions are lost to liquidation.

## Trading through the SDK

One WebSocket connection on `/vault` (same host as `/maker` and `/taker`) serves one vault. Auth is `StartVaultAuth { operator, vault_pda }`, a challenge signed by the operator key, then `VaultAuthSuccess { operator_kind, permissions }`. Every reconnect re-authenticates; there is no `ResumeAuth`. The only subscribable channel is `vaults`; `VaultProjectionAdvanced` on it means the vault row changed: reload `/api/v1/vaults/{pda}`.

- **Selling** (vault is writer, `OPEN_TAKER`): send `RfqRequest` and `AcceptQuote` as a taker. The server returns `SponsoredTxToSign`; the operator is fee payer and signs slot 0.
- **Buying** (vault is holder, `SIGN_MAKER_QUOTES`): answer `RfqBroadcast` with a quote whose `maker_owner` is the vault PDA, signed by the operator key. The external taker signs the transaction; the vault pays the premium from its quote-mint leg.
- **Swaps** (`REBALANCE`; `recovery` and `closing` need governance): `SwapQuote { mode, direction, amount, slippage_bps }` returns a server-built transaction with one Jupiter route, `minimum_out_amount` and `valid_until`. Sign it and send `SwapSubmit`; `SwapSubmitResult` confirms durable queueing, not execution. Rebalance works in `active` before expiry in either direction, and in `processing_capital` before equity only second-to-main.

Respect the venue limits: quantity on `size_rule`, strikes on `strike_tick` (`GET /api/v1/tokens`), quote validity, RFQ timeouts, per-taker RFQ limit and trading cutoff (`GET /api/v1/venue`). Vault sessions receive `ReferenceChanged` like maker and taker sessions (Rust `VaultOperatorClient::subscribe_reference_changes()`); on it, re-read those routes with `If-None-Match`.

### Checked signing

Never sign a served transaction blind. The checked helpers rebuild the expected instructions from what you agreed to and sign only on an exact match. They compare fee payer, signers, compute budget within your bounds, every account with its flags, instruction data, ATA creates, the maker's Ed25519 quote check and the order id. Jupiter accounts are allowed only in the swap's route tail: at most 64, no signers, no payer, no read-only vault account made writable. A swap whose `minimum_out_amount` is below your `min_output_floor` is rejected even if it fits the quote's slippage. Compute the floor from your own oracle price, not from the server's quote. The `_unverified` / `signSponsoredTxBase64Unverified` helpers skip all of this.

| | Rust (`acta-maker-sdk`, feature `vault-chain`) | TypeScript (`@acta-markets/ts-sdk/ws`) |
|---|---|---|
| Position open | `VaultOperatorClient::verify_and_sign_sponsored_tx` | `verifyAndSignSponsoredTx` |
| Swap | `VaultOperatorClient::verify_and_sign_swap` | `verifyAndSignSwap` |
| Verify only | `chain::vault::verify_vault_open` / `verify_vault_swap` | `flows.verifyVaultOpen` / `flows.verifyVaultSwap` |

Open intents: `Sale` / `"sale"` (vault writes for an external maker), `BetweenVaults` / `"betweenVaults"`, `Fill` / `"fill"` (an external taker writes against the vault's quote; for that taker's check). `terms` are the RFQ and quote terms. `maker_quote_signer` is the maker's `quote_signing` key (`GET /api/v1/makers` or its on-chain Maker account).

Rust, selling:

```rust
use acta_maker_sdk::chain::vault::{ComputeBudgetBounds, VaultAccount, VaultManager, VaultOpenIntent, VaultTradeTerms};

let vault = VaultManager::from_vault(program_id, operator_pubkey, &VaultAccount::parse(&vault_data)?)?;
let bounds = ComputeBudgetBounds { max_unit_limit: 400_000, max_unit_price_micro_lamports: 50_000 };

// On ServerMessage::SponsoredTxToSign(to_sign) for the RFQ this vault accepted:
let terms = VaultTradeTerms {
    underlying_mint, quote_mint, expiry_ts, is_put,
    strike, quantity, price: quote_price, valid_until: quote_valid_until, nonce: quote_nonce,
};
let signed = client.verify_and_sign_sponsored_tx(
    to_sign,
    &VaultOpenIntent::Sale { writer: &vault, maker_owner, maker_quote_signer, terms },
    &lookup_tables, // ChainClient::address_lookup_tables for the served v0 message
    bounds,
)?;
client.submit_signed_sponsored_tx(signed).await?;
```

For swaps, `client.swap_quote(request)` returns the quote; pass it with `VaultSwapIntent { vault, request, quote, main_price_update, second_price_update, min_output_floor }` to `verify_and_sign_swap`, then `swap_submit`. The [Rust SDK README](https://github.com/acta-markets/rust-maker-sdk) has the full swap example.

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
    intent: { kind: "sale", writer: planner, makerOwner, makerQuoteSigner, terms },
  });
  await vault.submitSignedSponsoredTx({ orderIdHex, txBase64: signed });
});

vault.on("swapQuoteResult", async (quote) => {
  const submit = await verifyAndSignSwap({
    intent: { vault: planner, request, quote, mainPriceUpdate, secondPriceUpdate, minOutputFloor },
    tables: await fetchAddressesForLookupTables(flows.servedLookupTableKeys(quote.tx_base64), rpc),
    bounds,
    signer: wallet,
  });
  vault.submitVaultSwap(submit);
});
```

`planner` is `await flows.VaultManagerPlanner.fromVault({ vault, actor, programAddress })`; `request` is what you passed to `vault.requestVaultSwapQuote(...)`; `resolveVaultPriceUpdates(rpc, planner)` from `@acta-markets/ts-sdk/chain` returns the two price-update accounts.

## Funding bought options

When the vault holds an option, the option is not funded at open. To exercise an ITM option, the vault must fund it (`FUND_MAKER`): it pays strike × quantity in quote for a call, or the quantity in underlying for a put, from the matching leg, and receives the collateral at settlement. **Fund before expiry.** After the market is finalized, anyone can liquidate an unfunded ITM position. The liquidator pays the writer and takes the collateral, and the vault keeps nothing: it loses the intrinsic value. Funding is still accepted after finalization until a settlement or liquidation lands, but then you are racing liquidators. Keep enough of the funding mint for every holder position you may need to exercise.

Builders: TS `planner.trading.fundMakerPosition(position)`; Rust `VaultManager::fund_maker_position` in `acta-maker-sdk` (feature `vault-chain`). The builder picks the funding mint from the market side and the vault ATA by that mint. The actor signs (`FUND_MAKER` or governance) and pays; `VaultFundMakerPosition` takes no arguments and moves the amount the position needs.

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

## Pyth freshness

`ProcessVaultEquity`, `FinalizeVaultShutdown`, manager top-ups and every swap read both legs' Pyth `PriceUpdateV2` accounts. Each must:

- carry the feed id of the vault's `OracleSource` and be fully verified;
- be at most **60 s** old on the mainnet program (900 s on the devnet build) and at most 5 s in the future;
- have publish times within 10 s of each other;
- have confidence at most 5% of price.

Pyth's sponsored mainnet feeds update about once a minute, so a sponsored account can be just over 60 s old. If a step fails on staleness, retry after the next update or post your own verified update for the same feed. The feed accounts come from Rust `pyth_price_feed_account` / `oracle_source_pyth_feed`, or TS `resolveVaultPriceUpdates`.

## If the operator disappears

If only the delegate is lost, governance can do everything the delegate did, disable the delegate at once, and rotate it in the next Idle.

If both governance and delegate stop acting:

- No new positions or rebalances happen.
- Positions settle permissionlessly after finalization. ITM positions the vault holds and never funded are liquidated; the vault loses their intrinsic value.
- After `expiry + capital_fallback_delay_secs`, anyone can price equity, process requests and finalize the cycle. Pricing still needs main to cover withdrawals and the liquidity floor. A large second balance can only be converted by `REBALANCE` or governance, so it blocks pricing until someone with those rights acts.
- In Idle, withdrawal requests older than the delay can be priced and processed by anyone; depositors can cancel pending deposits.
- Closing cannot start without governance. Acta cold can freeze the vault or start an emergency unwind.
