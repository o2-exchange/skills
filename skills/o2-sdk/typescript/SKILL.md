---
name: o2-sdk-typescript
description: "TypeScript reference for using the O2 SDK: installation, client setup, owner signer setup, trading account creation, session creation, market lookup, balances, order actions, nonce/session recovery, and account-action basics. Use when a user asks how to integrate O2 in TypeScript, create sessions, trade on O2, place or cancel orders, settle balances, inspect balances/orders, or prepare account actions used by bridge withdrawals."
---

# O2 SDK TypeScript

Use this skill for O2 TypeScript setup and trading flows. Bridge-specific deposit and withdrawal skills should reference this skill for account, session, signer, market, and order setup.

This skill is grounded in `@o2exchange/sdk@0.1.0` public exports. If a method is from a local extension/helper rather than the stock SDK, say that explicitly.

For full REST/signing details, read `../../../references/o2-skills.md` and `../../../references/o2-reference.md`.

## Install

```bash
pnpm add @o2exchange/sdk @o2exchange/contracts fuels ethers
```

## Identity Model

O2 has three identities. Do not collapse them.

```text
Owner signer      controls account setup, session creation, withdrawals/account actions
trade_account_id  Fuel contract account that holds funds
Session key       short-lived trading key for place/cancel/settle actions
```

Rules:

- Owner signs session creation and privileged account actions.
- Session signs order actions.
- Session key cannot withdraw.
- `trade_account_id` is a Fuel contract ID, not the owner address.

## Client Setup

```ts
import { Network, O2Client, getNetworkConfig } from "@o2exchange/sdk";

const client = new O2Client({
  config: {
    apiBase: "https://api.o2.app",
    wsUrl: "wss://api.o2.app/v1/ws",
    fuelRpc: FUEL_RPC_URL,
    faucetUrl: undefined,
  },
});
```

If the integration needs chain ID checks or helper methods, keep it on the client wrapper rather than scattering network constants through trading code.

```ts
const config = getNetworkConfig(Network.MAINNET);
const client = new O2Client({ config });
```

Stock shortcut:

```ts
const client = new O2Client({ network: Network.MAINNET });
```

## Owner EOA Signer

Use the SDK wallet helpers for EOA private-key signers.

```ts
import { O2Client } from "@o2exchange/sdk";

const ownerSigner =
  walletType === "fuel"
    ? O2Client.loadWallet(privateKey)
    : O2Client.loadEvmWallet(privateKey);

console.log(ownerSigner.b256Address);
```

Fuel EOA and EVM EOA owners are both secp256k1, but they do not sign the same digest and they do not derive the same `b256Address`.

```text
Fuel EOA:
  helper:      O2Client.loadWallet(privateKey)
  owner b256:  Fuel b256 address derived from the Fuel key
  signing:     Fuel personalSign digest

EVM EOA:
  helper:      O2Client.loadEvmWallet(privateKey)
  owner b256:  20-byte EVM address left-padded to 32 bytes
  signing:     Ethereum personal_sign digest
```

For O2 account setup, session creation, stock `client.withdraw(...)`, and custom account actions, always sign with the same owner EOA type that created the O2 account. Do not use `loadWallet` for an EVM owner key and do not use `loadEvmWallet` for a Fuel owner key.

The O2 signing bytes are the same shape for both owner types; the digest wrapper changes. Call `ownerSigner.personalSign(signingBytes)` and let the SDK signer choose the digest:

- `loadWallet(...).personalSign(...)` uses Fuel personal signing.
- `loadEvmWallet(...).personalSign(...)` uses Ethereum `personal_sign`.
- The submitted signature envelope is still `Secp256k1`; do not invent an EVM-specific envelope.
- Do not pre-hash or double-prefix the message before calling `personalSign`.

## Account Setup

Create or load the trading account before trading or withdrawing.

```ts
const account = await client.setupAccount(ownerSigner);
const tradeAccountId = account.tradeAccountId;
```

If you only have an owner address:

```ts
const accountInfo = await client.api.getAccount({
  owner: ownerSigner.b256Address,
});

if (!accountInfo.trade_account_id) {
  throw new Error("No O2 trading account found. Run setupAccount(ownerSigner) first.");
}

const tradeAccountId = accountInfo.trade_account_id;
```

## Market Lookup

Fetch market metadata before creating orders. Market metadata controls contract IDs, assets, decimals, and precision.

```ts
const market = await client.getMarket("ETH/USDC");
```

Keep the returned `market` object and pass it to order calls when possible. Avoid hardcoding market IDs and decimals unless the user explicitly pins them.

## Session Creation

Create a session when the app needs to place/cancel/settle orders.

