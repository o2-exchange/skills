# TypeScript Fast-Bridge Deposit Flow

## Install Dependencies

Add the packages used by this reference flow:

```bash
npm install @o2exchange/sdk ethers
```

## Setup

```ts
import { Network, O2Client } from "@o2exchange/sdk";
import {
  Contract as EvmContract,
  JsonRpcProvider,
  Wallet as EvmWallet,
} from "ethers";

import IERC20MetadataArtifact from "../abis/IERC20Metadata.json";
import MessengerArtifact from "../abis/Messenger.json";

const sourceChain = {
  chainId: 8453,
  rpcUrl: "https://mainnet.base.org",
  messengerAddress: "0x2B1c1E133F832EFB1e168dE6102304B03C4ba653",
};

const client = new O2Client({ network: Network.MAINNET });
const ownerSigner = O2Client.loadEvmWallet(ownerPrivateKey);
const account = await client.setupAccount(ownerSigner);
const tradeAccountId = account.tradeAccountId;

const provider = new JsonRpcProvider(sourceChain.rpcUrl, sourceChain.chainId);
const wallet = new EvmWallet(ownerPrivateKey, provider);
const messenger = new EvmContract(
  sourceChain.messengerAddress,
  MessengerArtifact.abi,
  wallet,
);
```

Deposit native ETH:

```ts
import { parseEther } from "ethers";

const value = parseEther(amountText);
const tx = await messenger.depositETH(
  tradeAccountId,
  true,
  { value },
);

const receipt = await tx.wait();
```

Deposit ERC-20 such as Base USDC:

```ts
import { MaxUint256, parseUnits } from "ethers";

const tokenAddress = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
const token = new EvmContract(
  tokenAddress,
  IERC20MetadataArtifact.abi,
  wallet,
);

const decimals = Number(await token.decimals());
const amount = parseUnits(amountText, decimals);
const ownerAddress = await wallet.getAddress();
const allowance = await token.allowance(
  ownerAddress,
  sourceChain.messengerAddress,
);

if (allowance < amount) {
  await (await token.approve(sourceChain.messengerAddress, MaxUint256)).wait();
}

const tx = await messenger.deposit(
  tradeAccountId,
  tokenAddress,
  amount,
  true,
);

const receipt = await tx.wait();
```

Alternative: `depositWithPermit` if the token supports EIP-2612:

```ts
const tx = await messenger.depositWithPermit(
  tradeAccountId,
  tokenAddress,
  amount,
  deadline,
  v,
  r,
  s,
  true,
);

const receipt = await tx.wait();
```

What matters:

- Use `tradeAccountId` as the default O2 recipient.
- Set `recipientIsContract = true` for an O2 trading account.
- Deposit amounts use source-token EVM decimals, not 9 decimals.
- After the source transaction confirms, O2 crediting is asynchronous.
