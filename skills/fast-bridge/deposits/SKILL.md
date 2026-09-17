---
name: o2-fast-bridge-deposits
description: Deposit or bridge ETH, USDC, or other supported assets from Base, Ethereum, or another supported EVM chain to a Fuel wallet or O2 trading account through the O2 Fast Bridge proxy. Covers TypeScript, Python, and Rust SDK discovery, prepare/inspect/sign/submit/status, Fuel-unit amounts, recipient types, ERC-20 allowance and EIP-2612 permits, and advanced direct-Messenger bypasses.
---

# O2 Fast Bridge Deposits

Use the O2 SDK's `FastBridgeClient` as the default integration. It talks to the stateless Cloudflare Worker proxy, returns an unsigned EVM transaction plus a short-lived `preparationProof`, and exposes offline parsers so callers can inspect the transaction before signing it.

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

- [TypeScript deposit flow](references/typescript-deposit-flow.md)
- [Python deposit flow](references/python-deposit-flow.md)
- [Rust deposit flow](references/rust-deposit-flow.md)

## Flow

1. Create `FastBridgeClient` with an independently chosen mainnet or testnet URL.
2. Read `getInfo`, `getAssets`, and `getDepositInfo` for discovery and availability. Check `routeEnabled`, `paused`, `whitelisted`, `remainingCapacity`, `amountEligible`/`ineligibilityReason`, `requiresAllowance`, `allowanceSpender`, and `permitSupported` as applicable.
3. Call `prepareDeposit` with the intended route and full Fuel AssetId.
4. Parse and inspect the returned `unsignedTransaction` locally.
5. Optionally parse the proof claims to display expiry and signer information. Parsed claims are not authenticated.
6. Recheck proof expiry, then sign the locally computed EVM signing digest.
7. Submit the exact prepared transaction bytes, exact proof, and separate signature.
8. Poll `getDepositStatus`; a 404 means not found, not fabricated pending.

Never treat discovery responses as a trust anchor. Compare the parsed transaction with the original request and independently trusted chain IDs, Messenger addresses, token addresses, recipient, amount, value, and fee limits.

