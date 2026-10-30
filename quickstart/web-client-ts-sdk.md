# Acta Web Client SDK (TypeScript)

The TS SDK wraps the taker WebSocket protocol: auth, market and position queries, RFQs for five products, and sponsored transactions.

Wire messages, errors and enums are in [`../reference/taker-api.md`](../reference/taker-api.md). SDK event callbacks mirror wire message names in lowerCamelCase. TypeScript types define the payloads.

## Installation

```bash
npm install @acta-markets/ts-sdk
```

This guide covers version 0.3.0. It speaks WS protocol `1.0.0` and targets the program deployed to devnet and mainnet; both use the same program ID, and `GET /api/v1/venue` returns the authoritative `program_id`. Refer to opcodes and errors by name through the generated constants exported from `@acta-markets/ts-sdk` (`*_DISCRIMINATOR` for opcodes, `ACTA_CONTRACT_ERROR__*` for errors), not by number. The package README and changelog cover the vault builders.

The wire is closed. The client does not dispatch a frame that lacks a required field or carries an unknown enum value. A message whose type the SDK does not know is reported on the `unknownMessage` event. Facts that are absent arrive as explicit `null`, typed `T | null`. State-dependent shapes are tagged unions. Wire addresses are validated when a frame is decoded. `server_time_unix_ms` and `sent_at_unix_ms` use milliseconds, and `expires_at` uses Unix seconds.

Taker-only apps should import **`@acta-markets/ts-sdk/ws`**: client, auth, RFQ and sponsored-tx signing, without instruction builders or IDL. The SDK is built on `@solana/kit`; you do not need `@solana/web3.js`. Spread economics helpers and account reads are in `@acta-markets/ts-sdk/chain`, and public reads such as the venue are in `@acta-markets/ts-sdk/http`.

---

## Quick start

### 1. Auth provider

`wallet` is a connected wallet adapter that exposes its current `publicKey` and `signMessage`. `walletPublicKeyBase58` is the account used for this client's authentication. Recreate the client when that account changes.

```typescript
import { WalletAuthProvider } from "@acta-markets/ts-sdk/ws";

const authProvider = new WalletAuthProvider({
  publicKeyBase58: walletPublicKeyBase58,
  signMessage: async (msg: Uint8Array) => await wallet.signMessage(msg), // 64-byte ed25519
});
```

Other providers: `KeypairAuthProvider` (Node, CI, bots; import from `@acta-markets/ts-sdk/ws/keypair`) and `CustomAuthProvider` (remote signer).

**Frontend wallets (Phantom, Privy).** WS auth requires `signMessage` (ed25519 over the UTF-8 challenge). For sponsored-tx signing, `verifyAndSignTakerFill(...)` checks the served transaction against the accepted quote and then signs with `signMessage` (no `@solana/web3.js`) or with the wallet's `signTransaction` for its own transaction preview (step 8). If a wallet does not expose `signMessage`, use `CustomAuthProvider` with a backend signer. `ActaWsClient` validates the challenge format before it calls any auth provider. Raw clients can call `validateAuthChallenge` before signing.

### 2. Connect

`onDisconnect`, `products`, `clientName` and `clientVersion` are required. The client sends them in `Hello` together with protocol version `1.0.0`.

```typescript
import { ActaWsClient } from "@acta-markets/ts-sdk/ws";

// Devnet; for mainnet use "wss://beta-api.acta.markets"
const wssEndpoint = "wss://devnet-api.acta.markets";
const ws = new ActaWsClient({
  url: wssEndpoint,
  role: "taker",
  onDisconnect: "keep_quotes",
  products: ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
  clientName: "my-app",
  clientVersion: "0.1.0",
});

// Register handlers and load the saved session before connecting (next step).

ws.on("connected", () => console.log("Connected"));
ws.on("error", (e) => console.error("Error:", e));
```

