# Acta Web Client SDK (TypeScript)

The TS SDK wraps the taker WebSocket protocol: auth, market/position queries, RFQs, and sponsored transactions.

Wire messages, errors, and enums are in [`../reference/taker-api.md`](../reference/taker-api.md). SDK event callbacks mirror wire message names in lowerCamelCase; TypeScript types define the payloads.

## Installation

```bash
yarn add @acta-markets/ts-sdk
```

Taker-only apps should import from the **`@acta-markets/ts-sdk/ws`** subpath. It carries just the WebSocket layer (client, auth, RFQ, sponsored-tx signing) and keeps the on-chain contract layer (instruction builders, IDL) out of your bundle. The SDK is built on `@solana/kit`; you do not need `@solana/web3.js`.

---

## Quick start

### 1. Auth provider

```typescript
import { WalletAuthProvider } from "@acta-markets/ts-sdk/ws";

const authProvider = new WalletAuthProvider({
  publicKeyBase58: walletPublicKeyBase58,
  signMessage: async (msg: Uint8Array) => await wallet.signMessage(msg), // 64-byte ed25519
});
```

Other providers: `KeypairAuthProvider` (Node/CI/bots), `CustomAuthProvider` (remote signer).

**Frontend wallets (Phantom / Privy).** WS auth requires `signMessage` (ed25519 over the UTF-8 challenge). For sponsored-tx signing, `signSponsoredTxBase64(...)` signs the raw message bytes (no `@solana/web3.js`); or call `wallet.signTransaction(...)` if you want the wallet's own tx preview. If a wallet doesn't expose `signMessage`, use `CustomAuthProvider` with a backend signer.

### 2. Connect

```typescript
import { ActaWsClient } from "@acta-markets/ts-sdk/ws";

// Devnet; for mainnet use "wss://beta-api.acta.markets"
const wssEndpoint = "wss://devnet-api.acta.markets";
const ws = new ActaWsClient({ url: wssEndpoint, role: "taker" });

ws.connectAndAuthenticate(authProvider);

ws.on("connected", () => console.log("Connected"));
ws.on("error", (e) => console.error("Error:", e));
```

The client appends `/taker` to the base URL from `role`. Alternative: `ws.connectAnonymous()` then `await ws.authenticate(authProvider)` later.

### 3. Authenticate with session resume

```typescript
ws.on("authenticated", (sessionId, expiresAt) => {
  localStorage.setItem("acta_session_id", sessionId);
  localStorage.setItem("acta_session_expires_at", String(expiresAt));
});

const savedSessionId = localStorage.getItem("acta_session_id");
const savedExpiresAt = Number(localStorage.getItem("acta_session_expires_at") || "0");

if (savedSessionId && Date.now() / 1000 < savedExpiresAt) {
  ws.resumeAuth(savedSessionId); // no wallet popup
} else {
  await ws.authenticate(authProvider); // full sign flow
}

ws.on("authError", async (reason, message) => {
  if (reason === "session_expired") {
    localStorage.removeItem("acta_session_id");
    localStorage.removeItem("acta_session_expires_at");
    await ws.authenticate(authProvider);
  }
});
```

`resumeAuth()` sends `ResumeAuth`. A valid session returns `AuthSuccess`; an expired or revoked session returns `AuthError`. No wallet popup is needed when resume succeeds.

### 4. Subscribe to live updates

```typescript
// Takers track position outcomes via chain events; `positions`/`positionUpdated`
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
ws.getMarketDescriptors({ active_only: true });
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

ws.createRfq({
  market: marketPdaBase58,
  position_type: "covered_call",
  strike: 136_000_000_000,
  quantity: 5_000_000_000, // 5 SOL in lamports
  timeoutSeconds: 30,
  clientRequestId: uuid(), // optional idempotency key, scoped per taker
});
```

Repeating a `createRfq` with the same `clientRequestId` returns the same `rfq_id` while the RFQ is active. TTL is server-defined.

### 7. Cancel RFQ

```typescript
ws.cancelRfq(rfqId);
// Listen for rfqClosed with reason="taker_cancelled"
```

### 8. Accept quote and sign

```typescript
import { signSponsoredTxBase64 } from "@acta-markets/ts-sdk/ws";

ws.on("sponsoredTxToSign", async (orderIdHex, txBase64, signatureDeadline) => {
  // No @solana/web3.js: the helper signs the tx message bytes into the taker's
  // signature slot with ed25519. Sign before signatureDeadline (unix seconds).
  // `taker` is a KeypairSigner (bots) or a wallet exposing signMessage(bytes).
  const signedTxBase64 = await signSponsoredTxBase64({ txBase64, taker });

  await ws.submitSignedSponsoredTx({ orderIdHex, txBase64: signedTxBase64 });
});

ws.acceptQuote(rfqId, makerPubkey, orderIdHex);
```

