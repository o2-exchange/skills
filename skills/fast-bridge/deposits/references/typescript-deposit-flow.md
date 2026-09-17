# TypeScript Fast Bridge Deposit

Use `FastBridgeClient` from `@o2exchange/sdk` 0.4 or newer. The proxy URL is independent of the O2 trading-network configuration.

```ts
import {
  FAST_BRIDGE_TESTNET_URL,
  FastBridgeClient,
  parseEvmUnsignedTransaction,
  parsePreparationProof,
  type bridge,
} from "@o2exchange/sdk";
import { bytesToHex, fuelCompactSign, hexToBytes } from "@o2exchange/sdk/internals";

const client = new FastBridgeClient({
  baseUrl: FAST_BRIDGE_TESTNET_URL,
  timeoutMs: 30_000,
});

const request: bridge.DepositPrepareRequest = {
  sourceChainId: 11155111,
  from: evmAddress,
  to: fuelRecipient,
  toType: "address", // "contract" for a Fuel contract or O2 trade_account_id
  assetId: fullFuelAssetId,
  amount: "1000000", // integer Fuel asset base units
};

async function deposit(
  request: bridge.DepositPrepareRequest,
  privateKey: Uint8Array,
  approve: (
    request: bridge.DepositPrepareRequest,
    tx: bridge.EvmDepositInspection,
  ) => boolean | Promise<boolean>,
) {
  const prepared = await client.prepareDeposit(request);
  const claims = parsePreparationProof(prepared.preparationProof);
  const inspected = parseEvmUnsignedTransaction(prepared.unsignedTransaction);
  console.dir(inspected, { depth: null });

  // Compare chain ID, nonce, Messenger, method, recipient/type, token,
  // amount/value, gas, and fee caps with request and trusted configuration.
  if (!(await approve(request, inspected))) {
    throw new Error("Prepared deposit does not match intent");
  }

  // Recheck immediately before signing/submission in case approval took time.
  if (claims.expiresAt <= Date.now() / 1000) throw new Error("Prepare again");

  // SDK-native path: sign the locally computed raw digest and expand compact
  // r || yParityAndS into the EVM r || s || v form expected by submit.
  const compact = fuelCompactSign(privateKey, hexToBytes(inspected.signingDigest));
  const signature = new Uint8Array(65);
  signature.set(compact);
  signature[64] = 27 + (compact[32] >>> 7);
  signature[32] &= 0x7f;

  return client.submitDeposit({
    unsignedTransaction: prepared.unsignedTransaction,
    preparationProof: prepared.preparationProof,
    signature: bytesToHex(signature),
  });
}
```

Parsing is not approval. Trusted Messenger and token addresses should come from application-owned deployment configuration populated from a separately verified source, not from the proxy alone.

If the application already uses ethers, `npm install ethers` provides an alternative inside the same `deposit` function in place of the SDK-native signing block:

```ts
import { Transaction } from "ethers";

const transaction = Transaction.from(prepared.unsignedTransaction);
// Inspect transaction and the SDK parser result before signing.
const signed = await evmWallet.signTransaction(transaction);
const signature = Transaction.from(signed).signature?.serialized;
if (!signature) throw new Error("Missing EVM signature");
```

Submit `prepared.unsignedTransaction`, not `signed`. Always derive the signing digest from the parsed unsigned bytes rather than trusting a digest supplied separately by a service.

## Status

Use `client.getDepositStatus(submitted.sourceChainId, submitted.evmTxHash)` with bounded backoff. A `BridgeApiError` with status 404 means not found. Stop proxy polling when `source.status` becomes `confirmed` or `reverted`; the proxy's `fuel.status` remains `unavailable` today. Confirm final Fuel delivery separately by querying the recipient wallet or contract balance after the expected relay delay. A submit timeout is ambiguous, so reconcile before resubmitting.

## Native ETH And ERC-20

The same request shape covers native ETH and ERC-20 routes. The proxy selects `depositETH`, `deposit`, or `depositWithPermit` from its route configuration.

For ERC-20, establish an allowance for the trusted Messenger before prepare, or pass an EIP-2612 `permit` object. A permit is a separately signed token approval, not the transaction signature.

`parsePreparationProof` exposes `version`, `keyId`, `expiresAt`, and `signer` for display only. Only the proxy authenticates the proof and its exact transaction binding.