- `onDisconnect` is `"cancel_quotes"` or `"keep_quotes"`.
- `products` is a non-empty list of the five product names. The SDK sends it in every `Subscribe`. The server uses it for maker and vault sessions and does not use it to filter what a taker receives.
- The client appends `/taker` to the base URL from `role`.

Alternative: `ws.connectAnonymous()` and later `await ws.authenticate(authProvider)`.

### 3. Authenticate with session resume

```typescript
const sessionKey = `acta_session:${wssEndpoint}:${walletPublicKeyBase58}`;
const expiryKey = `${sessionKey}:expires_at`;

ws.on("authenticated", (sessionId, expiresAt, makerPda) => {
  // makerPda is null for a taker.
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

Register application handlers before calling `connectAndAuthenticate`. It tries the supplied session, then the auth provider on `session_expired`. Do not also send `ResumeAuth` or start another authentication from `connected`. `connected` means the socket is open and `Welcome` has arrived. `authenticated` means authentication and the server's connection handoff are complete.

Persist credentials per wallet and environment. Resume succeeds without a wallet popup. Fresh authentication does not restore ownership of an older credential's pending signature. See [Delivery and recovery](../reference/taker-api.md#delivery-and-recovery).

`expiresAt` is a number in Unix seconds. A saved credential with a missing or expired deadline should use fresh authentication. The deadline limits starting another resume. It does not end an already authenticated connection.

### 4. Subscribe and track positions

`subscribe(channels, mints)` takes a required mint filter. `ALL_MINTS` selects every mint.

```typescript
import { ALL_MINTS } from "@acta-markets/ts-sdk/ws";

ws.on("authenticated", () => ws.subscribe(["chain_events"], ALL_MINTS));

// A taker's own positions arrive on the session as full rows, without a subscription.
ws.on("positionUpdated", (position, updateType) => {
  // position.state: "open" | "funded" | "liquidated" | "settled"
  console.log(updateType, position.pda, position.state, position.total_premium);
});

ws.on("chainEvent", (ev) => {
  // ev.event_type, for example "PositionSettled"
  console.log("Chain event:", ev);
});
```

The client keeps `ws.state.positions` (a `Map` keyed by position PDA). It merges `Positions` responses and `PositionUpdated` pushes and applies a row only if its `revision` is newer than the stored one, so `positionUpdated` does not fire for duplicates or old rows. `comparePositionRevision(a, b)` from `@acta-markets/ts-sdk/ws` compares two revisions `(slot, transaction_index, instruction_index)` lexicographically; use it if you read raw `Positions` pages yourself. Switching the auth provider or logging out clears the cache.

Subscriptions restore on reconnect.

### 5. Browse markets

```typescript
ws.on("markets", (markets) => console.log(markets));
ws.getMarkets();

// Descriptors include products, size_rule, decimals and oracle PDAs. Fetch before createRfq.
ws.on("marketDescriptors", (descriptors) => { /* cache these */ });
ws.getMarketDescriptors(); // tradable markets only
```

Each market and descriptor carries `products`, the pair's products for that market's side. A put market (`is_put: true`) lists a subset of `cash_secured_put`, `put_spread` and `cash_put`. A call market lists a subset of `covered_call` and `call_spread`. `settlement_price` is `null` until the market settles. Offer the user only products that appear in `products`.

To list positions, pass the full filter message. `ALL_POSITIONS` selects everything:

```typescript
import { ALL_POSITIONS } from "@acta-markets/ts-sdk/ws";

