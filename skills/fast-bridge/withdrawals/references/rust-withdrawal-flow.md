# Rust Fast-Bridge Withdrawal Flow

These snippets are the tested Rust withdrawal path without CLI/env boilerplate.

## Imports And Constants

```rust
use anyhow::{anyhow, bail, Context, Result};
use ethers::{
    signers::{LocalWallet, Signer},
    types::Address as EvmAddress,
};
use fuels::{
    prelude::{abigen as fuels_abigen, Execution, Provider as FuelProvider, Wallet},
    types::{AssetId, ContractId},
};
use o2_sdk::{
    crypto::{parse_hex_32, to_hex_string},
    encoding::{build_actions_signing_bytes, function_selector, u64_be, CallArg, GAS_MAX},
    Network, NetworkConfig, O2Client, SignableWallet, TradeAccountId,
};
use rand::thread_rng;
use reqwest::header::{ACCEPT, CONTENT_TYPE};
use serde::Deserialize;
use serde_json::{json, Value};
use sha2::{Digest, Sha256};

const API_BASE: &str = "https://api.o2.app";
const WS_URL: &str = "wss://api.o2.app/v1/ws";
const FUEL_RPC_URL: &str = "https://mainnet.fuel.network/v1/graphql";
const BASE_CHAIN_ID: u32 = 8453;
const ASSET_REGISTRY_CONTRACT_ID: &str =
    "0x91cfcbef2caad02996cdcb5b897222170e85a91cd2db8f23f07ea7d9ca030c19";
const WRAPPED_ASSETS_MINTER_CONTRACT_ID: &str =
    "0x0f9f509374c2da68997a3a1ad6d85be3f351c2f5f3da4c65bdbfad9b0bb25504";
const GAS_ORACLE_CONTRACT_ID: &str =
    "0x3d20e5a675c5fa1053fba11e176099711ba2f23112e385a7f6e2e759eca84f94";
const GAS_ORACLE_DEPENDENCY_CONTRACT_ID: &str =
    "0x801af4ee92bd9e64ac16b65f490d4cd7dae791662ffbc8f70e20cdb7a6b7fa8c";
const FAST_BRIDGE_ASSET_SYMBOL: &str = "uwUSDC";
const WITHDRAW_SELECTOR_HEX: &str =
    "000000000000002177697468647261775f7669615f666173745f6272696467655f776974685f666565";
```

## GasOracle Binding

Use the bundled full `GasOracle-abi.json`.

```rust
fuels_abigen!(Contract(
    name = "GasOracleContract",
    abi = ".agents/skills/o2-fast-bridge-withdrawals/abis/GasOracle-abi.json"
));
```

## O2 Client And Account

Use the EVM owner wallet that owns the O2 account. The SDK derives owner B256 as the left-padded EVM address and signs with Ethereum `personal_sign`.

```rust
fn build_o2_client() -> O2Client {
    let mut cfg = NetworkConfig::from_network(Network::Mainnet);
    cfg.api_base = API_BASE.into();
    cfg.ws_url = WS_URL.into();
    cfg.faucet_url = None;
    O2Client::with_config(cfg)
}

async fn ensure_trade_account_id<W: SignableWallet>(
    client: &mut O2Client,
    owner_wallet: &W,
) -> Result<TradeAccountId> {
    let account = client.setup_account(owner_wallet).await?;
    account
        .trade_account_id
        .ok_or_else(|| anyhow!("O2 setup_account did not return trade_account_id"))
}
```

## Quote Withdrawal

`FAST_BRIDGE_ASSET_SYMBOL` is `uwUSDC`, not `USDC`. Amounts use 9 decimals.

```rust
async fn quote_withdraw(amount: &str, recipient_address: &str) -> Result<QuoteOutput> {
    let asset_sub_id = sha256_hex(FAST_BRIDGE_ASSET_SYMBOL.as_bytes());
    let wrapped_asset_id = sha256_hex(
        &[
            parse_hex_32(WRAPPED_ASSETS_MINTER_CONTRACT_ID)?.as_slice(),
            parse_hex_32(&asset_sub_id)?.as_slice(),
        ]
        .concat(),
    );
    let desired_net_amount = scale_decimal_9(amount)?;
    let fee_quote = fetch_withdraw_fee(&wrapped_asset_id).await?;
    let gross_debit = desired_net_amount
        .checked_add(fee_quote)
        .ok_or_else(|| anyhow!("withdraw gross debit overflow"))?;

    Ok(QuoteOutput {
        recipient_address: recipient_address.to_owned(),
        asset: FAST_BRIDGE_ASSET_SYMBOL,
        desired_net_amount: amount.to_owned(),
        desired_net_amount_raw: desired_net_amount.to_string(),
        fee_quote_raw: fee_quote.to_string(),
        gross_debit_raw: gross_debit.to_string(),
        asset_sub_id,
        wrapped_asset_id,
    })
}
```

