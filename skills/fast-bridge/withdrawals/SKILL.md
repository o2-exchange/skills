---
name: o2-fast-bridge-withdrawals
description: Withdraw or bridge supported assets from a funded Fuel wallet to a Base, Ethereum, or other supported EVM address through the O2 Fast Bridge proxy. Covers TypeScript, Python, and Rust SDK discovery, fees, prepare/inspect/sign/submit/status, Fuel transaction parsing, gross/net amounts, trusted chain parameters, O2-account funding, and advanced direct-contract bypasses.
---

# O2 Fast Bridge Withdrawals

Use the O2 SDK's `FastBridgeClient` as the default integration. The stateless Cloudflare Worker constructs a Fuel transaction, returns its exact unsigned bytes plus a short-lived `preparationProof`, and accepts the unchanged tuple with a separate Fuel signature.

Official proxy roots:

```text
Mainnet: https://bridge.o2.app
Testnet: https://bridge.testnet.o2.app
```

Do not append `/v1`; the client adds endpoint paths.

Current SDK baselines:

```text
TypeScript: @o2exchange/sdk 0.4+
Python:     o2-sdk 0.5+
Rust:       o2-sdk 0.4+
```

Read the language flow that matches the implementation:

- [TypeScript withdrawal flow](references/typescript-withdrawal-flow.md)
- [Python withdrawal flow](references/python-withdrawal-flow.md)
- [Rust withdrawal flow](references/rust-withdrawal-flow.md)

## Wallet Boundary

Fast Bridge withdrawal spends coins owned by a Fuel wallet address. It does not directly spend an O2 trading account and does not use an O2 trading session.

If the funds are still in an O2 trading account:

1. Use the normal `O2Client.withdraw(...)` flow to move the asset to the owner Fuel wallet.
2. Wait until that Fuel-wallet coin is available.
3. Use `FastBridgeClient` to bridge from the funded Fuel wallet to EVM.

The owner/session/trading-account model matters for step 1 only. The Fast Bridge transaction in step 3 is signed by the Fuel wallet that owns its coin inputs.

## Flow

1. Create `FastBridgeClient` with an independently chosen mainnet or testnet URL.
2. Read `getInfo`, `getAssets`, `getWithdrawInfo`, and `getWithdrawFee` for discovery and availability. Check `routeEnabled`, `paused`, `withdrawEnabled`, `amountEligible`/`ineligibilityReason`, fee freshness, and `rateLimit.remainingToday` as applicable.
3. Obtain the trusted Fuel chain ID and consensus `maxInputs` independently of the proxy.
4. Call `prepareWithdraw` for a funded Fuel wallet.
5. Parse the exact transaction locally using the trusted chain ID and `maxInputs`.
6. Compare every relevant parsed field with the original intent and trusted configuration.
7. Recheck proof expiry, then sign the locally computed Fuel transaction ID.
8. Submit the exact prepared bytes, exact proof, and separate compact signature.
9. Poll `getWithdrawStatus`; a 404 means not found, not fabricated pending.

Do not use prepare as a balance/status poll. Prepare may perform RPC work, coin selection, construction, and simulation.

## Obtaining Trusted Fuel Parameters

Get the Fuel chain ID and consensus `txParameters.maxInputs` from a Fuel RPC provider that the application already trusts, or pin reviewed values in the application's own network configuration. Do not source them from the proxy response or discovery endpoints.

With fuels-ts 0.103, a trusted provider exposes both values:

```ts
const { consensusParameters } = await trustedFuelProvider.getChain();
const trustedFuelChainId = BigInt(consensusParameters.chainId.toString());
const trustedMaxInputs = consensusParameters.txParameters.maxInputs.toNumber();
```

Other languages can query the same trusted Fuel GraphQL `chain.consensusParameters` data or consume pinned application configuration. Cross-check `prepared.fuelChainId` against the trusted chain ID before parsing or signing. Pin the expected Asset Registry and Gas Oracle contract IDs in the same trusted deployment configuration; discovery responses may be compared against those values but are not substitutes for them.

## Prepare Request

The wire request is:

```text
destinationChainId  supported EVM destination chain ID
from                 32-byte funded Fuel wallet address
to                   20-byte EVM recipient address
assetId              full 32-byte Fuel AssetId
amount               gross integer amount in Fuel asset base units
```

`assetId` is the full Fuel AssetId, not the asset sub-ID. The asset sub-ID is the bridge-level identifier from which an Asset Registry contract derives an AssetId; callers normally use the full ID returned by `getAssets`.

`amount` is gross. The prepared transaction embeds the Fast Bridge fee:

```text
netAmount = grossAmount - bridgeFee
```

The quoted relationship is expected but must still be inspected. The oracle fee may move within the protocol's allowed tolerance before execution.