ws.on("positions", (positions) => { /* PositionInfo[] */ });
ws.getPositions(ALL_POSITIONS);
// Filtered: { market: { type: "only", value: marketPda }, underlying_mint: { type: "all" },
//             status: ["open"], min_expiry_ts: { type: "unbounded" } }
```

### 6. Create RFQ

**Prerequisite:** call `getMarketDescriptors()` and wait for the `marketDescriptors` event before `createRfq`. The SDK validates `quantity` against the market's `size_rule` (`min_size <= quantity <= max_size`, `(quantity - min_size) % step === 0`). Local failures throw before send. A missing server-side rule returns `missing_size_rule_for_underlying_mint`.

`quantity` is always in underlying atoms, including for cash-secured puts, spreads and cash puts. For a UI that takes USDC input, convert with `quoteAmountToQuantity(usdc, strike1e9, underlying_decimals)` from `@acta-markets/ts-sdk/ws` (see [Quantity and collateral by product](../reference/ws-common.md#quantity-and-collateral-by-product)).

`terms` is a tagged union and `side` must be `"sell"`:

```typescript
ws.on("rfqCreated", (rfq) => console.log(rfq.rfq_id, rfq.expires_at, rfq.order_options));

ws.on("quoteReceived", (quote) => {
  // quote.terms is one of rfq.order_options. price is gross; AcceptQuote uses the gross price, hash-bound via order_id.
  // net_price is Price | null: null for spreads and cash puts.
  console.log("Quote:", quote.net_price ?? quote.price, "from", quote.maker);
});

await ws.createRfq({
  market: marketPdaBase58,
  terms: { type: "covered_call", strike: 136_000_000_000 },
  side: "sell",
  quantity: 1_000_000_000, // 1 SOL in lamports; must satisfy size_rule
  timeoutSeconds: 30,
  requestId: crypto.randomUUID(), // required; also the idempotency key
});
```

Other products:

```typescript
{ type: "cash_secured_put", strike: 136_000_000_000 }
{ type: "call_spread", lower_strike: 150_000_000_000, upper_strike: 160_000_000_000 }
{ type: "put_spread", lower_strike: 130_000_000_000, upper_strike: 140_000_000_000 }
{ type: "cash_put", strike: 136_000_000_000 }
```

Spread and cash put strikes must be multiples of the underlying's `strike_tick` (`1e9`-scaled; read it from `TokenInfo.role`). A spread needs `lower_strike < upper_strike`, and a call spread needs `lower_strike > 0`. The market's `is_put` must match the product's side.

`createRfq` resolves with `requestId` on the server's `CommandAccepted` and rejects with `ActaWsRequestError` on the matching `RequestError`, a timeout or a disconnect. `RfqCreated` echoes `request_id` and may arrive before or after that receipt. Repeating a `createRfq` with the same `requestId` returns the same `rfq_id` while the RFQ is active. The RFQ TTL is server-defined within the venue range `min_rfq_timeout_seconds` .. `max_rfq_timeout_seconds` from `GET /api/v1/venue`.

For spreads and cash puts, `order_options` holds exactly your `terms`.

### Spread economics

Spreads and cash puts are cash-settled in the quote mint. The taker posts collateral `C` and receives the net premium. At expiry the maker gets `ceil(min(intrinsic, width) * quantity)` capped at `C`, and the taker gets the rest. The `chain` entry point has the exact bigint helpers:

```typescript
import { ActaHttpClient } from "@acta-markets/ts-sdk/http";
import { SpreadType, spreadPremium, spreadSettlement } from "@acta-markets/ts-sdk/chain";

const venue = await new ActaHttpClient({ baseUrl: "https://devnet-api.acta.markets" })
  .venue(AbortSignal.timeout(5_000));

const terms = {
  positionType: SpreadType.PutSpread,
  lowerStrike: 130_000_000_000n,
  upperStrike: 140_000_000_000n,
  quantity: 5_000_000_000n,
  quoteDecimals: 6,
  underlyingDecimals: 9,
};

// collateral, grossPremium, protocolFee, netPremium (quote atoms)
const economics = spreadPremium(
  terms,
  BigInt(quote.price),                 // gross price, 0 < price < width
  venue.protocol_fee_bps_premium,
  venue.protocol_fee_bps_volume,
);

