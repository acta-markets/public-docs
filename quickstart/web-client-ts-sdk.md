# Acta Web Client SDK (TypeScript)

The TS SDK wraps the taker WebSocket protocol: auth, market/position queries, RFQs, and sponsored transactions.

Wire messages, errors, and enums are in [`../reference/taker-api.md`](../reference/taker-api.md). SDK event callbacks mirror wire message names in lowerCamelCase; TypeScript types define the payloads.

## Installation

```bash
npm install @acta-markets/ts-sdk
```

This guide covers version 0.2.2, which targets the program deployed to devnet and mainnet on 2026-10-17. Its instruction opcodes and program error codes differ from the program 0.1.6 was built against, so upgrade both together; the package README has a "Migrating from 0.1.6" table. Refer to opcodes and errors by name through the generated constants exported from `@acta-markets/ts-sdk` (`*_DISCRIMINATOR` for opcodes, `ACTA_CONTRACT_ERROR__*` for errors), not by number.

**Vault payout fields were renamed in 0.2.0.** Deposit payouts take `refundTo` / `sharesTo` and withdrawal payouts `sharesBackTo`; `liquidateVaultPosition(liquidator, { owner, paidTo }, position)` replaces the old fourth destination argument. JavaScript silently ignores the old `refundDestination` / `shareDestination` fields and a fourth argument, so the funds go to the canonical ATAs instead of the destination you meant.
`ActaWsClient` validates challenge format before calling any auth provider;
raw clients can call `validateAuthChallenge` before signing.

The client requires these fields in server messages:
`Welcome.server_time_unix_ms`, `AuthSuccess.expires_at`,
`RfqBroadcast.sent_at_unix_ms`, `order_version` on order lifecycle events,
and `instruction_index` on every known chain event.
Frames missing these fields or containing `null` are rejected before
dispatch. `server_time_unix_ms` and `sent_at_unix_ms` use milliseconds;
`expires_at` uses Unix seconds. The WS protocol version is `1.0.0`.

Taker-only apps should import **`@acta-markets/ts-sdk/ws`**: client, auth, RFQ and sponsored-tx signing, without instruction builders or IDL. The SDK is built on `@solana/kit`; you do not need `@solana/web3.js`.

---

## Quick start

### 1. Auth provider

`wallet` is a connected wallet adapter exposing its current `publicKey` and `signMessage`; `walletPublicKeyBase58` is the account used for this client's authentication. Recreate the client when that account changes.

```typescript
import { WalletAuthProvider } from "@acta-markets/ts-sdk/ws";

const authProvider = new WalletAuthProvider({
  publicKeyBase58: walletPublicKeyBase58,
  signMessage: async (msg: Uint8Array) => await wallet.signMessage(msg), // 64-byte ed25519
});
```

Other providers: `KeypairAuthProvider` (Node/CI/bots), `CustomAuthProvider` (remote signer).

**Frontend wallets (Phantom / Privy).** WS auth requires `signMessage` (ed25519 over the UTF-8 challenge). For sponsored-tx signing, `verifyAndSignTakerFill(...)` checks the served transaction against the accepted quote and then signs with `signMessage` (no `@solana/web3.js`) or with the wallet's `signTransaction` for its own tx preview (step 8). If a wallet doesn't expose `signMessage`, use `CustomAuthProvider` with a backend signer.

### 2. Connect

```typescript
import { ActaWsClient } from "@acta-markets/ts-sdk/ws";

// Devnet; for mainnet use "wss://beta-api.acta.markets"
const wssEndpoint = "wss://devnet-api.acta.markets";
const ws = new ActaWsClient({ url: wssEndpoint, role: "taker" });

// Register handlers and load the saved session before connecting (next step).

ws.on("connected", () => console.log("Connected"));
ws.on("error", (e) => console.error("Error:", e));
```

The client appends `/taker` to the base URL from `role`. Alternative: `ws.connectAnonymous()` then `await ws.authenticate(authProvider)` later.

### 3. Authenticate with session resume

```typescript
const sessionKey = `acta_session:${wssEndpoint}:${walletPublicKeyBase58}`;
const expiryKey = `${sessionKey}:expires_at`;

ws.on("authenticated", (sessionId, expiresAt) => {
  localStorage.setItem(sessionKey, sessionId);
  localStorage.setItem(expiryKey, String(expiresAt));
});

const savedSessionId = localStorage.getItem(sessionKey);
const savedExpiresAt = Number(localStorage.getItem(expiryKey) || "0");

const sessionId = savedSessionId && Date.now() / 1000 < savedExpiresAt
  ? savedSessionId
  : undefined;

ws.connectAndAuthenticate(authProvider, { sessionId });
```