```ts
const account = await client.setupAccount(ownerSigner);
const market = await client.getMarket("ETH/USDC");

const session = await client.createSession(
  ownerSigner,
  [market],
  30, // expiry in days
);

console.log({
  tradeAccountId: account.tradeAccountId,
  sessionAddress: session.sessionAddress,
  expiresAt: new Date(session.expiry * 1000).toISOString(),
});
```

Session creation is owner-signed. After session creation, order actions are session-signed by the SDK.

Before reusing a session, check expiry and refresh if it expires within about 60 seconds.

```ts
const expiresSoon = session.expiry * 1000 <= Date.now() + 60_000;
if (expiresSoon) {
  await client.createSession(ownerSigner, [market], 30);
}
```

## Balances

Read balances from the trading account contract.

```ts
const balances = await client.getBalances(tradeAccountId);
console.log(balances);
```

Use `tradeAccountId`, not owner `b256`, when the question is about funds available inside O2.

## Trading Actions

Use `createOrder` for normal SDK trading. Prefer `settleFirst: true` in balance-sensitive flows.

```ts
const result = await client.createOrder(
  market,
  "buy",
  "2500.00",
  "0.1",
  {
    orderType: "Spot",
    settleFirst: true,
    collectOrders: true,
  },
);

if (!result.success) {
  throw new Error(result.reason ?? result.message ?? "O2 order failed");
}

const orderId = result.orders?.[0]?.order_id;
```

Bounded market example:

```ts
await client.createOrder(
  market,
  "sell",
  "2500.00",
  "0.1",
  {
    orderType: {
      BoundedMarket: {
        max_price: "2600.00",
        min_price: "2400.00",
      },
    },
    settleFirst: true,
    collectOrders: true,
  },
);
```

Batch action example using stock SDK helpers:

```ts
import {
  createOrderAction,
  settleBalanceAction,
  cancelOrderAction,
} from "@o2exchange/sdk";

const result = await client.batchActions(
  [
    {
      market: "ETH/USDC",
      actions: [
        settleBalanceAction(),
        createOrderAction("buy", "2500.00", "0.1", "Spot"),
      ],
    },
  ],
  true,
);
```

Cancel and settle helpers:

```ts
await client.cancelOrder(orderId, market);
await client.settleBalance(market);
```

## Order and Nonce Recovery

Common recovery rules:

- If the session is missing or expired, create a new session with the owner signer.
- If nonce is stale, call `client.refreshNonce()` and retry once.
- If a cancel targets a filled order, remove that cancel and rebuild the batch.
- Keep `collectOrders: true` when the caller needs order IDs.

Example retry shape:

```ts
let result = await client.createOrder(market, "buy", price, quantity, {
  orderType: "Spot",
  settleFirst: true,
  collectOrders: true,
});

if (!result.success && /nonce/i.test(result.reason ?? result.message ?? "")) {
  await client.refreshNonce();
  result = await client.createOrder(market, "buy", price, quantity, {
    orderType: "Spot",
    settleFirst: true,
    collectOrders: true,
  });
}
```

## Account Actions

Use owner signing for privileged account actions.

Account action payloads use:

```text
owner signer
trade_account_id
fresh account nonce
action payload
signature
```

When implementing a custom account action, prefer SDK/helper methods over hand-encoding signing bytes unless the user explicitly needs a low-level implementation.

Stock SDK withdrawal:

```ts
const response = await client.withdraw(
  ownerSigner,
  "fUSDC",
  "100.0",
  destinationB256, // optional; defaults to owner signer address
);
```

`client.withdraw(...)` is an owner-signed O2 account withdrawal to a Fuel identity. It is not the EVM fast-bridge withdrawal helper.

If code uses `withdrawToChain(...)`, treat that as an application/helper extension layered on top of O2 account actions and fast-bridge contracts. The stock SDK method list does not include `withdrawToChain(...)` in `@o2exchange/sdk@0.1.0`.

## Market Data and Streams

```ts
const depth = await client.getDepth(market, 10);
const trades = await client.getTrades(market, 50);
const ticker = await client.getTicker(market);
const orders = await client.getOrders(tradeAccountId, market, true, 50);
const order = await client.getOrder(market, orderId);
```

Streaming:

```ts
for await (const update of await client.streamOrders(tradeAccountId)) {
  console.log(update.orders);
}
```

## Sharp Rules

- Use `trade_account_id` for O2 funds.
- Use owner signer for account setup, session setup, withdrawals, and account actions.
- Use session key only for trading actions.
- Always create/refresh session before placing orders.
- Always settle before placing balance-sensitive orders.
- Use market metadata for decimals and precision.
- Do not use bridge amount rules for order quantities; O2 market decimals come from market metadata.
- Do not call `withdrawToChain(...)` on a stock `O2Client` unless the project added that extension.