// At settlement price S: makerPayoff and takerReturn sum to economics.collateral
const { makerPayoff, takerReturn } = spreadSettlement(terms, 133_000_000_000n, economics.collateral);
```

`spreadPremium` throws when the price is not strictly between `0` and the width after rounding. The fee is `min(premium_fee, volume_fee)`. For a cash put, map the terms with `executionTerms(terms)` from `@acta-markets/ts-sdk/ws`, which returns a put spread with legs `(0, strike)`, and use `SpreadType.PutSpread` with `lowerStrike: 0n`.

### 7. Cancel RFQ

```typescript
ws.cancelRfq(rfqId);
// Listen for rfqClosed with reason="taker_cancelled"
```

### 8. Accept quote and sign

`verifyAndSignTakerFill` rebuilds the fill from the accepted quote and the RFQ's terms and calls the wallet only if the served transaction matches. It checks:

- the configured program and chain, and the market PDA;
- the order ID recomputed from the quote, quantity and taker (it must equal `quote.order_id`);
- the maker's Ed25519 signature over that order ID right before the open instruction;
- the open instruction with the quoted terms, quantity, price, validity and nonce (a spread or cash put opens with the quote-mint collateral);
- the ATA creates and, for a covered call on wSOL, a wrap of at most the collateral;
- the keeper as fee payer in slot 0, which is not the taker;
- no other signers or instructions, and a compute budget within `bounds`.

A mismatch throws `VaultIntentError` and nothing is signed.

Inputs the helper needs:

- `keeper`: the canonical `GlobalConfig.hotAuthority` address. Read it once from chain.
- `makerVault`: `await fetchMakerVault(rpc, quote.maker)`, which is a decoded `Vault` when the quote's maker is a vault PDA and `null` for an ordinary maker. A vault maker needs the account; without it the fill is verified as an ordinary maker's.
- `rfq`: a `TakerFillRfq` built from the RFQ (below). Both variants need `rfq_id`, `market` (the descriptor's `market`), `terms`, `side`, `quantity` and `order_options`. A spread or cash put also needs `underlying_decimals` and `quote_decimals` from the descriptor, and its `order_options` must hold exactly `terms`.
- `tables`: address lookup tables, `{}` for sponsored fills.

`TakerFillRfq` is a union: the physical variant (`terms` is `covered_call` or `cash_secured_put`) and the cash variant (`terms` is a spread or `cash_put`, with decimals). Build it with a function that narrows `terms`:

```typescript
import type {
  CashPutTerms, MarketDescriptorInfo, PhysicalOptionTerms, RfqCreatedMessage,
  SpreadOptionTerms, TakerFillRfq,
} from "@acta-markets/ts-sdk/ws";

function fillRfq(created: RfqCreatedMessage, descriptor: MarketDescriptorInfo, quantity: number): TakerFillRfq {
  const base = { rfq_id: created.rfq_id, market: descriptor.market, side: created.side, quantity };
  const { terms } = created;
  if (terms.type === "covered_call" || terms.type === "cash_secured_put") {
    return {
      ...base,
      terms,
      order_options: created.order_options.filter(
        (o): o is PhysicalOptionTerms => o.type === "covered_call" || o.type === "cash_secured_put",
      ),
    };
  }
  return {
    ...base,
    terms,
    order_options: created.order_options.filter(
      (o): o is SpreadOptionTerms | CashPutTerms => o.type !== "covered_call" && o.type !== "cash_secured_put",
    ),
    underlying_decimals: descriptor.underlying_decimals,
    quote_decimals: descriptor.quote_decimals,
  };
}
```

```typescript
import { address, createSolanaRpc } from "@solana/kit";
import {
  fetchGlobalConfigByAddress,
  fetchMakerVault,
  findGlobalConfigPda,
} from "@acta-markets/ts-sdk/chain";
import { ActaWsRequestError, verifyAndSignTakerFill } from "@acta-markets/ts-sdk/ws";

const rpc = createSolanaRpc(rpcUrl);
const globalConfig = await fetchGlobalConfigByAddress(rpc, await findGlobalConfigPda());
const keeper = globalConfig.account.hotAuthority;

let connectionEpoch = 0;
ws.on("stateChange", () => { connectionEpoch++; });

