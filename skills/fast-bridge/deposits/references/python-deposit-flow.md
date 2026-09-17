# Python Fast Bridge Deposit

Use `FastBridgeClient` from `o2-sdk` 0.5 or newer. No additional cryptography package is required for the raw digest-signing path.

```python
import time

from o2_sdk import (
    FAST_BRIDGE_TESTNET_URL,
    FastBridgeClient,
    fuel_compact_sign,
    parse_evm_unsigned_transaction,
    parse_preparation_proof,
)
from o2_sdk.bridge.models import DepositPrepareRequest, SubmitRequest


async def deposit(evm_address, fuel_recipient, full_fuel_asset_id, private_key):
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
        if claims.expires_at <= time.time():
            raise ValueError("Prepare again: proof expired")

        inspected = parse_evm_unsigned_transaction(prepared.unsigned_transaction)
        print(inspected)
        # Compare chain_id, nonce, messenger_address, method, recipient/type,
        # token_address, amount/value, gas and fee caps with request and trusted
        # configuration. Parsing itself is not approval.
        if not approve_deposit(request, inspected):
            raise ValueError("Prepared deposit does not match intent")

        # Sign the locally computed raw digest. fuel_compact_sign returns r || yParityAndS.
        compact = bytearray(
            fuel_compact_sign(
                private_key,
                bytes.fromhex(inspected.signing_digest[2:]),
            )
        )
        signature = compact
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

Use bounded status polling. A submit timeout is ambiguous, so reconcile chain/status data before resubmitting.

## Native ETH, ERC-20, And Permit

The proxy selects `depositETH`, `deposit`, or `depositWithPermit` from its trusted route configuration. ERC-20 needs an existing Messenger allowance or a `DepositPermit` in the prepare request. That EIP-2612 permit is a separate token-approval signature, not the EVM transaction signature.

`parse_preparation_proof` exposes `version`, `key_id`, `expires_at`, and `signer` for display only. The claims are unauthenticated until the proxy verifies the proof's HMAC and exact transaction binding.
