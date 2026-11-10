# Agents and automation

How to run a headless client (a bot, a strategy loop or an LLM agent) against Acta. The protocol is
the same one the web app uses; this page covers what changes when nobody watches the session.
Examples use the TypeScript SDK (`@acta-markets/ts-sdk` 0.5.5 or later).

## Headless keypair

A taker needs only a Solana keypair: no registration and no on-chain setup. Load it from a file the
agent's process can read and nothing else can:

```typescript
import { readFileSync } from "node:fs";
import { KeypairAuthProvider } from "@acta-markets/ts-sdk";
import { ActaWsClient } from "@acta-markets/ts-sdk/ws";

const auth = await KeypairAuthProvider.fromJson(readFileSync(process.env.ACTA_KEYPAIR!, "utf8"));
const ws = new ActaWsClient({
  url: "wss://devnet-api.acta.markets",
  role: "taker",
  onDisconnect: "keep_quotes",
  products: ["covered_call", "cash_secured_put", "call_spread", "put_spread", "cash_put"],
  clientName: "my-agent",
  clientVersion: "0.1.0",
  autoReconnect: true,
});
ws.connectAndAuthenticate(auth);
await ws.waitForAuthenticated(30_000);
```

The same key signs the auth challenge and the sponsored transactions, so it controls the wallet's
funds. Use a dedicated wallet funded only with what the agent may trade. On devnet the faucet funds a
new wallet once; see [Sandbox](../reference/sandbox.md).

The SDK reconnects, authenticates again (resume first, a fresh signature after `session_expired`),
resubscribes and re-reads open RFQs and positions on its own. The agent does not handle any of that.

## Price before you ask

Creating an RFQ broadcasts it to makers. To learn a price without that, read indicatives and price
the fill locally:

```typescript
import { grossFromIndicative1e9, previewFill } from "@acta-markets/ts-sdk/ws";

const terms = { type: "cash_secured_put", strike: 136_000_000_000 } as const;
const fees = quoteToken.protocol_fees; // /tokens or GetTokens, per quote token
const preview = previewFill({
  terms,
  side: "sell",
  quantity: 1_000_000_000,
  // A sell indicative is already net of the fee; previewFill takes the gross price.
  price1e9: grossFromIndicative1e9(indicative.best_price, "sell", terms, fees),
  fees,
  underlyingDecimals: 9,
  quoteDecimals: 6,
});
// preview.premium.{grossAtoms, feeAtoms, netAtoms}, preview.collateral,
// preview.breakeven1e9, preview.maxProfitAtoms, preview.maxLossAtoms
```

The fee follows the program's rule: the smaller of `premium_bps` of the price and `volume_bps` of the
strike (or spread width). An indicative is the median of maker answers, net of the fee on a sell and
gross on a buy; it is not a firm quote. With a real quote, pass `QuoteReceived.price`, which is gross.
Indicatives exist for covered calls and cash-secured puts only.

## Submit, then reconcile

Two commands can take effect without the agent seeing the reply: `RfqRequest` and
`SubmitSignedSponsoredTx`. Resending either one blindly can open a second RFQ or try to fill an order
twice. The SDK owns both:

```typescript
const { rfqId, created } = await ws.createRfqOnce({
  market, terms, side: "sell", quantity, timeoutSeconds: 30,
});

const outcome = await ws.submitOrderOnce({ orderIdHex, txBase64: signedTx });
switch (outcome.type) {
  case "accepted":   // queued; follow OrderSubmitted / OrderConfirmed / OrderFailed
    break;
  case "rejected":   // this attempt was not forwarded; act on outcome.action
    break;
  case "reconciled": // the reply was lost, so the SDK looked the order up
    // outcome.state: pending | confirmed | unknown. unknown is not proof of non-execution.
    break;
}
```

- `createRfqOnce` resends a lost reply with the same id while the RFQ could still be open. A repeat is
  rejected with `duplicate_rfq_id`, which proves the first attempt created it.
- `submitOrderOnce` never sends one order twice. A timeout, a disconnect or `delivery: "unknown"` turns
  into a `GetOrderStatus` lookup, and every later call for that order only looks it up.
- Neither `OrderFailed` nor an `unknown` status authorizes signing the order again. Pick another quote
  or open a new RFQ instead.
- The SDK keeps this record in memory. After a process restart, read `GetMyActiveRfqs` and
  `GetOrderStatus` before submitting anything.

## Errors

Every error maps to one action: `retry`, `reauthenticate`, `refresh`, `reconcile`, `fix_request` or
`stop`. Branch on the action, never on `message` or `reason` text:

```typescript
import { ActaWsRequestError } from "@acta-markets/ts-sdk/ws";

try {
  await ws.acceptQuote(rfqId, maker, orderIdHex);
} catch (err) {
  if (!(err instanceof ActaWsRequestError)) throw err;
  switch (err.action) {
    case "retry": /* back off, send the same request */ break;
    case "refresh": /* fetch fresh quotes or state, build a new request */ break;
    case "reconcile": /* look the command up before anything else */ break;
    case "fix_request": /* the request is invalid; change it */ break;
    case "stop": /* market, venue or account state must change first */ break;
    case "reauthenticate": /* the SDK re-authenticates; resend after it does */ break;
  }
}
```

The action of every code is in the [WS error catalog](../reference/ws-common.md#ws-error-codes) and
the [ServerError catalog](../reference/ws-common.md#servererror-catalog). `errorAction(error, delivery)`
gives the same answer for an `Error` frame.

## Units

Most agent bugs here are unit bugs.

| Value | Unit |
|---|---|
| `quantity`, `size_rule` | Underlying atoms, for every product, including puts and spreads |
| `strike`, `price`, `net_price`, indicatives, `underlying_price` | Quote per whole underlying, scaled by 1e9 |
| `collateral`, premium totals | Atoms of the collateral mint: underlying for a covered call, quote otherwise |
| `protocol_fees` | Basis points, per quote token |

- `price` is gross. A seller receives `net_price`, the price after the protocol fee; a buyer pays
  `price`.
- A cash-secured put is sized in underlying atoms, not in quote. Convert a quote budget with
  `quoteAmountToQuantity`, then snap it to the size rule.
- Wire integers can exceed `Number.MAX_SAFE_INTEGER`. Use the SDK's bigint helpers
  (`totalPremiumQuoteAtoms`, `previewFill`) for money.

## Pace

- HTTP replies carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and, on `429`, `Retry-After`.
- `Welcome.rate_limits` lists the WebSocket budgets of the connection. The TS client paces its own
  messages by them: a message over budget waits instead of being rejected or closing the socket.
  The wallet budget is shared by all of a wallet's connections, so keep one connection per wallet.
- Positions and RFQs arrive as events; do not poll them in a loop. Re-read on reconnect, which the SDK
  already does.
