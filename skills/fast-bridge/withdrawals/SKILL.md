---
name: o2-fast-bridge-withdrawals
description: Explains O2 fast bridge withdrawals from O2/Fuel to EVM chains. Use for withdrawToChain-style helpers, setupAccount, createSession, owner signer vs session key, trade_account_id, WithdrawViaFastBridgeWithFee account actions, AssetRegistry.withdraw_via_fast_bridge_with_fee, gas-oracle fees, 9-decimal withdrawal amounts, unwrap vs bridge, and EVM recipient padding.
---

# O2 Fast Bridge Withdrawals

Use this skill when a trader wants to move funds out of O2/Fuel to an EVM chain.

For O2 SDK setup, sessions, markets, orders, nonces, and signer rules, use:

- `../../o2-sdk/typescript/SKILL.md`
- `../../o2-sdk/python/SKILL.md`
- `../../o2-sdk/rust/SKILL.md`

This skill covers the fast-bridge withdrawal parts.

Current important limitation: stock `@o2exchange/sdk@0.1.0` does not natively support fast-bridge withdrawals to EVM chains. Until native SDK support ships, agents should replicate the `withdrawToChain`-style helper flow described here by combining O2 SDK account/session primitives with the fast-bridge Fuel contract call and account-action API.

Use these bundled ABIs when writing bridge code:

```text
./abis/AssetRegistry-abi.json
./abis/GasOracle-abi.json
```

These are full Fuel ABIs so `fuels.Interface`, generated bindings, or equivalent ABI tooling can derive `selectorBytes` and encode arguments directly.

## Install

Use the minimal dependency set for the implementation style you are writing.

TypeScript / JavaScript helper flow:

```bash
npm install @o2exchange/sdk @o2exchange/contracts ethers fuels
```

Python helper flow:

```bash
pip install o2-sdk web3 python-dotenv
```

Python 3.10+ required.

Rust helper flow:

```bash
cargo add anyhow ethers fuels o2-sdk rand reqwest serde serde_json sha2 tokio
```

## Practical Notes

Treat the code in this skill as protocol guidance, not as guaranteed copy-paste code. Verify the current O2 SDK / contracts package version, `fuels-ts` / `fuels-rs` version, and live Fuel node behavior before assuming an example will run unchanged.

Important implementation rules:

- Withdrawal input amount is the desired net EVM-side amount in 9-decimal wrapped units unless the caller explicitly says otherwise.
- The actual trading-account debit is `grossDebit = desiredNetAmount + feeQuote`.
- Fast-bridge withdrawal uses the universal wrapped symbol such as `uwUSDC`, not the market label alone.
- The O2 API JSON uses a normal EVM address string. The AssetRegistry calldata uses that same address left-padded to 32 bytes.
- Python implementations should require `forc` CLI for the fee-quote step.
- Reason: this flow does not currently have a practical Python Fuel deployed-contract client equivalent to the JS/Rust `GasOracle.get_withdrawal_fee(...)` path.
- If using an EVM owner, make sure the submitted `Secp256k1` signature is the 64-byte compact Fuel form, not a raw 65-byte Ethereum `personal_sign` output.
- In Rust, a GasOracle fee quote may require `Execution::state_read_only()` and explicit dependency contract IDs instead of a default `.call()`.
- Do not feed shortened display addresses such as `0xb66c…d0d7` back into calldata or API payloads. Always use full `0x...` hex strings.

For compact language-specific walkthroughs, see:

- `references/python-withdrawal-flow.md`
- `references/typescript-withdrawal-flow.md`
- `references/rust-withdrawal-flow.md`
- `../deposits/references/python-deposit-flow.md`
- `../deposits/references/typescript-deposit-flow.md`
- `../deposits/references/rust-deposit-flow.md`

If an implementation does not want to install the full Fuel ABI encoding stack, this skill provides the pieces needed to build the signed withdrawal payload manually:

- precomputed Fuel `selectorBytes` values for the signed withdrawal path
- exact owner signing byte layout
- exact `WithdrawViaFastBridgeWithFee` O2 API payload shape
- exact `withdraw_via_fast_bridge_with_fee` call-data byte layout
- exact wrapped asset ID and asset sub ID derivation
- EVM-vs-Fuel owner signing rules

