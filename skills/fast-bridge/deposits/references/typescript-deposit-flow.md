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

const client = new FastBridgeClient({
  baseUrl: FAST_BRIDGE_TESTNET_URL,
  timeoutMs: 30_000,
});

const request: bridge.DepositPrepareRequest = {
  sourceChainId: 11155111,
  from: evmWallet.address,
  to: fuelRecipient,
  toType: "address", // use "contract" for a Fuel contract recipient
  assetId: fullFuelAssetId,
  amount: "1000000", // integer Fuel asset base units
};

const prepared = await client.prepareDeposit(request);
const claims = parsePreparationProof(prepared.preparationProof);
if (claims.expiresAt <= Date.now() / 1000) throw new Error("Prepare again");

const inspected = parseEvmUnsignedTransaction(prepared.unsignedTransaction);
console.dir(inspected, { depth: null });

// Application approval must compare chainId, nonce, Messenger, method,
// recipient/type, token, amount/value, gas, and fee caps against request and
// independently trusted configuration. Parsing itself is not approval.
if (!approveDeposit(request, inspected)) {
  throw new Error("Prepared deposit does not match intent");
}

// ethers parses the same exact unsigned EIP-1559 envelope. signTransaction signs
// its locally derived digest. Submit only the resulting r || s || v signature.
```

Extract the signature from the signed transaction with ethers' `Transaction` class:

```ts
import { Transaction } from "ethers";

const signed = await evmWallet.signTransaction(
  Transaction.from(prepared.unsignedTransaction),
);
const signature = Transaction.from(signed).signature?.serialized;
if (!signature) throw new Error("Missing EVM signature");

const submitted = await client.submitDeposit({
  unsignedTransaction: prepared.unsignedTransaction,
  preparationProof: prepared.preparationProof,
  signature,
});

console.log(submitted.evmTxHash);
```

Do not submit `signed`; submit the exact unsigned transaction returned by prepare plus its separate signature. Do not compute approval from `signingPayload` supplied by the service; the SDK parser computes `inspected.signingDigest` from the unsigned bytes.

Poll status with bounded backoff:

```ts
import { BridgeApiError } from "@o2exchange/sdk";

try {
  console.dir(
    await client.getDepositStatus(submitted.sourceChainId, submitted.evmTxHash),
    { depth: null },
  );
} catch (error) {
  if (!(error instanceof BridgeApiError) || error.status !== 404) throw error;
  console.log("Not found yet; this is not a fabricated pending state");
}
```

## Native ETH And ERC-20

The same request shape covers native ETH and ERC-20 routes. The proxy selects `depositETH`, `deposit`, or `depositWithPermit` from its route configuration.

For ERC-20, establish an allowance for the trusted Messenger before prepare, or pass an EIP-2612 `permit` object. A permit is a separately signed token approval, not the transaction signature.

## Proof Claims

`parsePreparationProof` exposes `version`, `keyId`, `expiresAt`, and `signer`, but those claims are unauthenticated. Only the proxy verifies the HMAC binding to this operation and the exact transaction bytes.
