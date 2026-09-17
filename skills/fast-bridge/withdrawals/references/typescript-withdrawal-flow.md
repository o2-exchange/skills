# TypeScript Fast Bridge Withdrawal

Use `FastBridgeClient` from `@o2exchange/sdk` 0.4 or newer. This flow spends a funded Fuel wallet, not an O2 trading account or session.

```ts
import {
  FAST_BRIDGE_TESTNET_URL,
  FastBridgeClient,
  parseFuelUnsignedTransaction,
  parsePreparationProof,
  type bridge,
} from "@o2exchange/sdk";
import { bytesToHex, fuelCompactSign, hexToBytes } from "@o2exchange/sdk/internals";

const client = new FastBridgeClient({
  baseUrl: FAST_BRIDGE_TESTNET_URL,
  timeoutMs: 30_000,
});

const request: bridge.WithdrawPrepareRequest = {
  destinationChainId: 11155111,
  from: fuelWalletAddress,
  to: evmRecipient,
  assetId: fullFuelAssetId,
  amount: "1000000", // gross integer amount in Fuel asset base units
};

const prepared = await client.prepareWithdraw(request);
const claims = parsePreparationProof(prepared.preparationProof);
if (claims.expiresAt <= Date.now() / 1000) throw new Error("Prepare again");

// These must come from independently trusted Fuel configuration. maxInputs is
// not encoded in the transaction and affects FuelVM absolute pointers.
if (BigInt(prepared.fuelChainId) !== trustedFuelChainId) {
  throw new Error("Unexpected Fuel chain");
}
const inspected = parseFuelUnsignedTransaction(
  prepared.unsignedTransaction,
  trustedFuelChainId,
  trustedMaxInputs,
);
console.dir(inspected, { depth: null });

// Check contract, destination chain/recipient, asset, gross/fee/net amounts,
// fee caps, expiry, every input owner/asset, and all outputs against request and
// trusted configuration. Parsing is not approval.
if (!approveWithdrawal(request, inspected)) {
  throw new Error("Prepared withdrawal does not match intent");
}

const signature = bytesToHex(
  fuelCompactSign(privateKey, hexToBytes(inspected.transactionId)),
);

const submitted = await client.submitWithdraw({
  unsignedTransaction: prepared.unsignedTransaction,
  preparationProof: prepared.preparationProof,
  signature,
});
```

Sign the locally computed raw `transactionId` without personal-sign or extra hashing. Submit only the exact prepared bytes, exact proof, and compact signature; do not spread `fuelChainId` or reconstructed fields into submit.

## Status

Save `inspected.transactionId` before submit and use:

```ts
import { BridgeApiError } from "@o2exchange/sdk";

try {
  console.dir(await client.getWithdrawStatus(inspected.transactionId), {
    depth: null,
  });
} catch (error) {
  if (!(error instanceof BridgeApiError) || error.status !== 404) throw error;
  console.log("Not found yet; this is not a fabricated pending state");
}
```

A submit timeout is ambiguous; query status before resubmitting. Fuel inclusion and EVM delivery are separate states.

## Amounts And Funding

The request amount is gross. Inspect `grossAmount`, `bridgeFee`, and `netAmount`; the expected relationship is `netAmount = grossAmount - bridgeFee`. `networkFee.maxFee` is a separate Fuel base-asset cap.

If funds are in an O2 trading account, first use the normal `O2Client.withdraw(...)` to move them to the owner Fuel wallet, then run this flow after the coin is available.

Parsed proof claims (`version`, `keyId`, `expiresAt`, `signer`) are unauthenticated. Only the proxy verifies the HMAC binding to this operation and exact transaction bytes.
