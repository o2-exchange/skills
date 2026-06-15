# Rust Fast-Bridge Deposit Flow

Use this reference when depositing Base mainnet USDC into an O2 trading account from an EVM owner wallet.

## Install Dependencies

Add the crates used by the reference flow:

```bash
cargo add anyhow ethers o2-sdk tokio
```

If you prefer a manifest snippet instead of `cargo add`, use:

```toml
[dependencies]
anyhow = "1"
ethers = { version = "2", default-features = false, features = ["abigen", "rustls"] }
o2-sdk = "0.2"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

## Before You Start

Keep these rules straight before writing code:

- The bridge recipient for an O2 deposit is the O2 `trade_account_id`.
- Set `recipientIsContract = true` for the O2 trading account.
- Deposit amounts use the source-chain token decimals, not Fuel's 9-decimal wrapped format.
- For Base mainnet USDC, `0.5` becomes `500000` raw units because USDC has 6 decimals.
- The Base transaction confirming does not mean O2 crediting is already visible. Relayer processing is still asynchronous.

## Imports And Constants

```rust
use std::sync::Arc;

use anyhow::{Context, Result};
use ethers::{
    contract::abigen,
    middleware::SignerMiddleware,
    providers::{Http, Provider},
    signers::{LocalWallet, Signer},
    types::{Address as EvmAddress, U256},
    utils::parse_units,
};
use o2_sdk::{Network, O2Client};

const BASE_RPC_URL: &str = "https://mainnet.base.org";
const BASE_CHAIN_ID: u32 = 8453;
const MESSENGER_ADDRESS: &str = "0x2B1c1E133F832EFB1e168dE6102304B03C4ba653";
const BASE_USDC_ADDRESS: &str = "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913";
```

## Messenger And Token Bindings

```rust
abigen!(
    BaseUsdc,
    r#"[
        function decimals() view returns (uint8)
        function allowance(address owner, address spender) view returns (uint256)
        function approve(address spender, uint256 value) returns (bool)
    ]"#,
);

abigen!(
    Messenger,
    r#"[
        function deposit(bytes32 to, address token, uint256 value, bool recipientIsContract)
    ]"#,
);
```

## Deposit Base USDC Into O2

```rust
async fn deposit_usdc_to_o2(private_key: &str, amount_text: &str) -> Result<()> {
    let mut client = O2Client::new(Network::Mainnet);
    let owner_wallet = client.load_evm_wallet(private_key)?;
    let account = client.setup_account(&owner_wallet).await?;
    let trade_account_id = account.trade_account_id.context("missing trade account id")?;

    let provider = Provider::<Http>::try_from(BASE_RPC_URL)?;
    let wallet = private_key.parse::<LocalWallet>()?.with_chain_id(BASE_CHAIN_ID as u64);
    let signer = Arc::new(SignerMiddleware::new(provider, wallet));

    let base_usdc = BASE_USDC_ADDRESS.parse::<EvmAddress>()?;
    let messenger_address = MESSENGER_ADDRESS.parse::<EvmAddress>()?;
    let token = BaseUsdc::new(base_usdc, signer.clone());
    let messenger = Messenger::new(messenger_address, signer.clone());

    let decimals = token.decimals().call().await?;
    let parsed_amount: U256 = parse_units(amount_text, decimals as usize)
        .context("invalid deposit amount")?
        .into();
    let allowance = token
        .allowance(signer.address(), messenger_address)
        .call()
        .await?;

    if allowance < parsed_amount {
        token.approve(messenger_address, U256::MAX).send().await?.await?;
    }

    let mut recipient = [0u8; 32];
    recipient.copy_from_slice(trade_account_id.as_ref());

    messenger
        .deposit(recipient, base_usdc, parsed_amount, true)
        .send()
        .await?
        .await?;

    Ok(())
}
```

## What Matters

- Use the O2 trading account as the bridge recipient.
- Set `recipientIsContract = true` for the O2 trading account.
- Deposit amounts use source-token EVM decimals.
- The Messenger call takes the source token address, not a Fuel asset id.
- Once the Base transaction confirms, O2 crediting still depends on relayer processing.

## Common Mistakes

- Using the owner address instead of `trade_account_id` as the deposit recipient.
- Setting `recipientIsContract = false` for an O2 trading-account deposit.
- Scaling the amount to 9 decimals instead of the source token decimals.
- Treating Base confirmation as proof that the O2 balance is already credited.