## Choose the Path

Most O2 users should use the O2 trading-account path.

```text
Funds in O2 trading account -> use O2 SDK / account action
Funds in plain Fuel wallet  -> direct AssetRegistry call
Staying on Fuel             -> unwrap, not bridge withdrawal
```

Do not blur these paths:

- Bridge withdrawal: releases value on an EVM destination chain.
- Unwrap: stays on Fuel and converts wrapped asset to canonical Fuel asset.
- Trading-account withdrawal: spends from `trade_account_id`, signed by the owner.

## Mainnet Contracts

```text
Fuel mainnet
- FastBridge:           0x12a1cf2d5b5b4eb7ece675b9d84f450fd49dfb969b46687627841a81c4ffb91f
- AssetRegistry:        0x91cfcbef2caad02996cdcb5b897222170e85a91cd2db8f23f07ea7d9ca030c19
- WrappedAssetsMinter:  0x0f9f509374c2da68997a3a1ad6d85be3f351c2f5f3da4c65bdbfad9b0bb25504
- GasOracle:            0x3d20e5a675c5fa1053fba11e176099711ba2f23112e385a7f6e2e759eca84f94
```

## ABI Signatures

Fast bridge fee quote:

```sway
abi GasOracle {
    #[storage(read)]
    fn get_withdrawal_fee(chain_id: u32, asset_id: AssetId) -> u64;
}
```

Fast bridge withdrawal:

```sway
abi AssetRegistry {
    #[storage(read, write), payable]
    fn withdraw_via_fast_bridge_with_fee(
        sub_id: b256,
        destination_chain: u32,
        recipient: b256,
        fee_quote: u64,
    ) -> b256;
}
```

O2 trading-account simulation call:

```sway
abi TradeAccount {
    #[storage(read, write), payable]
    fn call_contracts(
        signature: Option<Signature>,
        calls: Vec<CallContractArg>,
    );
}

enum Signature {
    Secp256k1: [u8; 64],
    Secp256r1: [u8; 64],
    Ed25519: [u8; 64],
}

struct CallContractArg {
    contract_id: ContractId,
    function_selector: Bytes,
    call_params: CallParams,
    call_data: Option<Bytes>,
}

struct CallParams {
    coins: u64,
    asset_id: AssetId,
    gas: u64,
}
```

Minimal ABI fragments to copy into generated bindings or encoding tests:

```json
{
  "GasOracle.get_withdrawal_fee": {
    "inputs": [
      { "name": "chain_id", "type": "u32" },
      { "name": "asset_id", "type": "AssetId" }
    ],
    "output": "u64"
  },
  "AssetRegistry.withdraw_via_fast_bridge_with_fee": {
    "inputs": [
      { "name": "sub_id", "type": "b256" },
      { "name": "destination_chain", "type": "u32" },
      { "name": "recipient", "type": "b256" },
      { "name": "fee_quote", "type": "u64" }
    ],
    "output": "b256"
  },
  "TradeAccount.call_contracts": {
    "inputs": [
      { "name": "signature", "type": "Option<Signature>" },
      { "name": "calls", "type": "Vec<CallContractArg>" }
    ],
    "output": "()"
  }
}
```

Minimal TradeAccount nested types:

```json
{
  "Signature": {
    "Secp256k1": "[u8; 64]",
    "Secp256r1": "[u8; 64]",
    "Ed25519": "[u8; 64]"
  },
  "CallContractArg": {
    "contract_id": "ContractId",
    "function_selector": "Bytes",
    "call_params": "CallParams",
    "call_data": "Option<Bytes>"
  },
  "CallParams": {
    "coins": "u64",
    "asset_id": "AssetId",
    "gas": "u64"
  }
}
```

Encoding rules for this helper:

