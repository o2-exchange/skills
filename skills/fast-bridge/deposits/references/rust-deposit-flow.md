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
    DepositPrepareRequest, DepositSubmitResponse, EvmDepositInspection,
    FastBridgeClient, SubmitRequest,
};
use o2_sdk::crypto::{fuel_compact_sign, parse_hex_32, to_hex_string};
use std::{error::Error, time::{SystemTime, UNIX_EPOCH}};

async fn deposit(
    client: &FastBridgeClient,
    request: &DepositPrepareRequest,
    private_key: &[u8; 32],
    approve: impl FnOnce(&DepositPrepareRequest, &EvmDepositInspection) -> bool,
) -> Result<DepositSubmitResponse, Box<dyn Error>> {
    let prepared = client.prepare_deposit(request).await?;
    let claims = parse_preparation_proof(&prepared.preparation_proof)?;
    let inspected = parse_evm_unsigned_transaction(&prepared.unsigned_transaction)?;
    println!("{inspected:#?}");

    // Compare chain ID, nonce, Messenger, method, recipient/type, token,
    // amount/value, gas, and fee caps with request and trusted configuration.
    if !approve(request, &inspected) {
        return Err("Prepared deposit does not match intent".into());
    }

    // Recheck immediately before signing/submission in case approval took time.
    if claims.expires_at <= SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() {
        return Err("Prepare again: proof expired".into());
    }

    let mut signature =
        fuel_compact_sign(private_key, &parse_hex_32(&inspected.signing_digest)?)?.to_vec();
    signature.push(27 + (signature[32] >> 7));
    signature[32] &= 0x7f;

    Ok(client
        .submit_deposit(&SubmitRequest {
            unsigned_transaction: prepared.unsigned_transaction,
            preparation_proof: prepared.preparation_proof,
            signature: to_hex_string(&signature),
        })
        .await?)
}
```

For a wallet recipient use `RecipientType::Address`; for an O2 `trade_account_id` or other Fuel contract use `RecipientType::Contract`. The request amount is an integer string in Fuel asset base units.

Submit the exact unsigned transaction and proof returned by prepare. The separate signature is 65-byte EVM `r || s || v`; do not submit a signed transaction envelope.

## Status

Use `client.get_deposit_status(submitted.source_chain_id, &submitted.evm_tx_hash)` with bounded backoff. Match `BridgeError::Api { status: 404, .. }` as not found. Stop when `source.status` becomes `Confirmed` or `Reverted`; the proxy's Fuel-side status remains `Unavailable` today. Confirm final Fuel delivery separately by querying the recipient wallet or contract balance after the expected relay delay. A submit timeout is ambiguous, so reconcile before resubmitting.

## Native ETH, ERC-20, And Permit

The proxy selects `depositETH`, `deposit`, or `depositWithPermit`. ERC-20 requires a Messenger allowance or an EIP-2612 `DepositPermit`. The permit is a separately signed token approval, not the EVM transaction signature.

Parsed proof claims are display-only; only the proxy authenticates the proof and exact transaction binding.
