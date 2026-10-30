# Sandbox / Devnet Guide

## Endpoints

| Service | Devnet | Mainnet |
|---------|--------|---------|
| Maker quote WebSocket | `wss://devnet-api.acta.markets/maker` | `wss://beta-api.acta.markets/maker` |
| Maker data WebSocket | `wss://devnet-api.acta.markets/maker/data` | `wss://beta-api.acta.markets/maker/data` |
| Taker WebSocket | `wss://devnet-api.acta.markets/taker` | `wss://beta-api.acta.markets/taker` |
| HTTP API | `https://devnet-api.acta.markets` | `https://beta-api.acta.markets` |
| Solana cluster | Devnet (`https://api.devnet.solana.com`) | Mainnet-beta (`https://api.mainnet-beta.solana.com`) |

Both environments use the same message envelope and encodings. See [WS common conventions](ws-common.md). The getting-started walkthrough below uses **Devnet** (free test tokens); Mainnet uses real SOL/USDC and live markets — the registration flow and protocol are identical, see [Differences from production](#differences-from-production).

## Getting started

### 1. Generate keypairs

Two Ed25519 keypairs are required:

```bash
# Maker owner wallet (on-chain transactions)
solana-keygen new -o maker-owner.json

# Quote signing key (signs order_id values)
solana-keygen new -o quote-signing.json
```

`quote_signing` can be the same key as `maker_owner`, but use a separate key for production-like testing.

### 2. Request maker registration

Maker registration is admin-gated. Contact the Acta team with:

- `maker_owner` pubkey (base58)
- `quote_signing` pubkey (base58)

The Acta admin executes the `RegisterMaker` on-chain instruction, which creates a maker PDA.

### 3. Get test tokens

You need devnet SOL (for tx fees) and devnet USDC (for premium/collateral).

**Devnet SOL:**

Get free devnet SOL from the Solana faucet: https://faucet.solana.com/

Select Devnet, paste your wallet address, request an airdrop. Fund both `maker_owner` and `quote_signing` wallets if they are separate.

Or via CLI:

```bash
solana airdrop 2 --url devnet
```

**Devnet USDC:**

Get USDC-Dev tokens from the SPL Token Faucet: https://spl-token-faucet.com/?token-name=USDC-Dev

Connect your wallet, select USDC-Dev, and mint tokens to your address.

These are devnet tokens with no real value.

If you have trouble claiming tokens, contact the Acta team.

### 4. Deposit premium

Deposit quote token (devnet USDC) into the maker PDA via the `DepositPremium` on-chain instruction.

- The `maker_owner` wallet signs this transaction.
- `maker_owner` needs an associated token account (ATA) for the quote mint, and so does the maker PDA.

The layout is under [On-chain maker instructions](#on-chain-maker-instructions).

### 5. Connect

```
wss://devnet-api.acta.markets/maker
wss://devnet-api.acta.markets/maker/data
```

Authentication uses the same Ed25519 challenge-response as production.

## What's available on devnet

| Feature | Details |
|---------|---------|
| Markets | Created by the Acta team; SOL/USDC pairs are typical. Each market row lists the `products` it accepts, including spreads and cash puts |
| Rate limits | Same as production. See [Rate limits and admission](ws-common.md#rate-limits-and-admission) |
| Caps | Test values; may differ from production |
| Indicative pricing | Supported |
| Sponsored tx signing | Supported |
| Oracle | Backend reads its configured Pyth/Hermes feeds; settlement publication writes Acta oracle accounts. Oracle readiness is checked independently of Solana cluster selection |
| HTTP API | Public read endpoints available under `/api/v1` (`/venue`, `/tokens`, `/markets`, `/makers`, `/vaults`, `/stats`); participant RFQs/orders require an authenticated session |

## Differences from production

| Aspect | Devnet | Production |
|--------|--------|------------|
| Token mints | Devnet test mints (contact team) | Mainnet SPL tokens |
| Oracle feeds | Backend-configured feed IDs and oracle mapping | Backend-configured feed IDs and oracle mapping |
| Settlement | Same logic, test tokens | Real assets |
| Markets | Created on request | Dynamic |
| SOL | Free via airdrop | Real SOL required |

> The devnet program build differs from production in two vault paths: vault swaps go through a
> mock router instead of Jupiter, and vault Pyth checks allow older and less aligned prices. See
> [Vault operator guide: devnet differences](../quickstart/vault-operator-guide.md#devnet-differences).
> Everything else differs only externally: mints, oracles, and environment config.

Browsers obtain current reference prices from the selected backend through [`GetTokenMarketsInfo`](taker-api.md#tokenmarketsinfo). They do not need Pyth credentials. Solana devnet selection does not itself choose a different Hermes endpoint or guarantee feed freshness.

## Program addresses

| Address | Value |
|---------|-------|
| Program ID | `33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U` (the same on devnet and mainnet) |

### PDA derivation

| PDA | Seeds |
|-----|-------|
| Maker account | `["maker", maker_owner_pubkey]` |
| Position | `["position", order_id]` |

`GET /api/v1/venue` returns `program_id`, which is authoritative. PDA derivation follows on-chain program logic. See the [Maker API reference](maker-api.md) and [Maker quickstart](../quickstart/maker-quickstart.md).

## On-chain maker instructions

The instruction data starts with a one-byte discriminant. Integers are little-endian. The full IDL ships in `@acta-markets/ts-sdk` as `ACTA_IDL`, and `ACTA_IDL_SHA256` is its hash.

Builders: `buildDepositPremiumIx` and `buildDepositFundsToPositionIx` from `@acta-markets/ts-sdk/chain`, or `ChainClient::deposit_premium` and `ChainClient::fund_position` in the Rust SDK (`chain-rpc` feature).

### `DepositPremium`

Moves quote tokens from the owner's wallet to the maker PDA's custody account. Fills debit this balance.

Data, 9 bytes: discriminant `4`, then `amount` as `u64`. `amount` must not be `0`.

| # | Account | Writable | Signer | Rule |
|---|---|---|---|---|
| 0 | `maker_owner` | no | yes | The registered owner |
| 1 | `maker_pda` | no | no | `["maker", maker_owner]` |
| 2 | `maker_owner_premium_ata` | yes | no | Source token account owned by `maker_owner` |
| 3 | `maker_premium_ata` | yes | no | The maker PDA's ATA for the same mint. It must already exist |
| 4 | `token_program` | no | no | The mint's token program |

### `DepositFundsToPosition`

Funds the settlement leg of an `open` covered call or cash-secured put. Spreads and cash puts are rejected.

Data, 1 byte: discriminant `9`. There are no arguments: the program computes the amount. A covered call takes `scaled_quote_amount(strike, quantity)` of the quote mint, and a cash-secured put takes `quantity` of the underlying mint.

| # | Account | Writable | Signer | Rule |
|---|---|---|---|---|
| 0 | `maker_owner` | no | yes | The position's maker owner |
| 1 | `maker_pda` | no | no | `["maker", maker_owner]` |
| 2 | `position_pda` | yes | no | `["position", order_id]` |
| 3 | `market_pda` | no | no | The position's market |
| 4 | `maker_funding_ata` | yes | no | Source token account owned by `maker_owner`, for the settlement mint |
| 5 | `pos_settlement_ata` | yes | no | The position PDA's ATA for the settlement mint. It must already exist |
| 6 | `settlement_token_program` | no | no | The settlement mint's token program |

## Support

Contact the Acta team for:

- Devnet access and maker registration
- Test token mints and faucet
- Market creation requests
- Integration support