- In the O2 signed `call_contracts` path, `CallContractArg.function_selector` is Fuel `selectorBytes`: `u64_be(method_name.length) || utf8(method_name)`.
- Do not use the 8-byte Fuel method hash in `bridge_call.function_selector` for this O2 account-action flow.
- Do not use Keccak/Solidity selectors.
- `call_data` is ABI-encoded arguments for `withdraw_via_fast_bridge_with_fee(sub_id, destination_chain, recipient, fee_quote)`.
- `recipient` inside `call_data` is the 32-byte left-padded EVM recipient.
- `coins` is the gross debit amount in the wrapped asset.
- `asset_id` is the wrapped asset ID derived from `WrappedAssetsMinter` and `sha256(asset_symbol)`.
- `gas` must be `u64::MAX`.
- For simulation, pass `Some(Signature::Secp256k1(signature_bytes))`.
- For O2 API JSON, submit `{ "Secp256k1": "0x..." }`.

Precomputed Fuel `selectorBytes` for this flow:

```text
GasOracle.get_withdrawal_fee
selectorBytes: 0x00000000000000126765745f7769746864726177616c5f666565
raw Fuel method selector: 0x000000007364c71f

AssetRegistry.withdraw_via_fast_bridge_with_fee
selectorBytes: 0x000000000000002177697468647261775f7669615f666173745f6272696467655f776974685f666565

TradeAccount.call_contracts
selectorBytes: 0x000000000000000e63616c6c5f636f6e747261637473
```

Use the `AssetRegistry.withdraw_via_fast_bridge_with_fee` `selectorBytes` inside `bridge_call.function_selector`. O2 account-action signing uses the same byte encoding for `actionSelector("call_contracts")`; the precomputed `TradeAccount.call_contracts` value above is included so agents do not need to derive it.

SelectorBytes derivation:

```ts
function selectorBytes(name: string): Uint8Array {
  const bytes = new TextEncoder().encode(name);
  return concat([u64BE(bytes.length), bytes]);
}

const withdrawSelectorBytes = selectorBytes("withdraw_via_fast_bridge_with_fee");
```

This does not require the Fuel TypeScript or Rust SDK.

O2 account-action signing also uses this variable-length selector encoding:

```ts
function actionSelector(name: string): Uint8Array {
  const bytes = new TextEncoder().encode(name);
  return concat([u64BE(bytes.length), bytes]);
}
```

Use `actionSelector("call_contracts")` in `signing_bytes`. Use `selectorBytes("withdraw_via_fast_bridge_with_fee")` for the target contract call payload.

Why the target selector bytes are needed:

```text
O2 account action
  -> TradeAccount.call_contracts(...)
    -> low-level Fuel CALL to AssetRegistry
      -> withdraw_via_fast_bridge_with_fee(...)
```

The O2 REST payload does not use the selector bytes directly. They are needed inside `bridge_call.function_selector` because `TradeAccount.call_contracts` builds a low-level Fuel VM call and must tell `AssetRegistry` which method to run. If generated Fuel bindings are available, use the function fragment's `selectorBytes`. If not, use the precomputed `selectorBytes` above.

Owner signing bytes for the O2 account action:

```text
signing_bytes =
  u64_be(nonce)
  || u64_be(o2_chain_id)
  || actionSelector("call_contracts")
  || build_actions_signing_bytes(nonce, [bridge_call])[8..]

bridge_call =
  contract_id: AssetRegistry
  function_selector: 0x000000000000002177697468647261775f7669615f666173745f6272696467655f776974685f666565
  amount: grossDebit
  asset_id: wrappedAssetId
  gas: u64::MAX
  call_data: abi_encode(sub_id, destination_chain, padded_evm_recipient, fee_quote)
```

No-SDK byte layouts:

```text
assetSubId =
  sha256(utf8(asset_symbol))

wrappedAssetId =
  sha256(bytes32(WrappedAssetsMinter contract id) || bytes32(assetSubId))

withdraw_via_fast_bridge_with_fee call_data =
  bytes32(sub_id)
  || u32_be(destination_chain)
  || bytes32(padded_evm_recipient)
  || u64_be(fee_quote)

get_withdrawal_fee call_data =
  u32_be(destination_chain)
  || bytes32(wrappedAssetId)

build_actions_signing_bytes(nonce, [bridge_call]) =
  u64_be(nonce)
  || u64_be(num_calls)
  || for each call:
       bytes32(contract_id)
       || u64_be(function_selector.length)
       || function_selector
       || u64_be(amount)
       || bytes32(asset_id)
       || u64_be(gas)
       || option_call_data

option_call_data =
  None: u64_be(0)
  Some(bytes): u64_be(1) || u64_be(bytes.length) || bytes
```

