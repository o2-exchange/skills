# Python Fast-Bridge Deposit Flow

## Install Dependencies

Add the packages used by this reference flow:

```bash
pip install o2-sdk web3 python-dotenv
```

## Setup

```python
from o2_sdk import Network, O2Client
from web3 import Web3

MESSENGER_ADDRESS = "0x2B1c1E133F832EFB1e168dE6102304B03C4ba653"
BASE_USDC_ADDRESS = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"
BASE_RPC_URL = "https://mainnet.base.org"
BASE_CHAIN_ID = 8453

client = O2Client(network=Network.MAINNET)
owner = client.load_evm_wallet(owner_private_key)
account = await client.setup_account(owner)
trade_account_id = account.trade_account_id

w3 = Web3(Web3.HTTPProvider(BASE_RPC_URL))
evm_account = w3.eth.account.from_key(owner_private_key)
```

Load the bundled ABIs:

```python
import json
from pathlib import Path

root = Path(".")
with open(root / "skills/fast-bridge/deposits/abis/Messenger.json", "r") as f:
    messenger_abi = json.load(f)["abi"]

with open(root / "skills/fast-bridge/deposits/abis/IERC20Metadata.json", "r") as f:
    erc20_abi = json.load(f)["abi"]

messenger = w3.eth.contract(
    address=Web3.to_checksum_address(MESSENGER_ADDRESS),
    abi=messenger_abi,
)

token = w3.eth.contract(
    address=Web3.to_checksum_address(BASE_USDC_ADDRESS),
    abi=erc20_abi,
)
```

## Deposit Base USDC

```python
from decimal import Decimal

decimals = int(token.functions.decimals().call())
amount_text = "0.5"
amount = int(Decimal(amount_text) * (10 ** decimals))

allowance = int(
    token.functions.allowance(
        evm_account.address,
        Web3.to_checksum_address(MESSENGER_ADDRESS),
    ).call()
)

if allowance < amount:
    approve_tx = token.functions.approve(
        Web3.to_checksum_address(MESSENGER_ADDRESS),
        (1 << 256) - 1,
    ).build_transaction(
        {
            "from": evm_account.address,
            "chainId": BASE_CHAIN_ID,
            "nonce": w3.eth.get_transaction_count(evm_account.address),
        }
    )
    approve_tx["gas"] = int(token.functions.approve(
        Web3.to_checksum_address(MESSENGER_ADDRESS),
        (1 << 256) - 1,
    ).estimate_gas({"from": evm_account.address}) * 1.2)
    signed_approve = evm_account.sign_transaction(approve_tx)
    approve_hash = w3.eth.send_raw_transaction(signed_approve.raw_transaction)
    w3.eth.wait_for_transaction_receipt(approve_hash)

deposit_tx = messenger.functions.deposit(
    trade_account_id,
    Web3.to_checksum_address(BASE_USDC_ADDRESS),
    amount,
    True,
).build_transaction(
    {
        "from": evm_account.address,
        "chainId": BASE_CHAIN_ID,
        "nonce": w3.eth.get_transaction_count(evm_account.address),
    }
)

deposit_tx["gas"] = int(
    messenger.functions.deposit(
        trade_account_id,
        Web3.to_checksum_address(BASE_USDC_ADDRESS),
        amount,
        True,
    ).estimate_gas({"from": evm_account.address}) * 1.2
)

signed_deposit = evm_account.sign_transaction(deposit_tx)
deposit_hash = w3.eth.send_raw_transaction(signed_deposit.raw_transaction)
receipt = w3.eth.wait_for_transaction_receipt(deposit_hash)
print(receipt["transactionHash"].hex())
```

## Deposit Native ETH

```python
value = w3.to_wei("0.01", "ether")

deposit_tx = messenger.functions.depositETH(
    trade_account_id,
    True,
).build_transaction(
    {
        "from": evm_account.address,
        "chainId": BASE_CHAIN_ID,
        "nonce": w3.eth.get_transaction_count(evm_account.address),
        "value": value,
    }
)
```

What matters:

- Use `trade_account_id` as the default O2 recipient.
- Set `recipientIsContract=True` for an O2 trading account.
- Deposit amounts use source-token EVM decimals, not 9 decimals.
- After the source transaction confirms, O2 crediting is asynchronous.