## Fetch Fee

The tested Rust flow uses generated Fuel bindings and includes the GasOracle dependency contract ID for simulation.

```rust
async fn fetch_withdraw_fee(wrapped_asset_id: &str) -> Result<u64> {
    let provider = FuelProvider::connect(FUEL_RPC_URL).await?;
    let mut rng = thread_rng();
    let view_wallet = Wallet::random(&mut rng, provider.clone());
    let contract_id: ContractId = GAS_ORACLE_CONTRACT_ID
        .parse()
        .map_err(|err| anyhow!("invalid gas oracle contract id: {err}"))?;
    let dependency_contract_id: ContractId = GAS_ORACLE_DEPENDENCY_CONTRACT_ID
        .parse()
        .map_err(|err| anyhow!("invalid gas oracle dependency contract id: {err}"))?;
    let asset_id: AssetId = wrapped_asset_id
        .parse()
        .map_err(|err| anyhow!("invalid wrapped asset id: {err}"))?;

    let gas_oracle = GasOracleContract::new(contract_id.into(), view_wallet);
    let call = gas_oracle
        .methods()
        .get_withdrawal_fee(BASE_CHAIN_ID, asset_id)
        .with_contract_ids(&[dependency_contract_id]);
    let mut call = call;
    let response = call.simulate(Execution::state_read_only()).await?;

    Ok(response.value)
}
```

## Fetch Nonce And O2 Chain ID

Use the O2 API chain id from `/v1/markets`, not a raw Fuel provider chain id.

```rust
#[derive(Debug, Deserialize)]
struct MarketsResponse {
    chain_id: String,
}

#[derive(Debug, Deserialize)]
struct AccountResponse {
    nonce: Option<Value>,
    trade_account: Option<NestedNonce>,
    account: Option<NestedNonce>,
}

#[derive(Debug, Deserialize)]
struct NestedNonce {
    nonce: Option<Value>,
}

async fn fetch_nonce(trade_account_id: &str) -> Result<u64> {
    let http = reqwest::Client::new();
    let response = http
        .get(format!("{API_BASE}/v1/accounts?trade_account_id={}", trade_account_id))
        .header(ACCEPT, "application/json")
        .send()
        .await?;
    let status = response.status();
    let payload: AccountResponse = response.json().await?;

    if !status.is_success() {
        bail!("GET /v1/accounts failed: {}", status.as_u16());
    }

    let nonce = payload
        .nonce
        .or_else(|| payload.trade_account.and_then(|inner| inner.nonce))
        .or_else(|| payload.account.and_then(|inner| inner.nonce))
        .ok_or_else(|| anyhow!("Could not read O2 account nonce"))?;

    parse_value_u64(nonce, "nonce")
}

async fn fetch_o2_chain_id() -> Result<u64> {
    let http = reqwest::Client::new();
    let response = http
        .get(format!("{API_BASE}/v1/markets"))
        .header(ACCEPT, "application/json")
        .send()
        .await?;
    let status = response.status();
    let payload: MarketsResponse = response.json().await?;

    if !status.is_success() {
        bail!("GET /v1/markets failed: {}", status.as_u16());
    }

    parse_u64(&payload.chain_id, "chain_id")
}
```

## Build Calldata

Recipient is a normal EVM address for the O2 API, but a 32-byte left-padded EVM address inside AssetRegistry calldata.

```rust
fn build_withdraw_calldata(
    asset_sub_id: &str,
    destination_chain: u32,
    recipient: &str,
    fee_quote: u64,
) -> Result<Vec<u8>> {
    let mut bytes = Vec::with_capacity(76);
    bytes.extend_from_slice(&parse_hex_32(asset_sub_id)?);
    bytes.extend_from_slice(&destination_chain.to_be_bytes());
    bytes.extend_from_slice(&left_pad_evm_address(recipient)?);
    bytes.extend_from_slice(&fee_quote.to_be_bytes());
    Ok(bytes)
}

fn left_pad_evm_address(address: &str) -> Result<[u8; 32]> {
    let addr = EvmAddress::from_str(address).context("Invalid recipient address")?;
    let mut padded = [0u8; 32];
    padded[12..].copy_from_slice(addr.as_bytes());
    Ok(padded)
}
```

## Sign Account Action

