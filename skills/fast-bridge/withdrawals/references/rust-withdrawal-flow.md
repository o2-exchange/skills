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
    FastBridgeClient, SubmitRequest, WithdrawPrepareRequest,
};
use o2_sdk::crypto::{fuel_compact_sign, parse_hex_32, to_hex_string};
use o2_sdk::FAST_BRIDGE_TESTNET_URL;
use std::time::{SystemTime, UNIX_EPOCH};

let client = FastBridgeClient::new(FAST_BRIDGE_TESTNET_URL)?;
let request = WithdrawPrepareRequest {
    destination_chain_id: 11155111,
    from_address: fuel_address,
    to: evm_recipient,
    asset_id: full_fuel_asset_id,
    amount: "1000000".into(), // gross Fuel asset base units
};

let prepared = client.prepare_withdraw(&request).await?;
if prepared.fuel_chain_id.parse::<u64>()? != trusted_fuel_chain_id {
    return Err("Unexpected Fuel chain".into());
}

let claims = parse_preparation_proof(&prepared.preparation_proof)?;
if claims.expires_at <= SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() {
    return Err("Prepare again: proof expired".into());
}

// trusted_max_inputs is consensus data not encoded in the transaction.
let inspected = parse_fuel_unsigned_transaction(
    &prepared.unsigned_transaction,
    trusted_fuel_chain_id,
    trusted_max_inputs,
)?;
println!("{inspected:#?}");
// Check contract, route, asset, gross/fee/net amounts, fees, expiry, every input
// owner/asset, and all outputs against request and trusted configuration.
if !approve_withdrawal(&request, &inspected) {
    return Err("Prepared withdrawal does not match intent".into());
}

let signature = fuel_compact_sign(
    &private_key,
    &parse_hex_32(&inspected.transaction_id)?,
)?;
let submitted = client
    .submit_withdraw(&SubmitRequest {
        unsigned_transaction: prepared.unsigned_transaction,
        preparation_proof: prepared.preparation_proof,
        signature: to_hex_string(&signature),
    })
    .await?;
```

Sign the locally computed raw transaction ID without extra hashing. Submit only the exact prepared bytes, exact proof, and compact signature.

## Status

Save `inspected.transaction_id` before submit. Use `client.get_withdraw_status(&inspected.transaction_id)` with bounded backoff. `BridgeError::Api { status: 404, .. }` means not found. A submit timeout is ambiguous; query status before resubmitting. Fuel inclusion and destination delivery are separate states.

The request amount is gross. Inspect `gross_amount`, `bridge_fee`, `net_amount`, and the separate `network_fee.max_fee`. If assets are held by an O2 trading account, first use normal `O2Client::withdraw(...)` to fund the owner Fuel wallet.

Parsed proof claims are unauthenticated display data. Only the proxy verifies the proof HMAC and binding to the exact transaction bytes.