Register application handlers before calling `connectAndAuthenticate`. It tries the supplied session, then the auth provider on `session_expired`. Do not also send `ResumeAuth` or start another authentication from `connected`. `connected` means the socket is open, before the Welcome/auth exchange has completed; `authenticated` means authentication and the server's connection handoff are complete.

Persist credentials per wallet and environment. Resume succeeds without a wallet popup. Fresh authentication does not restore ownership of an older credential's pending signature. See [Delivery & recovery](../reference/taker-api.md#delivery-and-recovery).

`expiresAt` is always a number in Unix seconds. A saved credential with a missing,
`null`, or expired deadline should use fresh authentication. The deadline limits
starting another resume; it does not end an already authenticated connection.

### 4. Subscribe to live updates

```typescript
// Takers track position outcomes via chain events; `positionUpdated`
// is a maker-only push and never fires for a taker.
ws.on("authenticated", () => ws.subscribe(["chain_events"]));

ws.on("chainEvent", (ev) => {
  // ev.event_type e.g. "PositionSettled" / "PositionLiquidated" — match to your position_pda
  console.log("Chain event:", ev);
});

// After a fill, refresh authoritative state with getPositions().
```

Subscriptions auto-restore on reconnect.

### 5. Browse markets

```typescript
ws.on("markets", (markets) => console.log(markets));
ws.getMarkets();

// Full descriptors include size_rule, decimals, and oracle PDAs. Fetch before createRfq.
ws.on("marketDescriptors", (descriptors) => { /* cache these */ });
ws.getMarketDescriptors(); // tradable markets only
```

### 6. Create RFQ

**Prerequisite:** call `getMarketDescriptors()` before `createRfq`. The SDK validates `quantity` against the market's `size_rule` (`min_size <= quantity <= max_size`, `(quantity - min_size) % step === 0`). Local failures throw before send; a missing server-side rule returns `missing_size_rule_for_underlying_mint`.

`quantity` is always in **underlying atomic units**, including for cash-secured puts. For CSP UIs that take USDC input, convert with `quoteAmountToQuantity(usdc, strike1e9, underlying_decimals)` from `@acta-markets/ts-sdk/ws` (see [CSP conversion in ws-common.md](../reference/ws-common.md)).

```typescript
ws.on("rfqCreated", (rfq) => console.log(rfq.rfq_id, rfq.expires_at));

ws.on("quoteReceived", (quote) => {
  // Display net_price (after protocol fee); AcceptQuote uses gross price (hash-bound via order_id).
  const display = quote.net_price ?? quote.price;
  console.log("Quote:", display, "from", quote.maker);
});

const requestId = crypto.randomUUID(); // idempotency key; generated when omitted
await ws.createRfq({
  market: marketPdaBase58,
  position_type: "covered_call",
  strike: 136_000_000_000,
  quantity: 5_000_000_000, // 5 SOL in lamports
  timeoutSeconds: 30,
  requestId,
});
```

`createRfq` resolves with `requestId` on the server's `CommandAccepted` and rejects with `ActaWsRequestError` on the matching `RequestError`, a timeout or a disconnect. `RfqCreated` echoes `request_id` and may arrive before or after that receipt. Repeating a `createRfq` with the same `requestId` returns the same `rfq_id` while the RFQ is active. TTL is server-defined.

### 7. Cancel RFQ

```typescript
ws.cancelRfq(rfqId);
// Listen for rfqClosed with reason="taker_cancelled"
```

### 8. Accept quote and sign

`verifyAndSignTakerFill` rebuilds the fill from the accepted quote and the RFQ's
terms and calls the wallet only if the served transaction matches: the configured
program and chain, the market PDA, the order id recomputed from the quote, quantity
and taker (it must equal `quote.order_id`), the maker's Ed25519 signature over that
order id right before `OpenPosition`, `OpenPosition` with the quoted strike,
quantity, price, validity and nonce, the ATA creates (and, for a covered call on
wSOL, a wrap of at most the collateral), a relayer fee payer in slot 0 that is not
the taker, no other signers or instructions, and a compute budget within `bounds`.
A mismatch throws `VaultIntentError` and nothing is signed. ALT references resolve
from `tables`.