Pin chain IDs, Messenger addresses, and token addresses in application-owned network/deployment configuration. Populate that configuration from a reviewed Fast Bridge deployment record or values independently verified before pinning. The [O2 Network Identifiers](https://docs.o2.app/network-identifiers.html) page is O2 API-backed live data that is useful for discovery and cross-checking, but it and the proxy's `getInfo`/`getAssets` responses must not replace trusted pinned configuration when approving a transaction.

## Prepare Request

The wire request is:

```text
sourceChainId  supported EVM source chain ID
from           20-byte EVM sender address
to             32-byte Fuel address or contract ID
toType         "address" or "contract"
assetId        full 32-byte Fuel AssetId
amount         integer string in Fuel asset base units
permit         optional EIP-2612 permit object
```

`assetId` is the full Fuel AssetId, not an EVM token address and not the asset sub-ID. The asset sub-ID is the bridge-level identifier used to derive the Fuel AssetId for a specific Fuel contract; callers normally use the full AssetId returned by `getAssets`.

All API amounts use the Fuel asset's decimals. The proxy converts the deposit amount into the source token's base units when it constructs the EVM transaction. Do not pass a float or pre-convert into EVM token units.

Recipient rules:

- Use `toType: "address"` for a Fuel wallet/address.
- Use `toType: "contract"` for a Fuel contract ID.
- To fund an O2 trading account directly, pass its `trade_account_id` as `to` with `toType: "contract"`. To fund the owner wallet, pass its Fuel address with `toType: "address"`.
- Do not infer the type from a 32-byte value; both encodings have the same length.

The selected route determines whether the transaction calls `depositETH`, `deposit`, or `depositWithPermit` on the source-chain Messenger.

## Allowance And Permit

ERC-20 deposits need either:

- an existing allowance for the configured Messenger, followed by `deposit`; or
- an EIP-2612 permit supplied to prepare, followed by `depositWithPermit`.

The permit is a separate token-approval signature. It is not the EVM transaction signature. Build it from trusted token domain data, the current token nonce, the Messenger spender, value, and deadline. Permit creation and ordinary `approve` transactions happen outside the proxy API.

Native ETH routes use `depositETH` and put the deposited value in the EVM transaction's `value` field.

## Inspection And Signing

Parse the prepared EIP-1559 transaction with the SDK before signing:

```text
TypeScript: parseEvmUnsignedTransaction(...)
Python:     parse_evm_unsigned_transaction(...)
Rust:       parse_evm_unsigned_transaction(...)
```

The inspection includes the locally derived `signingDigest`, chain ID, nonce, Messenger, method, recipient/type, token, amount, value, gas limit, fee caps, calldata, permit data, and `estimatedNetworkFee`.

Important distinctions:

- The API request amount uses Fuel base units.
- The parsed EVM amount/value uses source-token units or wei.
- `estimatedNetworkFee` is `gasLimit * maxFeePerGas`, a maximum execution gas budget. It is not the actual fee and may exclude rollup L1 data fees.
- Sign the raw `signingDigest`; do not use personal-sign or another helper that prefixes or hashes it again.

With ethers, callers may instead parse the exact unsigned transaction, inspect it, use `wallet.signTransaction(...)`, and extract only the serialized signature from the signed transaction. Submit does not accept a signed transaction envelope.

## Preparation Proof And Submit

`preparationProof` is an opaque, short-lived proof produced by the Worker. It binds the operation and exact unsigned transaction bytes with an HMAC known only to the Workers.

SDK proof parsers expose these unauthenticated claims for display and preflight checks:

```text
version
keyId
expiresAt
signer
```

Parsing does not verify authenticity. Forged or expired proofs can still parse. There is deliberately no client-side verification helper because clients do not have the Worker secret; the proxy authenticates the proof during submit.

Submit exactly:

```text
unsignedTransaction  unchanged value returned by prepare
preparationProof      unchanged value returned by prepare
signature             separate 65-byte EVM r || s || v signature
```

Do not reconstruct, normalize, or replace the prepared transaction. The proof is bound to the original raw bytes.

The proxy remains stateless: it verifies the proof, expiry, transaction binding, route, sender, and signature without storing prepare-call state.

## Status And Recovery

`submitDeposit` reports acceptance, not final delivery. Poll `getDepositStatus(sourceChainId, evmTxHash)` with bounded backoff.

- A 404 means the transaction is not currently found.
- The proxy's source-side terminal states are `confirmed` and `reverted`; poll only while `source.status` is `pending` or the transaction is not yet found.
- The proxy does not currently correlate Fuel-side delivery, so `fuel.status` remains `unavailable` and will not become a terminal delivery result through continued polling.
- To confirm final delivery after source confirmation and the expected relay delay, query the intended Fuel wallet or contract balance directly using a trusted Fuel provider.
- The clients do not automatically retry submissions or follow redirects.
- A transport timeout during submit is ambiguous; reconcile chain/status data before resubmitting.

## Advanced: Bypassing The Proxy

Direct Messenger integration is an advanced protocol-level path, not the normal SDK workflow. It requires the application to maintain chain deployments, token mappings and decimals, allowance/permit logic, gas policy, transaction construction, receipt handling, and relay tracking itself.

The bundled [Messenger ABI](abis/Messenger.json) is for that explicit bypass use case; do not use it to reimplement the default Fast Bridge flow when `FastBridgeClient` is available. The [ERC-20 metadata ABI](abis/IERC20Metadata.json) and [ERC-20 permit ABI](abis/IERC20Permit.json) may also be used with the default proxy flow to read token metadata/nonces and construct the separate allowance or permit that occurs outside the proxy API.

## Sharp Rules

- Use the official proxy URL selected by the application; discovery is not a trust anchor.
- Pass the full Fuel AssetId and a Fuel-base-unit integer string.
- Keep `toType` explicit.
- Inspect the locally parsed transaction before signing.
- Preserve the exact unsigned bytes and proof through submit.
- Treat parsed proof claims as unauthenticated.
- Do not expose or request the Worker HMAC secret.
- Do not confuse a permit signature with the transaction signature.
- Do not treat submit acceptance as final cross-chain delivery.