For the bridge withdrawal call data, the length is `32 + 4 + 32 + 8 = 76` bytes. `destination_chain` is a 4-byte big-endian `u32`, not an 8-byte `u64`.

External calls needed by the helper:

```text
1. Fetch nonce
   GET https://api.o2.app/v1/accounts?trade_account_id=<trade_account_id>
   Read nonce defensively from response.nonce or response.trade_account.nonce.

2. Fetch O2 signing chain ID
   GET https://api.o2.app/v1/markets
   Read response.chain_id. It may be a hex string such as "0x0"; parse it to u64 for signing.

3. Fetch fast-bridge fee
   Preferred: use a Fuel contract library with ./abis/GasOracle-abi.json:
     GasOracle.get_withdrawal_fee(destination_chain, { bits: wrappedAssetId })

   Python path in this skill: use `forc call` for the fee quote:
     forc call <GasOracle> --mainnet --abi ./abis/GasOracle-abi.json get_withdrawal_fee <chain_id> '{<wrappedAssetId>}' --external-contracts <dependency_contract_id> --mode dry-run -o json
   Parse `result` as `feeQuote`.

   If building a true raw Fuel read transaction:
     contract_id: GasOracle
     function_selector: 0x000000007364c71f
     call_data: u32_be(destination_chain) || bytes32(wrappedAssetId)
   Decode returned u64 as feeQuote.

4. Sign owner payload
   EVM owner: Ethereum personal_sign over signing_bytes.
   Fuel owner: Fuel personal_sign over signing_bytes.
   Submit envelope is always { "Secp256k1": "0x<64-byte-signature>" }.

5. Submit O2 action
   POST https://api.o2.app/v1/accounts/actions
   Headers:
    Content-Type: application/json
    O2-Owner-Id: <owner_b256>
   Body: the WithdrawViaFastBridgeWithFee payload shown below.
```

For signing, `o2_chain_id` means the O2 market/config chain ID used by the O2 API and SDK. Do not substitute an arbitrary raw Fuel provider chain ID if it differs from the O2 client/config value.

## Identifier Rules

These identifiers are different. Do not swap them.

```text
Owner b256:       owner identity, often padded EVM address
Trade account ID: O2 Fuel contract account holding user funds
EVM recipient:    normal 20-byte destination address on Base/Ethereum/etc.
```

Deposits into an O2 trading account use `trade_account_id` as the bridge recipient with `recipientIsContract = true`. Withdrawals from an O2 trading account also use `trade_account_id`.

## Amount Rules

Fast-bridge withdrawal amounts use Fuel wrapped-asset decimals: 9 decimals.

- `"0.5"` means `500000000` wrapped units.
- Do not use destination USDC's 6 decimals for withdrawal input.
- Do not use destination ETH's 18 decimals for withdrawal input.
- Raw bigint must already be scaled to 9 decimals.
- The Fuel contract receives a gross amount and deducts the current gas-oracle fee from it.

Example:

```ts
import { parseUnits } from "ethers";

const withdrawAmount = parseUnits("0.5", 9); // 500000000
```

Recommended user-facing helper semantics:

```text
desiredNetAmount = amount the user wants to receive on the EVM side
feeQuote         = GasOracle.get_withdrawal_fee(destinationChainId, wrappedAssetId)
grossDebit       = desiredNetAmount + feeQuote
```

Use `grossDebit` as the forwarded/action amount. If an existing helper treats its `amount` argument as the gross debit, say that clearly because the user receives `amount - requiredFee` on the destination chain.

## Asset Symbol Rules

Fast bridge withdrawal sub IDs use the universal wrapped asset symbol registered in the AssetRegistry.

