# Rust Fast Bridge Deposit

Use `o2-sdk` 0.4 or newer. The SDK contains the proxy client, parser, and compact signing helper.

```toml
[dependencies]
o2-sdk = "0.4"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

```rust
use o2_sdk::bridge::{
    parse_evm_unsigned_transaction, parse_preparation_proof,
    DepositPrepareRequest, FastBridgeClient, RecipientType, SubmitRequest,
    FAST_BRIDGE_TESTNET_URL,
};
use o2_sdk::crypto::{fuel_compact_sign, parse_hex_32, to_hex_string};
use std::time::{SystemTime, UNIX_EPOCH};

let client = FastBridgeClient::new(FAST_BRIDGE_TESTNET_URL)?;
let request = DepositPrepareRequest {
    source_chain_id: 11155111,
    from_address: evm_address,
    to: fuel_recipient,
    to_type: RecipientType::Address,
    asset_id: full_fuel_asset_id,
    amount: "1000000".into(), // integer Fuel asset base units
    permit: None,
};

let prepared = client.prepare_deposit(&request).await?;
let claims = parse_preparation_proof(&prepared.preparation_proof)?;
if claims.expires_at <= SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() {
    return Err("Prepare again: proof expired".into());
}

let inspected = parse_evm_unsigned_transaction(&prepared.unsigned_transaction)?;
println!("{inspected:#?}");
// Compare chain_id, nonce, messenger_address, method, recipient/type, token,
// amount/value, gas and fee caps with request and independently trusted config.
if !approve_deposit(&request, &inspected) {
    return Err("Prepared deposit does not match intent".into());
}

// Sign the locally derived raw EIP-1559 digest, without personal-sign hashing.
let mut signature =
    fuel_compact_sign(&private_key, &parse_hex_32(&inspected.signing_digest)?)?.to_vec();
signature.push(27 + (signature[32] >> 7));
signature[32] &= 0x7f;

let submitted = client
    .submit_deposit(&SubmitRequest {
        unsigned_transaction: prepared.unsigned_transaction,
        preparation_proof: prepared.preparation_proof,
        signature: to_hex_string(&signature),
    })
    .await?;
```

Submit the exact unsigned transaction and proof returned by prepare. The separate signature is 65-byte EVM `r || s || v`; do not submit a signed transaction envelope.

## Status

Use `client.get_deposit_status(submitted.source_chain_id, &submitted.evm_tx_hash)` with bounded backoff. Match `BridgeError::Api { status: 404, .. }` as not found rather than treating it as a fabricated pending state. A submit timeout is ambiguous; reconcile status before resubmitting.

## Native ETH, ERC-20, And Permit

The proxy selects `depositETH`, `deposit`, or `depositWithPermit`. ERC-20 requires a Messenger allowance or an EIP-2612 `DepositPermit`. The permit is a separately signed token approval, not the EVM transaction signature.

Parsed proof claims (`version`, `key_id`, `expires_at`, `signer`) are unauthenticated. Only the proxy verifies the HMAC and binding to the exact operation and transaction bytes.