`function_selector` is Fuel `selectorBytes`, not the 8-byte method hash.

```rust
let call = CallArg {
    contract_id: parse_hex_32(ASSET_REGISTRY_CONTRACT_ID)?,
    function_selector: hex::decode(WITHDRAW_SELECTOR_HEX)?,
    amount: parse_u64(&quote.gross_debit_raw, "gross debit")?,
    asset_id: parse_hex_32(&quote.wrapped_asset_id)?,
    gas: GAS_MAX,
    call_data: Some(build_withdraw_calldata(
        &quote.asset_sub_id,
        BASE_CHAIN_ID,
        &recipient_address,
        parse_u64(&quote.fee_quote_raw, "fee quote")?,
    )?),
};

let action_bytes = build_actions_signing_bytes(nonce, &[call]);
let mut signing_bytes = Vec::new();
signing_bytes.extend_from_slice(&u64_be(nonce));
signing_bytes.extend_from_slice(&u64_be(o2_chain_id));
signing_bytes.extend_from_slice(&function_selector("call_contracts"));
signing_bytes.extend_from_slice(&action_bytes[8..]);

let signature = owner_wallet.personal_sign(&signing_bytes)?;
let signature_hex = to_hex_string(&signature);
```

## Submit O2 Action

```rust
let body = json!({
    "actions": [{
        "WithdrawViaFastBridgeWithFee": {
            "amount": quote.gross_debit_raw,
            "fee_quote": quote.fee_quote_raw,
            "asset": {
                "sub_id": quote.asset_sub_id,
                "universal": quote.wrapped_asset_id,
            },
            "recipient": {
                "Evm": {
                    "chain_id": BASE_CHAIN_ID.to_string(),
                    "recipient": { "address": recipient_address }
                }
            }
        }
    }],
    "signature": { "Secp256k1": signature_hex },
    "nonce": nonce.to_string(),
    "trade_account_id": trade_account_id,
    "variable_outputs": 1,
    "contracts": [ASSET_REGISTRY_CONTRACT_ID],
});

let response = reqwest::Client::new()
    .post(format!("{API_BASE}/v1/accounts/actions"))
    .header(CONTENT_TYPE, "application/json")
    .header(ACCEPT, "application/json")
    .header("O2-Owner-Id", owner_b256)
    .json(&body)
    .send()
    .await?;

let status = response.status();
let response_json: Value = response.json().await.unwrap_or_else(|_| json!(null));
if !status.is_success() {
    bail!(
        "POST /v1/accounts/actions failed: {} {}",
        status.as_u16(),
        response_json
    );
}
```

## Utility Helpers

```rust
fn sha256_hex(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    format!("0x{}", hex::encode(hasher.finalize()))
}

fn parse_u64(value: &str, label: &str) -> Result<u64> {
    value
        .parse::<u64>()
        .with_context(|| format!("invalid {label}: {value}"))
}

fn parse_value_u64(value: Value, label: &str) -> Result<u64> {
    match value {
        Value::String(s) => parse_u64(&s, label),
        Value::Number(n) => n
            .as_u64()
            .ok_or_else(|| anyhow!("invalid {label}: {n}")),
        other => bail!("invalid {label}: {other}"),
    }
}

fn scale_decimal_9(input: &str) -> Result<u64> {
    let trimmed = input.trim();
    if trimmed.is_empty() {
        bail!("Amount cannot be empty");
    }
    if trimmed.starts_with('-') {
        bail!("Amount cannot be negative");
    }

    let mut parts = trimmed.split('.');
    let whole = parts.next().unwrap_or("0");
    let frac = parts.next().unwrap_or("");
    if parts.next().is_some() {
        bail!("Invalid decimal amount: {trimmed}");
    }
    if !whole.chars().all(|c| c.is_ascii_digit()) || !frac.chars().all(|c| c.is_ascii_digit()) {
        bail!("Invalid decimal amount: {trimmed}");
    }
    if frac.len() > 9 {
        bail!("Amount has more than 9 decimal places: {trimmed}");
    }

    let whole_value: u128 = if whole.is_empty() { 0 } else { whole.parse()? };
    let mut frac_text = frac.to_string();
    while frac_text.len() < 9 {
        frac_text.push('0');
    }
    let frac_value: u128 = if frac_text.is_empty() {
        0
    } else {
        frac_text.parse()?
    };

    let scaled = whole_value
        .checked_mul(1_000_000_000u128)
        .and_then(|value| value.checked_add(frac_value))
        .ok_or_else(|| anyhow!("Amount overflows u64"))?;

    u64::try_from(scaled).context("Amount overflows u64")
}
```