```text
USDC on EVM source/destination -> uwUSDC on Fuel/O2 fast bridge
ETH  on EVM source/destination -> uwETH on Fuel/O2 fast bridge
FUEL on EVM source/destination -> uwFUEL on Fuel/O2 fast bridge
```

Use:

```text
assetSubId = sha256(utf8("uwUSDC"))
wrappedAssetId = sha256(bytes32(WrappedAssetsMinter) || bytes32(assetSubId))
```

Do not hash `"USDC"` for a fast-bridge withdrawal unless that exact symbol is explicitly registered in the target AssetRegistry. O2 market labels may show `ETH/USDC`, but the fast-bridge withdrawal asset symbol is the universal wrapped symbol such as `uwUSDC`.

## O2 Helper Withdrawal Flow

This is the path to recommend when funds are in an O2 trading account.

Mental model:

1. Use the owner signer.
2. Make sure the O2 trading account exists.
3. Create or refresh the trading session only if the app also trades.
4. Settle balances into the trading account if the funds are still in open/fill state.
5. Call or implement a `withdrawToChain`-style helper with the `trade_account_id`.
6. Pass the destination EVM recipient as a plain EVM address string.

Session key reminder:

- Session key can place/cancel/settle orders.
- Session key cannot withdraw.
- Withdrawal is owner-signed.
- If the owner is an EVM EOA, use `O2Client.loadEvmWallet(privateKey)` so the owner `b256Address` is the padded EVM address and signatures use Ethereum `personal_sign`.
- If the owner is a Fuel EOA, use `O2Client.loadWallet(privateKey)` so the owner `b256Address` and signatures use Fuel signing.
- In both cases, call `ownerSigner.personalSign(signingBytes)` and submit the result as `Secp256k1`. The signer helper chooses the correct Fuel-vs-EVM digest. Do not manually hash or prefix first.

### Setup and Session Snippet

O2 setup:

```ts
import { Network, O2Client } from "@o2exchange/sdk";

const client = new O2Client({ network: Network.MAINNET });
const ownerSigner =
  ownerType === "evm"
    ? O2Client.loadEvmWallet(ownerPrivateKey)
    : O2Client.loadWallet(ownerPrivateKey);

// Creates the trade account if needed and returns its ID.
const account = await client.setupAccount(ownerSigner);
const tradeAccountId = account.tradeAccountId;

// Use the O2 API chain_id for signing, not a raw provider chain id.
const marketsResponse = await fetch("https://api.o2.app/v1/markets").then((r) => r.json());
const o2ChainId = BigInt(marketsResponse.chain_id);

// Sessions are for trading actions. This is not the withdrawal signer.
const market = await client.getMarket("ETH/USDC");
await client.createSession(ownerSigner, [market], 30);
```

For plain withdrawal-only flows, `setupAccount(ownerSigner)` and `tradeAccountId` are the important pieces. Session creation is needed when the same bot is also placing/canceling O2 orders.

### Withdrawal Helper Snippet

This is the shape to expose in an application wrapper. This method is not on stock `O2Client` today; implement it by following the lower-level flow in the next section.

```ts
const txId = await client.withdrawToChain(
  ownerSigner,
  tradeAccountId,
  "uwUSDC",
  "0.5",
  {
    chainId: 8453,
    recipientAddress: "0x1234567890abcdef1234567890abcdef12345678",
  },
  {
    fastbridgeContracts: {
      fastBridgeAssetRegistryContractId:
        "0x91cfcbef2caad02996cdcb5b897222170e85a91cd2db8f23f07ea7d9ca030c19",
      fastBridgeAssetsMinterContractId:
        "0x0f9f509374c2da68997a3a1ad6d85be3f351c2f5f3da4c65bdbfad9b0bb25504",
    },
  },
);
```

Equivalent wrapper:

