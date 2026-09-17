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

// Obtain these from a Fuel provider the application already trusts, not from
// the proxy. maxInputs is not encoded in the transaction.
// trustedFuelProvider is the application's configured fuels Provider for its
// independently selected Fuel RPC endpoint.
const { consensusParameters } = await trustedFuelProvider.getChain();
const trustedFuelChainId: bigint = BigInt(consensusParameters.chainId.toString());
const trustedMaxInputs: number = consensusParameters.txParameters.maxInputs.toNumber();

async function withdraw(
  request: bridge.WithdrawPrepareRequest,
  privateKey: Uint8Array,
  approve: (
    request: bridge.WithdrawPrepareRequest,
    tx: bridge.FuelWithdrawalInspection,
  ) => boolean | Promise<boolean>,
) {
  const prepared = await client.prepareWithdraw(request);
  if (BigInt(prepared.fuelChainId) !== trustedFuelChainId) {
    throw new Error("Unexpected Fuel chain");
  }

  const claims = parsePreparationProof(prepared.preparationProof);
  const inspected = parseFuelUnsignedTransaction(
    prepared.unsignedTransaction,
    trustedFuelChainId,
    trustedMaxInputs,
  );
  console.dir(inspected, { depth: null });

  // Check contract, destination chain/recipient, asset, gross/fee/net amounts,
  // fee caps, block expiry, every input owner/asset, and all outputs against
  // request and trusted deployment configuration.
  if (!(await approve(request, inspected))) {
    throw new Error("Prepared withdrawal does not match intent");
  }

  // Recheck proof expiry immediately before signing/submission. Also ensure the
  // parsed expirationBlockHeight remains usable for the intended submission.
  if (claims.expiresAt <= Date.now() / 1000) throw new Error("Prepare again");

  const signature = bytesToHex(
    fuelCompactSign(privateKey, hexToBytes(inspected.transactionId)),
  );
  const submitted = await client.submitWithdraw({
    unsignedTransaction: prepared.unsignedTransaction,
    preparationProof: prepared.preparationProof,
    signature,
  });
  return { transactionId: inspected.transactionId, submitted };
}
```

Sign the locally computed raw `transactionId` without personal-sign or extra hashing. Submit only the exact prepared bytes, exact proof, and compact signature; do not spread `fuelChainId` or reconstructed fields into submit. A wallet or external signer that supports raw-digest signing may replace direct private-key handling.

## Status

Use `client.getWithdrawStatus(transactionId)` with bounded backoff. A `BridgeApiError` with status 404 means not found. Stop when `fuel.status` becomes `success` or `reverted`; the proxy's destination status remains `unavailable` today. Confirm EVM delivery separately by checking the recipient balance or relevant trusted Outpost event after the expected relay delay. A submit timeout is ambiguous, so query status before resubmitting.

## Amounts And Funding

The request amount is gross. Inspect `grossAmount`, `bridgeFee`, and `netAmount`; the expected relationship is `netAmount = grossAmount - bridgeFee`. `networkFee.maxFee` is a separate Fuel base-asset cap.

If funds are in an O2 trading account, first use the normal `O2Client.withdraw(...)` to move them to the owner Fuel wallet, then run this flow after the coin is available.

Parsed proof claims are display-only; only the proxy authenticates the proof and exact transaction binding.
