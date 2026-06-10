---
name: o2-sdk-rust
description: "Rust reference for using the O2 SDK: installation, client setup, Fuel and EVM EOA wallets, trading account setup, session creation, market lookup, order placement, batch actions, balances, orders, nonce recovery, stock withdrawals, and WebSocket streams. Use when a user asks how to integrate O2 in Rust, create sessions, trade on O2, place or cancel orders, settle balances, inspect balances/orders, stream updates, or use the o2-sdk crate."
---

# O2 SDK Rust

Use this skill for Rust integrations with the `o2-sdk` crate. Keep this focused on normal O2 SDK usage: setup, sessions, trading, account state, market data, streams, and stock withdrawals.

This skill is grounded in the Rust `o2-sdk` public API around `o2-sdk = "0.2.0"` and Rust 1.75+.

For deeper REST, signing, byte layout, endpoint, and error-code details, use `../../o2-reference/SKILL.md`.

## Install

```toml
[dependencies]
o2-sdk = "0.2.0"
tokio = { version = "1", features = ["full"] }
```

## Identity Model

O2 has three identities. Do not collapse them.

```text
Owner wallet      controls account setup, session creation, and SDK withdrawals
trade_account_id  Fuel contract account that holds funds
Session key       short-lived trading key for place/cancel/settle actions
```

Rules:

- Owner wallet signs account setup, session setup, and SDK withdrawals.
- Session signs order actions.
- Session key cannot withdraw.
- `trade_account_id` is a Fuel contract ID, not the owner address.

## Client Setup

```rust
use o2_sdk::{Network, NetworkConfig, O2Client};

let mut client = O2Client::new(Network::Mainnet);
```

Custom config:

```rust
let mut cfg = NetworkConfig::from_network(Network::Mainnet);
cfg.api_base = "https://api.o2.app".into();
cfg.ws_url = "wss://api.o2.app/v1/ws".into();
cfg.faucet_url = None;

let mut client = O2Client::with_config(cfg);
```

Networks:

```text
Network::Mainnet
Network::Testnet
Network::Devnet
```

## Owner EOA Wallet

Use the SDK wallet helpers for EOA private keys.

```rust
let fuel_wallet = client.load_wallet("0x...")?;
let evm_wallet = client.load_evm_wallet("0x...")?;
```

Generate wallets:

```rust
let fuel_wallet = client.generate_wallet()?;
let evm_wallet = client.generate_evm_wallet()?;
```

Fuel EOA and EVM EOA owners both implement `SignableWallet`, but they do not derive the same owner B256 and they do not sign the same digest.

```text
Fuel EOA:
  helper:      client.load_wallet(...)
  owner B256:  Fuel b256 address derived from the Fuel key
  signing:     Fuel personal_sign format

EVM EOA:
  helper:      client.load_evm_wallet(...)
  owner B256:  20-byte EVM address left-padded to 32 bytes
  signing:     Ethereum personal_sign format
```

For O2 account setup, session creation, and withdrawals, always use the same owner wallet type that created the O2 account.

## Account Setup

Create or load the trading account before trading.

```rust
let account = client.setup_account(&wallet).await?;
let trade_account_id = account
    .trade_account_id
    .ok_or_else(|| o2_sdk::O2Error::Other("missing trade account id".into()))?;
```

`setup_account(&wallet)` is idempotent. On testnet/devnet it also attempts setup helpers such as faucet/whitelist behavior. On mainnet there is no faucet.

If you need an explicit testnet/devnet top-up:

```rust
let _ = client.top_up_from_faucet(&wallet).await?;
```

## Market Lookup

Fetch market metadata before creating orders. Market metadata controls contract IDs, assets, decimals, precision, and minimum order checks.

```rust
let market_symbol = "fFUEL/fUSDC";
let market = client.get_market(market_symbol).await?;
let price = market.price("0.02")?;
let quantity = market.quantity("50")?;
```

Prefer `market.price(...)` and `market.quantity(...)` when building orders. Direct strings are accepted by some methods, but market-bound values catch precision mistakes earlier.

## Session Creation

Create a session when the app needs to place/cancel/settle orders.

```rust
let mut session = client
    .create_session(
        &wallet,
        &[market_symbol],
        std::time::Duration::from_secs(30 * 24 * 3600),
    )
    .await?;

println!("trade_account_id={}", session.trade_account_id);
println!("session_expiry={}", session.expiry);
```

The returned `Session` is local mutable state. Pass `&mut session` to trading methods so the SDK can update nonce state.

If you need an absolute expiry timestamp:

```rust
let session = client
    .create_session_until(&wallet, &[market_symbol], expiry_unix_secs)
    .await?;
```

## Trading Actions

Place a normal order:

```rust
use o2_sdk::{OrderType, Side};

let response = client
    .create_order(
        &mut session,
        market_symbol,
        Side::Buy,
        market.price("0.02")?,
        market.quantity("50")?,
        OrderType::Spot,
        true, // settle_first
        true, // collect_orders
    )
    .await?;

if !response.is_success() {
    return Err(o2_sdk::O2Error::Other(
        response.message.unwrap_or_else(|| "O2 order failed".into()),
    ));
}

let order_id = response
    .orders
    .as_ref()
    .and_then(|orders| orders.first())
    .map(|order| order.order_id.clone());
```

