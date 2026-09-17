# Python Fast Bridge Withdrawal

Use `FastBridgeClient` from `o2-sdk` 0.5 or newer. This flow spends a funded Fuel wallet, not an O2 trading account/session.

```python
import time

from o2_sdk import (
    FAST_BRIDGE_TESTNET_URL,
    FastBridgeClient,
    fuel_compact_sign,
    parse_fuel_unsigned_transaction,
    parse_preparation_proof,
)
from o2_sdk.bridge.models import SubmitRequest, WithdrawPrepareRequest


async def withdraw(
    fuel_address,
    evm_recipient,
    full_fuel_asset_id,
    private_key,
    trusted_fuel_chain_id,
    trusted_max_inputs,
):
    async with FastBridgeClient(FAST_BRIDGE_TESTNET_URL) as client:
        request = WithdrawPrepareRequest(
            destination_chain_id=11155111,
            from_address=fuel_address,  # serializes as "from"
            to=evm_recipient,
            asset_id=full_fuel_asset_id,
            amount="1000000",           # gross Fuel asset base units
        )

        prepared = await client.prepare_withdraw(request)
        if int(prepared.fuel_chain_id) != trusted_fuel_chain_id:
            raise ValueError("Unexpected Fuel chain")

        claims = parse_preparation_proof(prepared.preparation_proof)
        if claims.expires_at <= time.time():
            raise ValueError("Prepare again: proof expired")

        # trusted_max_inputs is consensus data not encoded in the transaction.
        inspected = parse_fuel_unsigned_transaction(
            prepared.unsigned_transaction,
            trusted_fuel_chain_id,
            trusted_max_inputs,
        )
        print(inspected)
        # Check contract, route, asset, gross/fee/net amounts, fees, expiry,
        # every input owner/asset, and outputs against trusted configuration.
        if not approve_withdrawal(request, inspected):
            raise ValueError("Prepared withdrawal does not match intent")

        signature = fuel_compact_sign(
            private_key,
            bytes.fromhex(inspected.transaction_id[2:]),
        )
        submitted = await client.submit_withdraw(
            SubmitRequest(
                unsigned_transaction=prepared.unsigned_transaction,
                preparation_proof=prepared.preparation_proof,
                signature="0x" + signature.hex(),
            )
        )
        return inspected.transaction_id, submitted
```

Sign the locally computed raw transaction ID. Do not personal-sign, hash again, reconstruct the transaction, or include `fuel_chain_id` in submit.

## Status

Use `client.get_withdraw_status(transaction_id)` with bounded backoff. A `BridgeApiError` with status 404 means not found, not pending. A submit timeout is ambiguous, so reconcile status before resubmitting. Fuel inclusion and destination delivery are separate states.

The request amount is gross. Inspect `gross_amount`, `bridge_fee`, and `net_amount`. `network_fee.max_fee` is a separate Fuel base-asset fee cap.

If assets are in an O2 trading account, first use normal `O2Client.withdraw(...)` to fund the owner Fuel wallet. Parsed proof claims are display-only and unauthenticated; the proxy performs the HMAC verification.
