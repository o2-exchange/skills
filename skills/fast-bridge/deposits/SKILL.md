---
name: o2-fast-bridge-deposits
description: Explains O2 fast bridge deposits from an EVM source chain into O2/Fuel. Use for EVM-to-O2 deposits, Messenger.deposit, depositWithPermit, depositETH, recipientIsContract, O2 trade_account_id contract recipients, owner b256 wallet recipients, source-token decimals, whitelist/cap failures, deposit hashes, and relayer timing.
---

# O2 Fast Bridge Deposits

Use this skill when an agent must help a trader fund O2 from an EVM chain through the fast bridge.

For O2 account setup and `trade_account_id` discovery, use:

- `../../o2-sdk/typescript/SKILL.md`
- `../../o2-sdk/python/SKILL.md`

This skill is focused on the fast bridge deposit flow only.

Use these bundled ABIs when writing code:

```text
./abis/Messenger.json
./abis/IERC20Metadata.json
./abis/IERC20Permit.json
```

For reference snippets see:

```text
./references/python-deposit-flow.md
./references/typescript-deposit-flow.md
./references/rust-deposit-flow.md
```

## Install Dependencies

TypeScript / JavaScript:

```bash
npm install @o2exchange/sdk ethers
```

Python:

```bash
pip install o2-sdk web3
```

Rust:

```bash
cargo add anyhow ethers o2-sdk tokio
```

## User Action

The user sends one source-chain EVM transaction to `Messenger`.

```text
Messenger.deposit(bytes32 to, address token, uint256 value, bool recipientIsContract)
Messenger.depositWithPermit(bytes32 to, address token, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s, bool recipientIsContract)
Messenger.depositETH(bytes32 to, bool recipientIsContract)
IERC20Metadata.decimals() -> uint8
```

Mainnet source-chain defaults:

```text
Base mainnet (chainId 8453)
- Messenger: 0x2B1c1E133F832EFB1e168dE6102304B03C4ba653
- USDC:      0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
- FUEL:      0xFdedBefEc262fE0eeaA2bbdE1afA2ef09AaB6634

Ethereum mainnet (chainId 1)
- Messenger: 0x2B1c1E133F832EFB1e168dE6102304B03C4ba653
- USDC:      0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
- FUEL:      0x675B68AA4d9c2d3BB3F0397048e62E6B7192079c
```

## O2 Recipient Rule

For O2 trading-account funding, put the O2 `trade_account_id` into `Messenger.deposit`.

The O2 trading account is a Fuel contract. That means the default O2 deposit uses:

```text
to = trade_account_id
recipientIsContract = true
```

An owner `b256` recipient is different. For an EVM owner, the owner `b256` often looks like the 20-byte EVM owner address left-padded to 32 bytes. Use that only when the user is intentionally depositing to the owner identity instead of the O2 trading-account contract.

```text
Owner EVM address: 0xb5ab972BFe6B73382C138240441De34229244932
Owner b256:        0x000000000000000000000000b5ab972BFe6B73382C138240441De34229244932
Trade account ID:  0xf11b921863b55a03c7c770d4bea4f99a43cc34248fbce6320d4e3bc43d6a8e1f
```

Correct:

```ts
import { Contract, parseUnits } from "ethers";
import MessengerArtifact from "./abis/Messenger.json";
import IERC20MetadataArtifact from "./abis/IERC20Metadata.json";

const tradeAccountId =
  "0xf11b921863b55a03c7c770d4bea4f99a43cc34248fbce6320d4e3bc43d6a8e1f";
const messenger = new Contract(messengerAddress, MessengerArtifact.abi, signer);
const token = new Contract(usdcAddress, IERC20MetadataArtifact.abi, signer);
const amount = parseUnits("0.5", await token.decimals());

await messenger.deposit(
  tradeAccountId,
  usdcAddress,
  amount,
  true, // O2 trading account is a Fuel contract
);
```

Wrong:

```ts
await messenger.deposit(
  ownerB256, // wrong for default O2 trading-account funding
  usdcAddress,
  amount,
  false,
);
```

## Amount Rule

Deposit inputs use the source asset's EVM decimals, not Fuel's 9-decimal wrapped-asset format.

- ERC-20: `parseUnits(userAmount, await token.decimals())`
- ETH: `parseEther(userAmount)`
- Raw bigint: already scaled to the source asset's native decimals

Examples:

```ts
import { parseEther, parseUnits } from "ethers";

const usdcAmount = parseUnits("0.5", 6); // 500000 raw USDC
const fuelAmount = parseUnits("12.34", 18); // FUEL ERC-20 uses its own token decimals
const ethAmount = parseEther("0.01");
```

`Messenger` normalizes the deposit to the bridge's 9-decimal format internally. If source decimals are greater than 9, the amount must scale down cleanly or the transaction reverts.

On Fuel/O2, the credited bridge asset is the universal wrapped asset, such as `uwUSDC`, `uwETH`, or `uwFUEL`. Deposits still use the source-chain token address and source token decimals on EVM; the `uw` symbol matters later when deriving withdrawal `sub_id` / wrapped asset ID.

## Source-Side Examples

ERC-20 deposit:

```ts
import { Contract, MaxUint256, parseUnits } from "ethers";
import MessengerArtifact from "./abis/Messenger.json";
import IERC20MetadataArtifact from "./abis/IERC20Metadata.json";

const messenger = new Contract(messengerAddress, MessengerArtifact.abi, signer);
const token = new Contract(tokenAddress, IERC20MetadataArtifact.abi, signer);
const depositTo = tradeAccountId;
const amount = parseUnits(userAmount, await token.decimals());

const allowance = await token.allowance(ownerAddress, messengerAddress);
if (allowance === 0n) {
  await token.approve(messengerAddress, MaxUint256);
}

const tx = await messenger.deposit(
  depositTo,
  tokenAddress,
  amount,
  true,
);
```

Native ETH deposit:

```ts
import { Contract, parseEther } from "ethers";
import MessengerArtifact from "./abis/Messenger.json";

const messenger = new Contract(messengerAddress, MessengerArtifact.abi, signer);
const depositTo = tradeAccountId;
const value = parseEther(userAmount);

const tx = await messenger.depositETH(
  depositTo,
  true,
  { value },
);
```

## `recipientIsContract`

This flag describes the Fuel-side recipient type emitted in the deposit event.

- O2 owner `b256` / padded EVM owner address: `false`
- Normal Fuel address recipient: `false`
- O2 `trade_account_id`: `true`
- Other Fuel contract ID recipient: `true`

Set `recipientIsContract = true` for the default O2 trading-account deposit because the O2 trading account is a Fuel contract. Set it to `false` only for a normal Fuel address or owner `b256` wallet recipient.

## After the Transaction

Once the EVM transaction emits `Deposit`, the source-side deposit succeeded. The user does not call any destination-side relayer or Fuel bridge method.

Relayer processing is asynchronous. Docs may say around 10 seconds, but production timing can vary. Do not diagnose a successful source `Deposit` event as a user-side failure just because the O2 balance is not visible immediately.

## Checks and Failure Modes

Watch for:

- token not whitelisted in `Messenger`
- user cap exceeded
- missing ERC-20 allowance
- wrong source-chain token address
- wrong source-token decimals
- amount cannot normalize cleanly to 9 decimals
- amount exceeds `uint64` after normalization
- owner `b256` accidentally used as `to` when the user meant to fund an O2 trading account
- wrong `recipientIsContract`

When answering, show only what the user must submit on the EVM source chain, then explain what condition passed or failed.
