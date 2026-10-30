# Acta Protocol — Tech Docs

## Start here

- **Maker:** [Sandbox](reference/sandbox.md) (keypairs, registration, premium deposit), [Maker quickstart](quickstart/maker-quickstart.md), [Maker wire examples](quickstart/maker-wire-examples.md), [Maker API](reference/maker-api.md). Rust bots start with the [Rust Maker SDK](quickstart/maker-rust-sdk.md).
- **Taker:** [Taker quickstart](quickstart/taker-quickstart.md), [Taker wire examples](quickstart/taker-wire-examples.md), [Taker API](reference/taker-api.md). TypeScript apps start with the [Web client SDK](quickstart/web-client-ts-sdk.md).
- **Vault operator:** [Vault operator guide](quickstart/vault-operator-guide.md).

## Overview

- [Protocol flow](reference/protocol-flow.md) — products and lifecycles, RFQ mechanics, economics and fees, funding deadline, settlement fixing, expiries, risk

## Quickstart

- [Taker quickstart](quickstart/taker-quickstart.md) — auth, RFQ, accept, sponsored tx (language-neutral)
- [Taker wire examples](quickstart/taker-wire-examples.md) — complete JSON session + branch scenarios
- [Web client SDK (TypeScript)](quickstart/web-client-ts-sdk.md) — taker integration via TS SDK
- [Maker quickstart](quickstart/maker-quickstart.md) — auth, first quote, fills, reconnect
- [Maker wire examples](quickstart/maker-wire-examples.md) — complete JSON session
- [Rust Maker SDK](quickstart/maker-rust-sdk.md) — Rust SDK integration guide

## Vaults

- [Vault operator guide](quickstart/vault-operator-guide.md) — roles, cycle, trading options and spreads, checked signing, funding, fallbacks
- [Vault HTTP endpoints](reference/http-api.md#vaults) — vault, position, cycle and depositor reads

## Reference

- [Maker API](reference/maker-api.md) — WS messages, quote rules, events, enums
- [Taker API](reference/taker-api.md) — WS messages, RFQs, sponsored transactions
- [WS common conventions](reference/ws-common.md) — encodings, units, collateral formulas, timeouts
- [HTTP API](reference/http-api.md) — REST endpoints, reference data (venue, tokens, markets), vaults
- [Capacity limits](reference/caps.md) — OI caps, maker limits, monitoring
- [Governance and security](reference/governance.md) — cold/hot authority split, authority matrix, on-chain timelock, settlement attestation
- [Sandbox / Devnet](reference/sandbox.md) — test environment, endpoints, program addresses
- [FAQ](reference/faq.md) — common questions and troubleshooting