**Browser wallets.** To show the wallet's own transaction preview/simulation, deserialize with `@solana/web3.js` and call `wallet.signTransaction(tx)` instead. `@solana/web3.js` is only needed for that UX path — it's the wallet adapter's own dependency, not the SDK's.

### 9. Track order status

```typescript
ws.on("orderAccepted", (orderIdHex) => {});
ws.on("orderSubmitted", (orderIdHex, txSignature) => {});
ws.on("orderConfirmed", (orderIdHex, positionPda) => {});
ws.on("orderFailed", (orderIdHex, reason) => {});

ws.on("rfqClosed", (data) => {
  // Terminal — clean up RFQ state. data.reason: "taker_cancelled" | "expired" | "filled" | ...
});
```

### 10. Auto-retry on blockhash expiry

Under Solana congestion, a sponsored tx can exceed its blockhash validity. The server retries internally up to 5 times; if all fail, you get `OrderFailed` with `reason` containing `"blockhash_expired"`, and the RFQ reopens via `RfqAvailableAgain`. You can re-accept the same quote automatically:

```typescript
const BLOCKHASH_MAX_RETRIES = 3;
const blockhashRetries = new Map<string, number>();

ws.on("orderFailed", (orderIdHex, reason) => {
  if (!reason.includes("blockhash_expired")) return;
  const count = (blockhashRetries.get(orderIdHex) ?? 0) + 1;
  if (count > BLOCKHASH_MAX_RETRIES) { blockhashRetries.delete(orderIdHex); return; }
  blockhashRetries.set(orderIdHex, count);
  // RfqAvailableAgain arrives shortly - re-accept there
});

ws.on("rfqAvailableAgain", (data) => {
  if (pendingRetryRfqId === data.rfq_id) {
    ws.acceptQuote(data.rfq_id, lastMaker, lastOrderIdHex);
  }
});
```

Other `OrderFailed` reasons (`on_chain:`, `submission_rejected:`, `shutdown`) are not recoverable by retrying the same quote. Show the error to the user. Reason catalog: [taker-api.md](../reference/taker-api.md).

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

**Recovery on reconnect.** The SDK reconnects after network drops, but not after `VersionMismatch`. Subscriptions are restored. After `connected`:

1. `resumeAuth(savedSessionId)` or `authenticate(authProvider)`.
2. `getMyActiveRfqs()` to reconcile in-flight RFQs.
3. `getOrderStatus(orderIdHex)` for any pending orders.
4. `getPositions()`, `getMarkets()` to refresh view state.

Transport note: during reconnect, WebSocket control frames (`Ping`/`Pong`) may arrive before the first protocol JSON message. The client ignores control frames until `Welcome`, `VersionMismatch`, or `Error`.

---

## Error handling

```typescript
ws.on("error", (e) => {
  // ServerError object (not a JS Error). Check e.type: typed variants have e.type !== "generic";
  // generic errors carry e.data.code and e.data.message.
});

ws.on("requestError", (envelope) => {
  // { request_id, error: ServerError } - correlates with a specific request
});
```

Most query methods (`getMarkets`, `getPositions`, `getOrderStatus`, ...) return a `request_id`; match it against `msg.request_id` on the corresponding response event to pair UI state with responses.

Error codes and `OrderFailed` reasons: [taker-api.md](../reference/taker-api.md). Common cases: [faq.md](../reference/faq.md).

---

## Other features

Each helper mirrors a wire message in the API reference.

- **Invite gating (closed mainnet).** If `requireInvite` fires, redeem before trading via `redeemInvite(rawCode)`; claim your own code via `claimReferralCode`; inspect stats via `getMyReferralInfo`. Errors: [taker-api.md](../reference/taker-api.md).
- **Token caps.** `getTokenCaps()` -> `tokenCaps` event. OI and notional capacity per token. Schema: [caps.md](../reference/caps.md).
- **Earn summary.** `getEarnSummary()` -> `earnSummary` event. APR ranges and capacity per asset for landing pages.
- **Indicative prices.** `getIndicativePrices({ market, position_type })` -> `indicativePrices` event. Non-binding UI reference prices; server refreshes roughly every 30s.
- **APR/APY helper.** `ws.computeApyFromScaledPrices({ positionType, underlyingAmount, grossPremiumPerUnit1e9, strike1e9, spotPrice1e9, secondsToExpiry })` returns `{ apy, apr, termYield }`.

---

## Production notes

- Sponsored transactions are **v0 VersionedTransaction**; wallet must support versioned tx signing. Signing shows the wallet's preview/simulation UI.
- If the wallet can't sign arbitrary bytes, WS auth won't work directly - use a server-side signer via `CustomAuthProvider`.

---

## Support

- **Devnet:** `wss://devnet-api.acta.markets`
- **Mainnet:** `wss://beta-api.acta.markets`
- **Questions:** contact the Acta team
