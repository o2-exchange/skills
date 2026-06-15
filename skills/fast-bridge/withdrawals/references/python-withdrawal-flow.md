# Python Fast-Bridge Withdrawal Flow

Python 3.10+ required.

## Install Dependencies

Add the packages used by this reference flow:

```bash
pip install o2-sdk web3 python-dotenv
```

Also require `forc` CLI for the fee-quote step.

## Setup

This is an SDK-based reference. It uses the Python SDK for owner identity, account setup, signing helpers, and account lookups. For the underlying byte layout, see `../SKILL.md`.

```python
import hashlib
import aiohttp

from o2_sdk import GAS_MAX, Network, O2Client, build_actions_signing_bytes, function_selector, u64_be

API_BASE = "https://api.o2.app"
BASE_CHAIN_ID = 8453
FUEL_RPC_URL = "https://mainnet.fuel.network/v1/graphql"
ASSET_REGISTRY_CONTRACT_ID = "0x91cfcbef2caad02996cdcb5b897222170e85a91cd2db8f23f07ea7d9ca030c19"
WRAPPED_ASSETS_MINTER_CONTRACT_ID = "0x0f9f509374c2da68997a3a1ad6d85be3f351c2f5f3da4c65bdbfad9b0bb25504"
GAS_ORACLE_CONTRACT_ID = "0x3d20e5a675c5fa1053fba11e176099711ba2f23112e385a7f6e2e759eca84f94"
FAST_BRIDGE_ASSET_SYMBOL = "uwUSDC"

client = O2Client(network=Network.MAINNET)
owner = client.load_evm_wallet(owner_private_key)
account = await client.setup_account(owner)
trade_account_id = account.trade_account_id
```

Expected setup result:

```json
{
  "trade_account_id": "0x..."
}
```

## Resolve The Wrapped Asset

Use the universal wrapped bridge asset, not the market symbol alone.

```python
desired_net_amount = 300_000_000  # 0.3 in 9 decimals

asset_sub_id = "0x" + hashlib.sha256(FAST_BRIDGE_ASSET_SYMBOL.encode("utf-8")).hexdigest()
wrapped_asset_id = "0x" + hashlib.sha256(
    bytes.fromhex(WRAPPED_ASSETS_MINTER_CONTRACT_ID[2:]) + bytes.fromhex(asset_sub_id[2:])
).hexdigest()
```

## Fetch The Fee Quote From `GasOracle`

For Python in this skill, use `forc call` instead of assuming a Python Fuel deployed-contract client.

```python
import json
import subprocess

result = subprocess.run(
    [
        "forc",
        "call",
        GAS_ORACLE_CONTRACT_ID,
        "--mainnet",
        "--abi",
        "skills/fast-bridge/withdrawals/abis/GasOracle-abi.json",
        "get_withdrawal_fee",
        str(BASE_CHAIN_ID),
        "{" + wrapped_asset_id + "}",
        "--external-contracts",
        "0x801af4ee92bd9e64ac16b65f490d4cd7dae791662ffbc8f70e20cdb7a6b7fa8c",
        "--mode",
        "dry-run",
        "-o",
        "json",
    ],
    capture_output=True,
    text=True,
    check=True,
)

fee_quote = None
for line in result.stdout.splitlines():
    line = line.strip()
    if not line.startswith("{"):
        continue
    parsed = json.loads(line)
    if "result" in parsed:
        fee_quote = int(parsed["result"])
        break
    message = (parsed.get("fields") or {}).get("message")
    if isinstance(message, str) and message.startswith("result::"):
        fee_quote = int(message.split("::", 1)[1].strip())
        break

if fee_quote is None:
    raise RuntimeError("Could not parse fee quote from forc output")

gross_debit = desired_net_amount + fee_quote
```

Important argument detail:

```text
AssetId argument syntax for forc call: '{<wrappedAssetId>}'
```

Assumptions:

- The ABI path above assumes your current working directory is the root of the cloned `o2-skills` repo.
- `0x801af4ee92bd9e64ac16b65f490d4cd7dae791662ffbc8f70e20cdb7a6b7fa8c` is the current mainnet dependency contract used by this GasOracle read path. Treat it as version-sensitive and re-check if the GasOracle implementation changes.

## Fetch Nonce And O2 Chain ID

```python
nonce = await client.get_nonce(trade_account_id)

markets = await client.api.get_markets()
chain_id_raw = markets["chain_id"] if isinstance(markets, dict) else markets.chain_id
o2_chain_id = int(chain_id_raw, 16) if isinstance(chain_id_raw, str) and chain_id_raw.startswith("0x") else int(chain_id_raw)
```

If you are not using the SDK helper for nonce, fetch it from:

```text
GET https://api.o2.app/v1/accounts?trade_account_id=<trade_account_id>
```

Read it defensively from `response.nonce` or `response.trade_account.nonce`.

## Build The AssetRegistry Call Data

The target contract call is:

```text
withdraw_via_fast_bridge_with_fee(sub_id, destination_chain, recipient, fee_quote)
```

Encode the 76-byte call-data payload manually:

```python
def u32_be(value: int) -> bytes:
    return value.to_bytes(4, "big", signed=False)

def pad_evm_address(address: str) -> bytes:
    raw = bytes.fromhex(address[2:])
    return (b"\x00" * 12) + raw

destination_evm_address = "0xb66cbadd2b30debfe1d0a77deeab0f56875cd0d7"

call_data = (
    bytes.fromhex(asset_sub_id[2:])
    + u32_be(BASE_CHAIN_ID)
    + pad_evm_address(destination_evm_address)
    + u64_be(fee_quote)
)
```

## Build The Owner Signing Bytes

```python
def u64_be(value: int) -> bytes:
    return value.to_bytes(8, "big", signed=False)

def selector_bytes(name: str) -> bytes:
    encoded = name.encode("utf-8")
    return u64_be(len(encoded)) + encoded

def build_actions_signing_bytes(nonce: int, calls: list[dict]) -> bytes:
    parts = [u64_be(nonce), u64_be(len(calls))]
    for call in calls:
        call_data = call.get("call_data")
        parts.extend(
            [
                call["contract_id"],
                u64_be(len(call["function_selector"])),
                call["function_selector"],
                u64_be(call["amount"]),
                call["asset_id"],
                u64_be(call["gas"]),
            ]
        )
        if call_data is None:
            parts.append(u64_be(0))
        else:
            parts.extend([u64_be(1), u64_be(len(call_data)), call_data])
    return b"".join(parts)

withdraw_selector = function_selector("withdraw_via_fast_bridge_with_fee")

calls_signing_bytes = build_actions_signing_bytes(
    nonce,
    [
        {
            "contract_id": bytes.fromhex(ASSET_REGISTRY_CONTRACT_ID[2:]),
            "function_selector": withdraw_selector,
            "amount": gross_debit,
            "asset_id": bytes.fromhex(wrapped_asset_id[2:]),
            "gas": GAS_MAX,
            "call_data": call_data,
        }
    ],
)

signing_bytes = (
    u64_be(nonce)
    + u64_be(o2_chain_id)
    + function_selector("call_contracts")
    + calls_signing_bytes[8:]
)

signature = owner.personal_sign(signing_bytes)
signature_hex = "0x" + signature.hex()
```

For EVM owners, be explicit about the signature format:

- Ethereum `personal_sign` normally returns 65 bytes: `r || s || v`.
- O2 expects Fuel's 64-byte compact Secp256k1 format in the JSON payload.
- If your SDK helper already returns the compact 64-byte form, use it directly.
- If it returns a 65-byte Ethereum signature, convert it before submitting.

## Submit The O2 Account Action

```python
payload = {
    "actions": [
        {
            "WithdrawViaFastBridgeWithFee": {
                "amount": str(gross_debit),
                "fee_quote": str(fee_quote),
                "asset": {
                    "sub_id": asset_sub_id,
                    "universal": wrapped_asset_id,
                },
                "recipient": {
                    "Evm": {
                        "chain_id": str(BASE_CHAIN_ID),
                        "recipient": {
                            "address": destination_evm_address,
                        },
                    }
                },
            }
        }
    ],
    "signature": {"Secp256k1": signature_hex},
    "nonce": str(nonce),
    "trade_account_id": trade_account_id,
    "variable_outputs": 1,
    "contracts": [ASSET_REGISTRY_CONTRACT_ID],
}

headers = {
    "content-type": "application/json",
    "accept": "application/json",
    "O2-Owner-Id": owner.b256_address,
}

async with aiohttp.ClientSession() as session:
    async with session.post(
        f"{API_BASE}/v1/accounts/actions",
        json=payload,
        headers=headers,
    ) as response:
        body = await response.text()
        if response.status >= 400:
            raise RuntimeError(f"O2 withdrawal action failed ({response.status}): {body}")
        print(body)
```

What matters:

- Treat the user input as the desired net EVM-side amount.
- Fetch `fee_quote` from `GasOracle.get_withdrawal_fee(...)`.
- Compute `gross_debit = desired_net_amount + fee_quote`.
- Use `uwUSDC` for the bridge asset symbol, not plain `USDC`.
- Sign with the owner wallet, not the session key.
- Send the EVM recipient as a normal `0x...` address in the O2 API JSON.
- Left-pad that same EVM address to 32 bytes inside `call_data`.