// acceptedQuote: the QuoteReceivedMessage you accepted; descriptor: the cached
// MarketDescriptorInfo of the RFQ's market; rfqCreated: the RfqCreated frame; requestedQuantity: what you passed to createRfq.
ws.on("sponsoredTxToSign", async (id, txBase64, signatureDeadline, lastValidBlockHeight) => {
  if (id !== orderIdHex || !ws.isAuthenticated()) return;
  const epoch = connectionEpoch;
  const walletMatches = () => wallet.publicKey?.toBase58() === walletPublicKeyBase58;
  const expired = () => Date.now() / 1000 >= signatureDeadline;
  if (!walletMatches() || expired()) return;
  let signedTxBase64: string;
  try {
    signedTxBase64 = await verifyAndSignTakerFill({
      txBase64,
      quote: acceptedQuote,
      rfq: fillRfq(rfqCreated, descriptor, requestedQuantity),
      keeper,
      taker: address(walletPublicKeyBase58),
      makerVault: await fetchMakerVault(rpc, acceptedQuote.maker),
      tables: {},
      bounds: { maxUnitLimit: 400_000, maxUnitPriceMicroLamports: 100_000 },
      signer: wallet,
    });
  } catch (error) {
    console.error(id, error); // do not submit; cancel the RFQ or pick another quote
    return;
  }
  if (epoch !== connectionEpoch || id !== orderIdHex || !walletMatches() || expired()) return;
  try {
    await ws.submitSignedSponsoredTx({ orderIdHex: id, txBase64: signedTxBase64 });
  } catch (err) {
    if (err instanceof ActaWsRequestError && err.delivery === "not_forwarded") {
      // This attempt was not sent: retry, or choose another quote if no earlier attempt was unknown.
    } else {
      ws.getOrderStatus(id); // May have been sent: resolve before anything else.
    }
  }
});

try {
  await ws.acceptQuote(rfqId, makerPubkey, orderIdHex);
} catch (err) {
  if (err instanceof ActaWsRequestError) console.error(err.kind, err.error);
}
```

`acceptQuote` resolves on `CommandAccepted` and rejects with `ActaWsRequestError` on its `RequestError`, a timeout or a disconnect; `SponsoredTxToSign` may arrive before or after the receipt. A rejected `submitSignedSponsoredTx` carries `err.delivery`. `not_forwarded` means this attempt was not sent; if every attempt for the order returned it, retrying or choosing another quote is safe. `unknown`, or no `delivery` (timeout, disconnect), means it may have been sent: query `GetOrderStatus` before signing or submitting anything for that RFQ. `lastValidBlockHeight` is the block height the served transaction's blockhash is valid through.

`verifyAndSignTakerFill` fills signature slot 1, the taker's slot. The keeper pays fees and signs slot 0, for ordinary and vault makers alike.

`orderIdHex` is the application's selected order. If the wallet, connection or selection changes during signing, discard that result. Track the sent state before submitting and use the recovery rules below when the result is uncertain.

**Browser wallets.** To show the wallet's own transaction preview and simulation, pass `signer: { signTransaction: async (wire) => (await wallet.signTransaction(VersionedTransaction.deserialize(wire))).serialize() }`. The SDK verifies the transaction before the wallet sees it and verifies what the wallet returns again (a wallet that reorders account keys passes; one that adds or changes instructions does not). Never call `wallet.signTransaction` on a served transaction directly. `@solana/web3.js` is needed only for that UX path. It is the wallet adapter's own dependency, not the SDK's.

### 9. Track order status

```typescript
ws.on("orderAccepted", (orderIdHex, orderVersion) => {});
ws.on("orderSubmitted", (orderIdHex, txSignature, orderVersion) => {});
ws.on("orderConfirmed", (orderIdHex, positionPda, orderVersion) => {});
ws.on("orderFailed", (orderIdHex, reason, orderVersion) => {});