```ts
async function withdrawFundsToEvm({
  client,
  ownerSigner,
  tradeAccountId,
  asset,
  amount,
  destinationChainId,
  destinationAddress,
}: {
  client: {
    withdrawToChain(
      ownerSigner: unknown,
      tradeAccountId: string,
      asset: string,
      amount: string | bigint,
      to: { chainId: number; recipientAddress: string },
      options: {
        fastbridgeContracts: {
          fastBridgeAssetRegistryContractId: string;
          fastBridgeAssetsMinterContractId: string;
        };
      },
    ): Promise<string>;
  };
  ownerSigner: unknown;
  tradeAccountId: string;
  asset: string;
  amount: string | bigint;
  destinationChainId: number;
  destinationAddress: string;
}) {
  return client.withdrawToChain(
    ownerSigner,
    tradeAccountId,
    asset.trim(),
    amount,
    {
      chainId: destinationChainId,
      recipientAddress: destinationAddress,
    },
    {
      fastbridgeContracts: {
        fastBridgeAssetRegistryContractId:
          "0x91cfcbef2caad02996cdcb5b897222170e85a91cd2db8f23f07ea7d9ca030c19",
        fastBridgeAssetsMinterContractId:
          "0x0f9f509374c2da68997a3a1ad6d85be3f351c2f5f3da4c65bdbfad9b0bb25504",
      },
    },
  );
}
```

## What The Helper Must Do

The helper must perform the parts that are easy to get wrong:

- derives the asset sub ID from the asset symbol
- derives the wrapped asset ID from `WrappedAssetsMinter`
- scales decimal-string amounts to 9 decimals
- fetches the fast-bridge fee quote from `GasOracle.get_withdrawal_fee(destinationChainId, wrappedAssetId)`
- uses gross debit as the account-action amount when the caller asks for a desired net EVM amount
- encodes `AssetRegistry.withdraw_via_fast_bridge_with_fee`
- pads the EVM recipient internally for the contract call
- fetches the trade-account nonce
- signs with the owner signer
- simulates the trade-account contract call
- submits a `WithdrawViaFastBridgeWithFee` account action to O2

### TypeScript Flow

For the tested standalone TypeScript implementation snippets, read:

```text
./references/typescript-withdrawal-flow.md
```

Use that reference when an agent needs to implement the flow with stock public packages and ordinary REST/RPC calls. It includes the tested steps for fee quote, `uw*` asset derivation, nonce and O2 chain-id fetch, manual call-data encoding, owner signing bytes, optional simulation, and direct `POST /v1/accounts/actions`.

## Rust Implementation Notes

The Rust `o2-sdk` does not natively expose fast-bridge withdrawal yet. Use `../../o2-sdk/rust/SKILL.md` for normal Rust setup, account creation, owner wallet loading, sessions, and trading.

For the tested standalone Rust implementation snippets, read:

```text
./references/rust-withdrawal-flow.md
```

Use that reference when an agent needs to implement the flow with stock public crates and ordinary REST/RPC calls. It includes the tested steps for GasOracle binding, fee quote, `uwUSDC` asset derivation, nonce and O2 chain-id fetch, manual call-data encoding, owner signing bytes, and direct `POST /v1/accounts/actions`.

Signing rules:

- Load an EVM owner with `client.load_evm_wallet(...)`; load a Fuel owner with `client.load_wallet(...)`.
- `o2_chain_id` is the O2 SDK/network config chain ID used for account actions. Do not replace it with a raw Fuel provider chain ID if they differ.
- Always call `owner.personal_sign(&signing_bytes)` from the Rust SDK `SignableWallet` trait.
- The Rust SDK chooses Fuel personal signing for `Wallet` and Ethereum `personal_sign` for `EvmWallet`.
- The owner B256 for `EvmWallet` is the 20-byte EVM address left-padded to 32 bytes.
- The owner B256 for `Wallet` is the Fuel b256 address derived from the Fuel key.
- Submit the signature as `{ "Secp256k1": "0x..." }` even for EVM owners.
- Do not pre-hash, prefix manually, or sign with the session key.

Rust bridge rules:

- Use `gross_debit = desired_net_amount + fee_quote`.
- Use `amount = gross_debit` in the O2 account action and simulation call params.
- Use a plain 20-byte EVM recipient in the O2 API JSON.
- Use a 32-byte left-padded EVM recipient only inside the AssetRegistry contract calldata.
- Simulate before posting to O2.