```typescript
import { verifyAndSignTakerFill } from "@acta-markets/ts-sdk/ws";

let connectionEpoch = 0;
ws.on("stateChange", () => { connectionEpoch++; });

// acceptedQuote: the QuoteReceivedMessage you accepted; descriptor: the cached
// MarketDescriptorInfo of the RFQ's market; rfqRequest: what you passed to createRfq.
ws.on("sponsoredTxToSign", async (id, txBase64, signatureDeadline) => {
  if (id !== orderIdHex || !ws.isAuthenticated()) return;
  const epoch = connectionEpoch;
  const walletMatches = () => wallet.publicKey?.toBase58() === walletPublicKeyBase58;
  const expired = () => Date.now() / 1000 >= signatureDeadline;
  if (!walletMatches() || expired()) return;
  try {
    const signedTxBase64 = await verifyAndSignTakerFill({
      txBase64,
      quote: acceptedQuote,
      rfq: {
        market: descriptor.market,
        position_type: rfqRequest.position_type,
        quantity: rfqRequest.quantity,
      },
      taker: walletPublicKeyBase58,
      tables: {},
      bounds: { maxUnitLimit: 400_000, maxUnitPriceMicroLamports: 100_000 },
      signer: wallet,
    });
    if (epoch !== connectionEpoch || id !== orderIdHex || !walletMatches() || expired()) return;
    await ws.submitSignedSponsoredTx({ orderIdHex: id, txBase64: signedTxBase64 });
  } catch (error) {
    console.error(id, error); // do not submit; cancel the RFQ or pick another quote
  }
});

ws.acceptQuote(rfqId, makerPubkey, orderIdHex);
```

`verifyAndSignTakerFill` fills signature slot 1, the taker's slot on an ordinary maker quote: the keeper pays fees and signs slot 0. When the quote's `maker` is a vault PDA, the taker is the fee payer and only signer: pass `makerVault: await fetchMakerVault(rpc, quote.maker)` (from `@acta-markets/ts-sdk/chain`, `null` for a non-vault maker) and the same call verifies the vault fill against that on-chain account and signs slot 0.

`orderIdHex` is the application's selected order. If the wallet, connection or selection changes during signing, discard that result. Track sent state before submitting and use the recovery rules below when the result is uncertain.

**Browser wallets.** To show the wallet's own transaction preview/simulation, pass `signer: { signTransaction: async (wire) => (await wallet.signTransaction(VersionedTransaction.deserialize(wire))).serialize() }`. The SDK verifies the transaction before the wallet sees it and verifies what the wallet returns again (a wallet that reorders account keys passes; one that adds or changes instructions does not). Never call `wallet.signTransaction` on a served transaction directly. `@solana/web3.js` is only needed for that UX path — it's the wallet adapter's own dependency, not the SDK's.

### 9. Track order status

```typescript
ws.on("orderAccepted", (orderIdHex, orderVersion) => {});
ws.on("orderSubmitted", (orderIdHex, txSignature, orderVersion) => {});
ws.on("orderConfirmed", (orderIdHex, positionPda, orderVersion) => {});
ws.on("orderFailed", (orderIdHex, reason, orderVersion) => {});

ws.on("rfqClosed", (data) => {
  // Terminal — clean up RFQ state. data.reason: "taker_cancelled" | "expired" | "filled" | ...
});
```

### 10. Recover an interrupted order

Persist the selected `rfq_id`, maker, `order_id`, auth session and whether `SubmitSignedSponsoredTx` was sent, scoped to the wallet and backend. Keep them across reconnect and page reload, separately from the browsing cache. Do not persist signed transaction payloads for replay.

After successful resume of the same auth session, reconcile with `GetMyActiveRfqs`. If the same order is still `pending_signature`, repeat only that exact `AcceptQuote` to retrieve its signing payload; respect the original signature deadline. If it is `enqueued`, or the transaction was already sent, query `GetOrderStatus` instead of signing or submitting again.

```typescript
ws.on("orderStatus", ({ order_id, state }) => {
  switch (state.type) {
    case "confirmed":
      console.log(order_id, "opened position", state.position_pda);
      break;
    case "pending":
      console.log(order_id, "execution pending");
      break;
    case "unknown":
      console.log(order_id, "outcome unresolved");
      break;
  }
});
```

`OrderFailed`, timeout and `unknown` do not prove nonexecution. Do not automatically replay a trading command after losing its response. When `RfqAvailableAgain` reopens an auction, refresh the available quotes; the discarded winning quote is not an automatic retry target.

---

## Connection management

```typescript
// Devnet; for mainnet use "wss://beta-api.acta.markets"
const wssEndpoint = "wss://devnet-api.acta.markets";
const ws = new ActaWsClient({
  url: wssEndpoint,
  role: "taker",
  autoReconnect: true,
  reconnectDelay: 1000,
  maxReconnectDelay: 30000,
  reconnectJitterRatio: 0.2,
  pingInterval: 30000,
  protocolVersion: "1.0.0",
  maxPendingMessages: 100,
  pendingMessagesOverflowPolicy: "drop_oldest", // "drop_oldest" | "drop_newest" | "throw"
});

ws.on("disconnected", (code, reason) => {});
ws.on("versionMismatch", (msg) => {
  // Auto-reconnect stops on VersionMismatch until explicit reconnect
});
```