ws.on("rfqClosed", (data) => {
  // Terminal: clean up RFQ state.
  // data.reason: "expired" | "taker_cancelled" | "filled" | "market_expired" | "ladder_timeout"
  // data.your_quote and data.winner are null when they do not apply.
});
```

### 10. Recover an interrupted order

Persist the selected `rfq_id`, maker, `order_id`, auth session and whether `SubmitSignedSponsoredTx` was sent, scoped to the wallet and backend. Keep them across reconnect and page reload, separately from the browsing cache. Do not persist signed transaction payloads for replay.

After a successful resume of the same auth session, reconcile with `GetMyActiveRfqs`. If the same order is still `pending_signature`, repeat only that exact `AcceptQuote` to retrieve its signing payload, and respect the original signature deadline. If it is `enqueued`, or the transaction was already sent, query `GetOrderStatus` instead of signing or submitting again.

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

`OrderFailed`, a timeout and `unknown` do not prove nonexecution. Do not automatically replay a trading command after losing its response. When `RfqAvailableAgain` reopens an auction, refresh the available quotes. The discarded winning quote is not an automatic retry target.

---

## Connection management

```typescript
// Devnet; for mainnet use "wss://beta-api.acta.markets"
const wssEndpoint = "wss://devnet-api.acta.markets";
const ws = new ActaWsClient({
  url: wssEndpoint,
  role: "taker",
  onDisconnect: "keep_quotes",
  products: ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
  clientName: "my-app",
  clientVersion: "0.1.0",
  autoReconnect: true,
  reconnectDelay: 1000,
  maxReconnectDelay: 30000,
  reconnectJitterRatio: 0.2,
  pingInterval: 30000,
  maxPendingMessages: 100,
  pendingMessagesOverflowPolicy: "drop_oldest", // "drop_oldest" | "drop_newest" | "throw"
});

ws.on("disconnected", (code, reason) => {});
// code: a server close code (4000–4006; see ws-common.md) or one of the client's own drops:
// 4900 "pong_timeout", 4901 "auth_error".
```

Close code `4006` (`unsupported_protocol_version`) means the server did not accept the `protocolVersion` the client sent. The default is `1.0.0`; leave it unset.

**Recovery on reconnect.** The SDK reconnects and resumes authentication after network drops. After `authenticated`, it restores the desired subscriptions and, with the default `autoReconcile`, requests `GetMyActiveRfqs` and `GetPositions` (all positions). Authentication alone does not mean these reads have completed.

Wait for those responses before rebuilding application state, and request `GetOrderStatus` for each unresolved order. Apply the exact-order recovery rules above. Do not clear an unresolved order because it is missing from an RFQ or position response.

Transport note: during reconnect, WebSocket control frames (`Ping`, `Pong`) may arrive before the first protocol JSON message. The client ignores control frames until `Welcome` or `Error`.

---

## Error handling

```typescript
ws.on("error", (e) => {
  if (e.type === "Generic") console.error(e.data.code, e.data.message);
  else console.error(e.type);
});

