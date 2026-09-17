# Rust Fast Bridge Withdrawal

Use `o2-sdk` 0.4 or newer. This flow spends a funded Fuel wallet, not an O2 trading account/session.

```toml
[dependencies]
o2-sdk = "0.4"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

```rust
use o2_sdk::bridge::{
    parse_fuel_unsigned_transaction, parse_preparation_proof,
    FastBridgeClient, FuelWithdrawalInspection, SubmitRequest,
    WithdrawPrepareRequest, WithdrawSubmitResponse,
};
use o2_sdk::crypto::{fuel_compact_sign, parse_hex_32, to_hex_string};
use std::{error::Error, time::{SystemTime, UNIX_EPOCH}};

async fn withdraw(
    client: &FastBridgeClient,
    request: &WithdrawPrepareRequest,
    private_key: &[u8; 32],
    trusted_fuel_chain_id: u64,
    trusted_max_inputs: u16,
    approve: impl FnOnce(&WithdrawPrepareRequest, &FuelWithdrawalInspection) -> bool,
) -> Result<(String, WithdrawSubmitResponse), Box<dyn Error>> {
    let prepared = client.prepare_withdraw(request).await?;
    if prepared.fuel_chain_id.parse::<u64>()? != trusted_fuel_chain_id {
        return Err("Unexpected Fuel chain".into());
    }

    let claims = parse_preparation_proof(&prepared.preparation_proof)?;
    // trusted_max_inputs is consensus data not encoded in the transaction.
    let inspected = parse_fuel_unsigned_transaction(
        &prepared.unsigned_transaction,
        trusted_fuel_chain_id,
        trusted_max_inputs,
    )?;
    println!("{inspected:#?}");

    // Check contract, route, asset, gross/fee/net amounts, fees, block expiry,
    // every input owner/asset, and all outputs against trusted configuration.
    if !approve(request, &inspected) {
        return Err("Prepared withdrawal does not match intent".into());
    }

    // Recheck immediately before signing/submission in case approval took time.
    if claims.expires_at <= SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() {
        return Err("Prepare again: proof expired".into());
    }

    let transaction_id = inspected.transaction_id;
    let signature = fuel_compact_sign(private_key, &parse_hex_32(&transaction_id)?)?;
    let submitted = client
        .submit_withdraw(&SubmitRequest {
            unsigned_transaction: prepared.unsigned_transaction,
            preparation_proof: prepared.preparation_proof,
            signature: to_hex_string(&signature),
        })
        .await?;
    Ok((transaction_id, submitted))
}
```

Get `trusted_fuel_chain_id` and `trusted_max_inputs` from trusted Fuel RPC consensus data or reviewed application configuration. Sign the locally computed raw transaction ID without extra hashing. A wallet or external signer that supports raw-digest signing may replace direct private-key handling.

## Status

Use `client.get_withdraw_status(&transaction_id)` with bounded backoff. `BridgeError::Api { status: 404, .. }` means not found. Stop when the Fuel state becomes `Success` or `Reverted`; the proxy's destination status remains `Unavailable` today. Confirm EVM delivery separately through the recipient balance or relevant trusted Outpost event after the expected relay delay. A submit timeout is ambiguous, so query status before resubmitting.

The request amount is gross. Inspect `gross_amount`, `bridge_fee`, `net_amount`, and the separate `network_fee.max_fee`. If assets are held by an O2 trading account, first use normal `O2Client::withdraw(...)` to fund the owner Fuel wallet.

Parsed proof claims are display-only; only the proxy authenticates the proof and exact transaction binding.