**Recovery on reconnect.** The SDK reconnects and resumes authentication after network drops, but not after `VersionMismatch`. After `authenticated`, it restores desired subscriptions and, with the default `autoReconcile`, requests `GetMyActiveRfqs` and `GetPositions`. Authentication alone does not mean these reads have completed.

Wait for those responses before rebuilding application state, and request `GetOrderStatus` for each unresolved order. Apply the exact-order recovery rules above. Do not clear an unresolved order because it is missing from an RFQ or position response.

Transport note: during reconnect, WebSocket control frames (`Ping`/`Pong`) may arrive before the first protocol JSON message. The client ignores control frames until `Welcome`, `VersionMismatch`, or `Error`.

---

## Error handling

```typescript
ws.on("error", (e) => {
  if (e.type === "Generic") console.error(e.data.code, e.data.message);
  else console.error(e.type);
});

ws.on("requestError", (envelope) => {
  // { request_id, error: ServerError } - correlates with a specific request
});
```

Most query methods (`getMarkets`, `getPositions`, `getOrderStatus`, ...) return a `request_id`; match it against `msg.request_id` on the corresponding response event to pair UI state with responses.

Error codes and `OrderFailed` reasons: [taker-api.md](../reference/taker-api.md). Common cases: [faq.md](../reference/faq.md).

---

## Other features

- **Invite gating (closed mainnet).** If `requireInvite` fires, redeem before trading via `redeemInvite(rawCode)`; claim your own code via `claimReferralCode`; inspect stats via `getMyReferralInfo`. Errors: [taker-api.md](../reference/taker-api.md).
- **Token caps.** `getTokenCaps()` -> `tokenCaps` event. OI and notional capacity per token. Schema: [caps.md](../reference/caps.md).
- **Earn summary.** `getEarnSummary()` -> `earnSummary` event. APR ranges and capacity per asset for landing pages.
- **Market price snapshot.** `getTokenMarketsInfo(underlyingMint)` -> `tokenMarketsInfo` returns backend `reference_price`, size rules, decimals and indicative premiums together. See [TokenMarketsInfo](../reference/taker-api.md#tokenmarketsinfo). Correlate the response with the returned request ID and the mint you requested; the response does not echo the mint. Refresh while the view is active; the Acta web app uses 30 seconds.
- **Indicative prices.** `getIndicativePrices({ market, position_type })` -> `indicativePrices` event. Non-binding UI reference prices; server refreshes roughly every 30s.
- **APR inputs.** Use spot and indicative premium from the same `TokenMarketsInfo` response. Suppress the preview when the price is unavailable or the indicative has `is_stale: true`; clear cached metadata after a failed refresh or disconnect. Pyth credentials belong on the backend, not in browser code.
- **APR/APY helper.** `computeApyFromScaledPrices({ underlyingAmount, grossPremiumPerUnit1e9, spotPrice1e9, secondsToExpiry })` from `@acta-markets/ts-sdk/ws` returns `{ premiumNotional, collateralNotional, termYield, apr, apy }`. `spotPrice1e9` is required: collateral is valued at spot for calls and puts. `computeApyFromAmounts({ underlyingAmount, premiumPerUnderlying, spotPrice, secondsToExpiry })` takes human units. For the market preview, pass backend `best_price` as `grossPremiumPerUnit1e9`: the backend applies the quote-mint fee adjustment when fee configuration is present. Do not subtract the fee a second time.

- **Nonces.** `generateNonce()` from `@acta-markets/ts-sdk/ws` returns a fresh `bigint` nonce for signed orders.
- **Program events.** `decodeActaEventsFromLogsScoped(logs, programId)` from `@acta-markets/ts-sdk` decodes only events emitted by that program.
- **Instruction builders.** Builders in `@acta-markets/ts-sdk/chain` that resolve a mint's token program need `tokenProgram` or `rpc`; with neither they throw.
- **Vault catalog.** An authenticated session subscribed to `vaults` receives `vaultCatalogChanged` (`{ vault_pda }`) when a vault is created, re-delegated or closed, and `vaultProjectionAdvanced` when a vault's state changes. Neither is replayed: re-list vaults with `GET /api/v1/vaults` (or reload `/api/v1/vaults/{pda}`) on them and after every `authenticated`.

---

## Production notes

- Sponsored transactions are **v0 VersionedTransaction**. The `signTransaction` signer requires versioned transaction support and can show the wallet's preview; a `signMessage` signer signs raw message bytes and does not guarantee a transaction preview.
- If the wallet can't sign arbitrary bytes, WS auth won't work directly - use a server-side signer via `CustomAuthProvider`.

---

## Support

- **Devnet:** `wss://devnet-api.acta.markets`
- **Mainnet:** `wss://beta-api.acta.markets`
- **Questions:** contact the Acta team
