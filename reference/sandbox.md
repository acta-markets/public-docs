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
- Ensure `maker_owner` has an associated token account (ATA) for the quote mint.

### 5. Connect

```
wss://devnet-api.acta.markets/maker
wss://devnet-api.acta.markets/maker/data
```

Authentication uses the same Ed25519 challenge-response as production.

## What's available on devnet

| Feature | Details |
|---------|---------|
| Markets | Created by the Acta team; SOL/USDC pairs are typical |
| Rate limits | 30 msg/s sustained, 60 burst (same as production) |
| Caps | Test values; may differ from production |
| Indicative pricing | Supported |
| Sponsored tx signing | Supported |
| Oracle | Backend reads its configured Pyth/Hermes feeds; settlement publication writes Acta oracle accounts. Oracle readiness is checked independently of Solana cluster selection |
| HTTP API | Public read endpoints available under `/api/v1` (`/markets`, `/makers`, `/vaults`, `/stats`); participant RFQs/orders require an authenticated session |

## Differences from production

| Aspect | Devnet | Production |
|--------|--------|------------|
| Token mints | Devnet test mints (contact team) | Mainnet SPL tokens |
| Oracle feeds | Backend-configured feed IDs and oracle mapping | Backend-configured feed IDs and oracle mapping |
| Settlement | Same logic, test tokens | Real assets |
| Markets | Created on request | Dynamic |
| SOL | Free via airdrop | Real SOL required |

> The on-chain program logic is the same on devnet and production.
> Differences are external: mints, oracles, and environment config.

Browsers obtain current reference prices from the selected backend through [`GetTokenMarketsInfo`](taker-api.md#tokenmarketsinfo). They do not need Pyth credentials. Solana devnet selection does not itself choose a different Hermes endpoint or guarantee feed freshness.

## Program addresses

| Address | Value |
|---------|-------|
| Program ID | `33Ezs5eoa16QyPW8wifnyz2nCyMEcq2crqkVBNjnTE8U` |

### PDA derivation

| PDA | Seeds |
|-----|-------|
| Maker account | `["maker", maker_owner_pubkey]` |

PDA derivation follows on-chain program logic. See the [Maker API reference](maker-api.md) and [Maker quickstart](../quickstart/maker-quickstart.md).

## Support

Contact the Acta team for:

- Devnet access and maker registration
- Test token mints and faucet
- Market creation requests
- Integration support