## Recipient Encoding

This is the most common mistake.

```text
withdrawToChain-style helper:
  recipientAddress = "0x1234..."       // plain EVM address

Direct AssetRegistry call:
  recipient = "0x0000000000000000000000001234..." // 32-byte left-padded EVM address
```

Do not pass a padded 32-byte address into a `withdrawToChain`-style helper.

## Direct Fuel Contract Path

Use this only when funds are in a plain Fuel wallet and the user is intentionally bypassing the O2 trading-account action path.

```text
GasOracle.get_withdrawal_fee(chain_id, asset_id)
AssetRegistry.withdraw_via_fast_bridge_with_fee(sub_id, destination_chain, recipient, fee_quote)
```

Direct call example using desired-net semantics:

```ts
import { parseUnits, zeroPadValue } from "ethers";
import { Contract } from "fuels";
import AssetRegistryAbi from "./abis/AssetRegistry-abi.json";
import GasOracleAbi from "./abis/GasOracle-abi.json";

const desiredNetAmount = parseUnits(userAmount, 9);
const assetRegistry = new Contract(assetRegistryContractId, AssetRegistryAbi, wallet);
const gasOracle = new Contract(gasOracleContractId, GasOracleAbi, wallet);

const feeQuote = await gasOracle.functions
  .get_withdrawal_fee(destinationChainId, { bits: assetId })
  .get();

const grossDebit = desiredNetAmount + BigInt(feeQuote.value.toString());

await assetRegistry.functions
  .withdraw_via_fast_bridge_with_fee(
    subId,
    destinationChainId,
    zeroPadValue(evmRecipientAddress, 32),
    feeQuote.value.toString(),
  )
  .callParams({
    forward: {
      assetId,
      amount: grossDebit.toString(),
    },
  })
  .call();
```

Direct path rules:

- Fetch fee from `GasOracle.get_withdrawal_fee(chain_id, asset_id)`.
- Use the universal wrapped symbol/sub ID, such as `uwUSDC`, not the source-chain symbol `USDC`.
- `amount` inside `.callParams.forward` is the gross debit.
- If the caller asks to receive a specific net amount on EVM, forward `desiredNetAmount + feeQuote`.
- If the caller passes a gross debit amount, expected EVM-side net is `grossDebit - currentRequiredFee`.
- Encode EVM recipient as 32-byte left-padded.
- Use 9-decimal wrapped amount.

## Unwrap vs Bridge

Use `AssetRegistry.unwrap_to_canonical_asset(canonical_asset_id, recipient)` only when the user wants a canonical asset on Fuel.

Do not describe unwrap as an EVM withdrawal. Unwrap does not send assets to Base or Ethereum.

## Withdrawal Checklist

For a trader moving funds across the bridge:

1. Ensure O2 account exists and keep `trade_account_id`.
2. Keep owner signer available for withdrawals.
3. Use session key only for order actions.
4. Settle funds into the trading account before withdrawal if needed.
5. Call or implement a `withdrawToChain`-style helper with a plain EVM recipient.
6. Track the returned `txId`.
7. Expect asynchronous EVM settlement after the Fuel-side action succeeds.

## Common Failures

- user has funds in `trade_account_id` but tries a plain wallet `AssetRegistry` withdrawal
- session key used for withdrawal instead of owner signer
- missing `trade_account_id`
- stale nonce in account action
- hashing `USDC`, `ETH`, or `FUEL` instead of the registered universal wrapped symbol such as `uwUSDC`, `uwETH`, or `uwFUEL`
- destination recipient padded before passing to a `withdrawToChain`-style helper
- amount scaled with destination token decimals instead of 9
- direct contract call forwards only withdrawal amount and forgets fee
- unsupported destination chain or missing route mapping
- fee drift exceeds tolerance
- amount is too small after destination-decimal conversion
- user meant unwrap, not EVM bridge withdrawal

When answering, first identify which path the funds are in: O2 trading account, plain Fuel wallet, or Fuel unwrap. Then show the exact source-side call and the one identifier that matters for that path.
