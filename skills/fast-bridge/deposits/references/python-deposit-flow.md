# Python Fast Bridge Deposit

Use `FastBridgeClient` from `o2-sdk` 0.5 or newer. No additional cryptography package is required for the raw digest-signing path.

```python
import time
from collections.abc import Callable

from o2_sdk import (
    FAST_BRIDGE_TESTNET_URL,
    FastBridgeClient,
    fuel_compact_sign,
    parse_evm_unsigned_transaction,
    parse_preparation_proof,
)
from o2_sdk.bridge.models import DepositPrepareRequest, SubmitRequest
from o2_sdk.bridge.inspection import EvmDepositInspection


async def deposit(
    evm_address: str,
    fuel_recipient: str,
    full_fuel_asset_id: str,
    private_key: bytes,
    approve: Callable[[DepositPrepareRequest, EvmDepositInspection], bool],
):
    async with FastBridgeClient(FAST_BRIDGE_TESTNET_URL) as client:
        request = DepositPrepareRequest(
            source_chain_id=11155111,
            from_address=evm_address,  # serializes as "from"
            to=fuel_recipient,
            to_type="address",        # "contract" for a Fuel contract
            asset_id=full_fuel_asset_id,
            amount="1000000",         # integer Fuel asset base units
        )

        prepared = await client.prepare_deposit(request)
        claims = parse_preparation_proof(prepared.preparation_proof)
        inspected = parse_evm_unsigned_transaction(prepared.unsigned_transaction)
        print(inspected)
        # Compare chain_id, nonce, messenger_address, method, recipient/type,
        # token_address, amount/value, gas and fee caps with request and trusted
        # configuration. Parsing itself is not approval.
        if not approve(request, inspected):
            raise ValueError("Prepared deposit does not match intent")

        # Recheck immediately before signing/submission in case approval took time.
        if claims.expires_at <= time.time():
            raise ValueError("Prepare again: proof expired")

        # Sign the locally computed raw digest. fuel_compact_sign returns r || yParityAndS.
        signature = bytearray(
            fuel_compact_sign(
                private_key,
                bytes.fromhex(inspected.signing_digest[2:]),
            )
        )
        signature.append(27 + (signature[32] >> 7))
        signature[32] &= 0x7F

        submitted = await client.submit_deposit(
            SubmitRequest(
                unsigned_transaction=prepared.unsigned_transaction,
                preparation_proof=prepared.preparation_proof,
                signature="0x" + signature.hex(),
            )
        )
        return submitted
```

Submit the exact unsigned transaction and proof returned by prepare. The 65-byte EVM signature is separate; do not submit a signed transaction envelope.

## Status

```python
from o2_sdk import BridgeApiError

try:
    status = await client.get_deposit_status(
        submitted.source_chain_id,
        submitted.evm_tx_hash,
    )
except BridgeApiError as error:
    if error.status != 404:
        raise
    status = None  # not found, not fabricated pending
```

Use bounded status polling. Stop when `source.status` becomes `confirmed` or `reverted`; the proxy's `fuel.status` remains `unavailable` today. Confirm final Fuel delivery separately by querying the recipient wallet or contract balance after the expected relay delay. A submit timeout is ambiguous, so reconcile chain/status data before resubmitting.

## Native ETH, ERC-20, And Permit

The proxy selects `depositETH`, `deposit`, or `depositWithPermit` from its trusted route configuration. ERC-20 needs an existing Messenger allowance or a `DepositPermit` in the prepare request. That EIP-2612 permit is a separate token-approval signature, not the EVM transaction signature.

To fund an O2 trading account directly, use its `trade_account_id` as `to` with `to_type="contract"`; use the owner Fuel wallet with `to_type="address"` to fund the wallet itself.

`parse_preparation_proof` exposes display-only claims; only the proxy authenticates the proof and exact transaction binding.