ws.on("requestError", (envelope) => {
  // { request_id, error: ServerError, delivery: "not_forwarded" | "unknown" | null }
});
```

Most query methods (`getMarkets`, `getPositions`, `getOrderStatus`, and others) return a `request_id`. Match it against `msg.request_id` on the corresponding response event to pair UI state with responses. `getPositionsAsync(args)` and `getMyActiveRfqsAsync()` return a promise instead.

Error codes and `OrderFailed` reasons: [taker-api.md](../reference/taker-api.md#errors). Common cases: [faq.md](../reference/faq.md).

---

## Other features

- **Invite gating (closed mainnet).** If `requireInvite` fires, redeem before trading via `redeemInvite(rawCode)`. Claim your own code with `claimReferralCode` and inspect stats with `getMyReferralInfo`. Errors: [taker-api.md](../reference/taker-api.md#invite-error-types).
- **Token caps.** `getTokenCaps(includeMarkets)` -> `tokenCaps` event. The flag is required and the server ignores it. The response carries `tokens`, `markets`, `quotes` and `quote_liabilities`, all always present. Schema: [taker-api.md](../reference/taker-api.md#token-caps) and [caps.md](../reference/caps.md).
- **Earn summary.** `getEarnSummary()` -> `earnSummary` event. APR range (`apr` is `unavailable` or `available { min, max }`) and capacity per asset for landing pages.
- **Market price snapshot.** `getTokenMarketsInfo(underlyingMint)` -> `tokenMarketsInfo` returns backend `reference_price`, size rules, decimals and indicative premiums together. See [TokenMarketsInfo](../reference/taker-api.md#tokenmarketsinfo). Correlate the response with the returned request ID and the mint you requested; the response does not echo the mint. Refresh while the view is active; the Acta web app uses 30 seconds.
- **Indicative prices.** `getIndicativePrices({ market, position_type })` -> `indicativePrices` event. Available for `covered_call` and `cash_secured_put`. A spread or cash put request returns an `Error` with `spread_invalid_strikes`. Non-binding UI reference prices; the server refreshes roughly every 30s.
- **Expiries.** `getExpiries({ underlying_mint, quote_mint, is_put })` takes `{ type: "all" }` or `{ type: "only", value }` for each filter.
- **APR inputs.** Use spot and indicative premium from the same `TokenMarketsInfo` response. Suppress the preview when the price is unavailable or the indicative has `is_stale: true`. Clear cached metadata after a failed refresh or disconnect. Pyth credentials belong on the backend, not in browser code.
- **APR and APY helper.** `computeApyFromScaledPrices({ underlyingAmount, grossPremiumPerUnit1e9, spotPrice1e9, secondsToExpiry })` from `@acta-markets/ts-sdk/ws` returns `{ premiumNotional, collateralNotional, termYield, apr, apy }`. `spotPrice1e9` is required: collateral is valued at spot for calls and puts. `computeApyFromAmounts({ underlyingAmount, premiumPerUnderlying, spotPrice, secondsToExpiry })` takes human units. For the market preview, pass backend `best_price` as `grossPremiumPerUnit1e9`: the backend applies the quote-mint fee adjustment when fee configuration is present. Do not subtract the fee a second time.
- **Fill premium.** `totalPremiumQuoteAtoms(price, quantity, underlyingDecimals, quoteDecimals)` returns the gross premium in quote atoms as a `bigint`, rounded as the contract rounds it. The net premium is `total_premium` on the position row.
- **Nonces.** `generateNonce()` from `@acta-markets/ts-sdk/ws` returns a fresh `bigint` nonce for signed orders.
- **Program events.** `decodeActaEventsFromLogsScoped(logs, programId)` from `@acta-markets/ts-sdk` decodes only events emitted by that program.
- **Instruction builders.** Builders in `@acta-markets/ts-sdk/chain` that resolve a mint's token program need `tokenProgram` or `rpc`. With neither they throw.
- **Vault catalog.** An authenticated session subscribed to `vaults` receives `vaultCatalogChanged` (`{ vault_pda }`) when a vault is created, re-delegated or closed, and `vaultProjectionAdvanced` when a vault's state changes. Neither is replayed. Re-list vaults with `GET /api/v1/vaults` (or reload `/api/v1/vaults/{pda}`) on them and after every `authenticated`.

---

## Production notes

- Sponsored transactions are v0 `VersionedTransaction`s. The `signTransaction` signer requires versioned transaction support and can show the wallet's preview. A `signMessage` signer signs raw message bytes and does not guarantee a transaction preview.
- If the wallet cannot sign arbitrary bytes, WS auth does not work directly. Use a server-side signer through `CustomAuthProvider`.
- Cache `keeper` and the market descriptors. Re-read descriptors on `referenceChanged`.

---

## Support

- **Devnet:** `wss://devnet-api.acta.markets`
- **Mainnet:** `wss://beta-api.acta.markets`
- **Questions:** contact the Acta team