## Fee And Eligibility Reads

Use:

- `getWithdrawInfo(destinationChainId, assetId?, amount?)` for route, contracts, pause/rate-limit state, fee freshness, recipient requirements, and optional amount eligibility.
- `getWithdrawFee(destinationChainId, assetId)` for the current asset-denominated bridge fee and observation metadata.

The bridge fee is separate from Fuel network fees. `networkFee.maxFee` is a Fuel transaction fee cap in Fuel's base asset; it is not necessarily denominated in the asset being withdrawn.

## Inspection And Signing

Parse the prepared transaction with:

```text
TypeScript: parseFuelUnsignedTransaction(unsigned, trustedChainId, trustedMaxInputs)
Python:     parse_fuel_unsigned_transaction(unsigned, trusted_chain_id, trusted_max_inputs)
Rust:       parse_fuel_unsigned_transaction(unsigned, trusted_chain_id, trusted_max_inputs)
```

`maxInputs` is not encoded in the transaction but affects FuelVM absolute pointers. Obtain it from trusted consensus configuration, not the proxy response.

The parser exposes, among other fields:

```text
transactionId                 locally computed signing digest
assetId and assetSubId        full Fuel ID and bridge asset sub-ID
assetRegistryContractId       called Fuel contract
destinationChainId, recipient intended EVM route
grossAmount, bridgeFee, netAmount
networkFee.maxFee
expirationBlockHeight
scriptGasLimit and policies
inputs and outputs
```

Before signing, verify the transaction against the request and independently trusted configuration. In particular, check chain ID, Asset Registry contract, Gas Oracle/route information, asset, recipient, gross/fee/net amounts, fee caps, expiry, every input owner/asset, and all outputs.

Fuel `Change` and `Variable` output amounts, and a `Variable` output's recipient/asset, are execution results excluded from the signing ID. They are not signed guarantees. The parser performs no RPC calls and does not certify economic safety.

Sign the raw 32-byte `transactionId` with compact secp256k1. Do not personal-sign or otherwise hash it again.

Applications may use a wallet, KMS, HSM, or other external signer instead of exposing a raw private key, provided it signs this exact raw digest and returns the compact Fuel signature expected by submit.

## Preparation Proof And Submit

`preparationProof` is an opaque, short-lived Worker HMAC binding the operation and exact unsigned transaction bytes.

Proof parsers expose unauthenticated display claims:

```text
version
keyId
expiresAt
signer
```

Parsing is not verification. Only the proxy has the HMAC secret and authenticates the proof during submit.

Submit exactly:

```text
unsignedTransaction  unchanged value returned by prepare
preparationProof      unchanged value returned by prepare
signature             separate compact Fuel signature
```

Do not reconstruct the transaction or spread other prepare-response fields into submit. Proof expiry and Fuel block-height expiry are independent; check both.

## Status And Recovery

Save the locally computed transaction ID before submit. Poll `getWithdrawStatus(transactionId)` with bounded backoff.

- A 404 means not currently found.
- The proxy's Fuel-side terminal states are `success` and `reverted`; poll only while `fuel.status` is `pending` or the transaction is not yet found.
- The proxy does not currently correlate destination-chain delivery, so `destination.status` remains `unavailable` and will not become a terminal delivery result through continued polling.
- To confirm final delivery after Fuel success and the expected relay delay, query the EVM recipient balance or the relevant trusted Outpost event directly.
- The clients do not automatically retry submissions or follow redirects.
- A submit timeout can mean the transaction was accepted; query status before resubmitting.

## Advanced: Bypassing The Proxy

Direct calls to the Fuel Asset Registry and Gas Oracle are an advanced protocol-level bypass. They require the application to track deployments, ABI compatibility, oracle freshness/tolerance, coin selection, Fuel transaction layout and consensus parameters, output semantics, simulation, signing, submission, and relay state.

The bundled [Asset Registry ABI](abis/AssetRegistry-abi.json) and [Gas Oracle ABI](abis/GasOracle-abi.json) exist for that explicit bypass use case. Do not reproduce the old `withdrawToChain` account-action flow as the default Fast Bridge integration; the proxy withdraws from a funded Fuel wallet.

## Sharp Rules

- Fund a Fuel wallet first; do not pass an O2 `trade_account_id` as `from`.
- Pass the full Fuel AssetId and a gross Fuel-base-unit integer string.
- Trust chain ID and consensus `maxInputs` from independent configuration.
- Parse and inspect before signing.
- Sign the locally computed transaction ID without extra hashing.
- Preserve exact unsigned bytes and proof through submit.
- Treat proof claims as unauthenticated.
- Distinguish bridge fee, network fee cap, and delivered net amount.
- Do not treat submit acceptance as final EVM delivery.