Cancel and settle:

```rust
if let Some(order_id) = &order_id {
    client.cancel_order(&mut session, order_id, market_symbol).await?;
}

client.cancel_all_orders(&mut session, market_symbol).await?;
client.settle_balance(&mut session, market_symbol).await?;
```

Batch actions with the builder:

```rust
let actions = client
    .actions_for(market_symbol)
    .await?
    .settle_balance()
    .create_order(
        Side::Buy,
        market.price("0.02")?,
        market.quantity("50")?,
        OrderType::Spot,
    )
    .build()?;

let response = client
    .batch_actions(&mut session, market_symbol, actions, true)
    .await?;
```

Batch rules:

- Use `batch_actions(...)` for one market.
- Use `batch_actions_multi(...)` for multiple markets.
- Keep each request at 5 actions or fewer.
- Use `settle_balance` before balance-sensitive creates.

## Balances And Orders

Read account state with `trade_account_id`, not owner B256.

```rust
let balances = client.get_balances(&trade_account_id).await?;
for (symbol, balance) in &balances {
    println!(
        "{symbol}: available={}, locked={}, unlocked={}",
        balance.trading_account_balance,
        balance.total_locked,
        balance.total_unlocked,
    );
}

let orders = client
    .get_orders(market_symbol, &trade_account_id, Some(true), 20, None, None)
    .await?;
```

`get_balances(trade_account_id)` is aggregated across trading-account and market contracts. A successful `settle_balance(...)` may not change aggregate totals, but it moves filled proceeds back into the trading account.

## SDK Withdrawal

Use `client.withdraw(...)` for the Rust SDK's normal owner-signed withdrawal from an O2 trading account to a Fuel identity address.

```rust
let response = client
    .withdraw(
        &wallet,
        &session,
        &asset_id,
        "100000000", // raw amount string expected by the SDK method
        None,        // defaults to owner B256
    )
    .await?;
```

Rules:

- Owner wallet signs the withdrawal.
- Session key cannot withdraw.
- `asset_id` is a hex asset ID.
- `amount` is a raw integer string for the asset amount expected by this method.
- Optional `to` must be a 32-byte hex Fuel identity address.

## Market Data

```rust
let depth = client.get_depth(market_symbol, 1, Some(50)).await?;
let trades = client.get_trades(market_symbol, 50, None, None).await?;
let ticker = client.get_ticker(market_symbol).await?;
let bars = client.get_bars(market_symbol, "1m", from_ts, to_ts).await?;
```

Depth precision is `1..=18`; `1` is most precise. Bar timestamps are milliseconds.

## Streams

Use `tokio_stream::StreamExt` to consume streams.

```rust
use o2_sdk::Identity;
use tokio_stream::StreamExt;

let identity = Identity::ContractId(trade_account_id.to_string());

let mut balances = client.stream_balances(&[identity.clone()]).await?;
while let Some(update) = balances.next().await {
    let update = update?;
    println!("balance entries={}", update.balance.len());
}
```

Market streams use market IDs:

```rust
let mut depth = client.stream_depth(&market.market_id, 1).await?;
let mut trades = client.stream_trades(&market.market_id).await?;
```

Account-scoped streams use identities:

```rust
let ids = [Identity::ContractId(trade_account_id.to_string())];
let mut orders = client.stream_orders(&ids).await?;
let mut nonce = client.stream_nonce(&ids).await?;
```

## Nonce And Recovery

Common recovery rules:

- If the session expired, create a new session with the owner wallet.
- If action submission fails, the SDK increments local nonce and attempts to refresh it.
- If retrying manually after an uncertain error, call `client.refresh_nonce(&mut session).await?`.
- If a cancel targets an already-filled or missing order, remove that cancel and rebuild the batch.
- Keep `collect_orders = true` when the caller needs order IDs.

Example:

```rust
match client
    .create_order(
        &mut session,
        market_symbol,
        Side::Buy,
        market.price("0.02")?,
        market.quantity("50")?,
        OrderType::Spot,
        true,
        true,
    )
    .await
{
    Ok(resp) if resp.is_success() => {}
    Ok(resp) => {
        return Err(o2_sdk::O2Error::Other(
            resp.message.unwrap_or_else(|| "O2 action failed".into()),
        ));
    }
    Err(err) => {
        client.refresh_nonce(&mut session).await?;
        return Err(err);
    }
}
```

## Sharp Rules

- Use `trade_account_id` for balances, orders, and account state.
- Use owner wallet for account setup, session setup, and SDK withdrawals.
- Use mutable `Session` for trading actions.
- Session actions use the session key; owner wallet is not used for place/cancel/settle after session creation.
- Use `market.price(...)` and `market.quantity(...)` to avoid decimal/precision mistakes.
- Keep action batches at 5 actions or fewer.
- Do not use owner B256 where `trade_account_id` is required.
- Do not use a Fuel wallet helper for an EVM owner key or an EVM wallet helper for a Fuel owner key.
