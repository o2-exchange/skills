---
name: o2-sdk-python
description: "Python reference for using the O2 SDK: installation, client setup, Fuel and EVM EOA wallets, trading account setup, session creation, market lookup, order placement, batch actions, balances, orders, nonce recovery, stock withdrawals, and async lifecycle. Use when a user asks how to integrate O2 in Python, create sessions, trade on O2, place or cancel orders, settle balances, inspect balances/orders, or use the o2-sdk package."
---

# O2 SDK Python

Use this skill for Python integrations with `o2-sdk`. Keep this focused on normal SDK usage: setup, sessions, trading, account state, market data, and stock withdrawals.

This skill is grounded in the public Python SDK (`pip install o2-sdk`). For deeper REST, signing, byte layout, endpoint, and error-code details, use `../../o2-reference/SKILL.md`.

## Install

```bash
pip install o2-sdk
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

```python
from o2_sdk import Network, NetworkConfig, O2Client

client = O2Client(network=Network.MAINNET)
```

Custom config:

```python
client = O2Client(
    custom_config=NetworkConfig(
        api_base="https://api.o2.app",
        ws_url="wss://api.o2.app/v1/ws",
        fuel_rpc="https://mainnet.fuel.network/v1/graphql",
        faucet_url=None,
    )
)
```

## Owner EOA Wallet

Use the SDK wallet helpers for EOA private keys.

```python
fuel_wallet = client.load_wallet("0x...")
evm_wallet = client.load_evm_wallet("0x...")
```

Generate wallets:

```python
fuel_wallet = client.generate_wallet()
evm_wallet = client.generate_evm_wallet()
```

Fuel EOA and EVM EOA owners do not derive the same owner B256 and do not sign the same digest.

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

```python
account = await client.setup_account(owner)
trade_account_id = account.trade_account_id
```

`setup_account(owner)` is idempotent. On testnet/devnet it also attempts faucet and whitelist flows. On mainnet there is no faucet.

Explicit testnet/devnet top-up:

```python
await client.top_up_from_faucet(owner)
```

## Market Lookup

Fetch market metadata before creating orders.

```python
markets = await client.get_markets()
market = await client.get_market("ETH/USDC")
pair = market.pair
```

Keep the returned market object when possible. It carries pair, market ID, precision, and scaling helpers.

## Session Creation

Create a session when the app needs to place, cancel, or settle orders.

```python
session = await client.create_session(
    owner=owner,
    markets=[market.pair],
    expiry_days=30,
)
```

Session creation is owner-signed. After that, order actions are session-signed by the SDK.

## Trading Actions

Place a normal order:

```python
from o2_sdk import OrderSide, OrderType

result = await client.create_order(
    market=market.pair,
    side=OrderSide.BUY,
    price="1",
    quantity="5",
    order_type=OrderType.SPOT,
    session=session,
    settle_first=True,
    collect_orders=True,
)
```

Cancel and settle:

```python
await client.cancel_order(
    order_id=result.orders[0].order_id,
    market=market.pair,
    session=session,
)

await client.settle_balance(market.pair, session=session)
```

Batch actions with the builder:

```python
batch = (
    client.actions_for(market.pair)
    .settle_balance()
    .create_order(OrderSide.BUY, "0.1", "50")
    .create_order(OrderSide.SELL, "2", "5", OrderType.POST_ONLY)
    .build()
)

result = await client.batch_actions([batch], collect_orders=True, session=session)
```

`batch_actions(...)` requires a session, uses the session key for signing, and refreshes nonce after failed submissions.

## Balances And Orders

Read balances from the trading account contract:

```python
balances = await client.get_balances(trade_account_id)
orders = await client.get_orders(trade_account_id, market.pair, is_open=True)
```

Use `trade_account_id`, not owner `b256_address`, when the question is about funds or open orders inside O2.

## Stock SDK Withdrawals

Use the stock SDK withdrawal helper only for standard O2 withdrawals, not fast-bridge EVM withdrawal helpers.

```python
result = await client.withdraw(
    owner=owner,
    asset="USDC",
    amount=100.0,
)
```

Rules:

- `amount` is human-readable.
- `asset` can be a symbol or asset ID.
- destination defaults to the owner wallet address.
- owner wallet signs the withdrawal.
- session key cannot withdraw.

Fast-bridge EVM withdrawals are not yet a native Python SDK helper. Use the fast-bridge withdrawal skill for that path.

## Nonce Recovery And Cleanup

If a session action fails and nonce state is uncertain:

```python
await client.refresh_nonce(session)
```

Close the client when done:

```python
await client.close()
```
