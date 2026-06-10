# O2 Exchange Trading Bot Integration Guide

O2 is a fully on-chain order book exchange on Fuel. Every order placement, fill, and cancellation executes on-chain. This reference preserves the full integration context (workflow, signatures, byte layout rules, endpoints, errors, bot patterns) in a shorter operational format.

Primary docs:
- [o2 API Reference](https://docs.o2.app/api-endpoints-reference.html)
- [Developer Quick Start (TypeScript)](https://docs.o2.app/developer-quick-start)
- [Developer Quick Start (Python)](https://docs.o2.app/developer-quick-start-python)
- [MCP Server](https://docs.o2.app/mcp-server)
- [Detailed reference file](https://docs.o2.app/o2-reference-detailed.md)
---

## 1. Quick Start (TL;DR)

1. Generate owner secp256k1 keypair and derive owner address (B256).
2. Fund owner wallet (testnet/devnet/sandbox faucet; no mainnet faucet).
3. Create trading account: `POST /v1/accounts` -> persist `trade_account_id`.
4. Deposit funds (faucet mint on non-mainnet or on-chain transfer on mainnet).
5. Fetch market metadata: `GET /v1/markets` -> market IDs, contract IDs, assets, decimals.
6. Whitelist account: `POST /analytics/v1/whitelist` (required before trading in test/dev environments).
7. Generate session keypair; create session via `PUT /v1/session` signed by owner wallet.
8. Fetch account nonce: `GET /v1/accounts?trade_account_id=...`.
9. Place orders via `POST /v1/session/actions` signed by session wallet.
10. Monitor and manage orders/balances (`/v1/orders`, `/v1/order`, `/v1/balance`).
11. Cancel, settle, withdraw as needed (`session/actions`, `/v1/accounts/withdraw`).

Critical rules:
- Session creation signing: owner key + `personalSign` digest flow.
- Session actions signing: session key + raw `sha256(message)` (no personalSign prefix).
- Nonce increments on all session action tx attempts, including reverts.
- Respect max 5 actions per request.
- Include `SettleBalance` in balance-sensitive cycles before new order placements.

---

## 2. Environment Setup

### Devnet URLs

| Component | URL |
|---|---|
| API | `https://api.devnet.o2.app` |
| WS | `wss://api.devnet.o2.app/v1/ws` |
| Fuel RPC | `https://devnet.fuel.network/v1/graphql` |
| Faucet | `https://fuel-o2-faucet.vercel.app/api/devnet/mint-v2` |

### Testnet URLs

| Component | URL |
|---|---|
| API | `https://api.testnet.o2.app` |
| WS | `wss://api.testnet.o2.app/v1/ws` |
| Fuel RPC | `https://testnet.fuel.network/v1/graphql` |
| Faucet | `https://fuel-o2-faucet.vercel.app/api/testnet/mint-v2` |

Mainnet API: `https://api.o2.app` (no faucet).  
Sandbox faucet path uses `/api/sandbox/mint-v2`.

All REST endpoints are under `/v1` and use JSON.

### Required Dependencies

Required capabilities:
- secp256k1 signing and compact signature handling
- SHA-256
- UTF-8 encoding
- u64 big-endian byte encoding
- BigInt/u64-safe arithmetic

Common library options:
- Python: `coincurve`/`ecdsa`, `hashlib`
- Rust: `secp256k1`, `sha2`
- JS/TS: `@noble/secp256k1`, `@noble/hashes/sha2.js`
- Go: `btcec`, `crypto/sha256`

If owner wallet is EVM-based, include keccak256 support.

JS/TS caveat:
- `@noble/secp256k1` v3 defaults to `prehash: true`. In O2 flows pass `prehash: false` to avoid double hashing and recovered-address mismatch (`4000`).

---

## 3. Authentication Architecture

O2 identity model:
- **Owner Wallet**: controls account and withdrawal capability.
- **Trading Account** (`trade_account_id`): on-chain contract that holds funds and executes trading calls.
- **Session Wallet**: delegated short-lived key for trading actions.

Owner responsibilities:
- account creation
- session creation/rotation
- privileged account operations (withdraw/upgrade/actions)

Session capabilities:
- create/cancel orders
- settle balances
- cannot withdraw or transfer ownership funds

Address support:
- Fuel-native owner addresses (B256)
- EVM owner addresses represented in B256 form (zero-padded to 32 bytes)

---

## 4. Step-by-Step Walkthrough

### 4.1 Generate a Wallet

Generate owner secp256k1 keypair and derive B256 identity.

Persist:
- `owner_private_key`
- `owner_b256_address`

### 4.2 Fund Your Wallet via Faucet (Testnet/Devnet/Sandbox Only)

Mint endpoint:
```json
POST https://fuel-o2-faucet.vercel.app/api/<network>/mint-v2
{"address":"0x<target_b256_or_trade_account>"}
```

Use owner address first; later you can mint directly to `trade_account_id`.

#### Mint to a wallet address

Send owner B256 in `address`.

#### Mint directly to a trading account contract

After account creation, fund `trade_account_id` directly in test/dev/sandbox.

### 4.3 Create a Trading Account

```json
POST /v1/accounts
{"identity":{"Address":"0x<owner_b256>"}}
```

Persist:
- `trade_account_id`
- deploy tx metadata if returned

### 4.4 Deposit Funds

#### Option A: Mint Directly to the Trading Account (Devnet/Testnet)

Use faucet and set `address = trade_account_id`.

#### Option B: On-Chain Transfer (Mainnet / Production)

Transfer asset to account contract (`trade_account_id`) on Fuel.

### 4.5 Fetch Market Info

`GET /v1/markets`

For each market cache:
- `market_id`
- order book `contract_id`
- `base_asset` / `quote_asset`
- decimal precision configuration
- top-level registry IDs when needed by advanced actions (for example referer registration)

### 4.5b Whitelist Your Trading Account (Required Before Trading)

```json
POST /analytics/v1/whitelist
{"tradeAccount":"0x<trade_account_id>"}
```

Whitelist missing/misconfigured state can block order placement.

### 4.6 Create a Trading Session

Session request binds a session wallet to selected contract IDs until expiry.

Request skeleton:
```json
PUT /v1/session
{
  "session": {"Address":"0x<session_b256>"},
  "expiry": "1737504000",
  "contract_ids": ["0x<market_contract_id>"],
  "signature": {"Secp256k1":"0x<owner_signature>"}
}
```

Required header:
- `O2-Owner-Id: 0x<owner_b256>`

Critical details:
- owner key signs session creation
- use personalSign-style digest prefixing
- session scope is restricted to allowlisted contracts
- unsupported signature schemes return `4000`

#### Step 1: Generate a Session Wallet

Persist:
- `session_private_key`
- `session_b256_address`

#### Step 2: Get the Current Nonce

`GET /v1/accounts?trade_account_id=0x...` -> read `nonce`.

#### Step 3: Construct Signing Bytes

Session signing bytes must deterministically include:
- function selector for `set_session`
- owner identity
- session identity
- expiry
- allowed contract IDs
- nonce

Any byte mismatch causes signature recovery mismatch.

#### Step 4: Sign with Owner Wallet (personalSign)

Use digest form:
- `sha256("\x19Fuel Signed Message:\n" + len(message) + message)`

#### Step 4b: EVM Owner Signing (Alternative)

For EVM owners:
- keep EVM signing/recovery behavior consistent
- still transmit identity as B256 format

#### Step 5: Send the Session Request

Submit request with owner header + owner signature.

### 4.7 Place an Order

Endpoint:
- `POST /v1/session/actions`

Limits:
- max 5 actions per request
- max 5 market groups per request

Canonical action ordering:
1. stale cancels (optional)
2. `SettleBalance`
3. new creates

#### Signing Process for Session Actions

Session action signing rules:
- digest: raw `sha256(message)` (no personalSign prefix)
- signer: session key, not owner key
- signature must cover exact serialized payload bytes and nonce

Minimal payload shape:
```json
{
  "nonce": "123",
  "actions": [
    {
      "market_id": "0x<market_id>",
      "actions": [
        {"SettleBalance":{"to":{"ContractId":"0x<trade_account_id>"}}},
        {"CreateOrder":{"side":"Buy","price":"1000000","quantity":"1000000","order_type":"Spot"}}
      ]
    }
  ],
  "collect_orders": true,
  "signature": {"Secp256k1":"0x<session_signature>"}
}
```

Action mapping:
- `CreateOrder` -> `OrderBook.create_order`
- `CancelOrder` -> `OrderBook.cancel_order`
- `SettleBalance` -> `OrderBook.settle_balance`
- `RegisterReferer` -> `TradeAccountRegistry.register_referer`

Byte/encoding rules:
- Fuel function selector is not keccak; use `u64_be(len(name)) + utf8(name)`
- all u64 numeric fields are 8-byte big-endian
- gas field typically set to `u64::MAX` client-side; server can override
- `OrderType` enum bytes must match on-chain discriminant+payload layout
- side determines forwarded asset/amount for create flows

OrderType variants used in `OrderArgs`:
- `Limit`: variant 0 with payload `price` + `timestamp`
- `Spot`: variant 1
- `FillOrKill`: variant 2
- `PostOnly`: variant 3
- `Market`: variant 4
- `BoundedMarket`: variant 5 with `max_price` + `min_price`

#### Complete Request Example

Production requests must have:
- fresh nonce
- correctly scaled integer amounts
- exact market/contract IDs
- signature generated over exact message bytes
- `collect_orders: true` when you need returned order IDs for lifecycle tracking

### 4.8 Check Order Status

#### Get All Orders

`GET /v1/orders?market_id=...&contract=...&direction=desc&count=20`

Supports:
- `is_open=true`
- pagination via `start_timestamp` + `start_order_id`

#### Get a Single Order

`GET /v1/order?market_id=...&order_id=...`

### 4.9 Cancel an Order

Use `CancelOrder` action via `POST /v1/session/actions`.

Batch revert risk:
- canceling an already-filled order may revert the full batch.
- mitigate via live order state tracking and stale-order filtering.

### 4.10 Check Balances

`GET /v1/balance?asset_id=...&contract=...`

Endpoint selection rules:
- provide either `address` or `contract` as supported by query contract.

### 4.11 Withdraw Funds

`POST /v1/accounts/withdraw` (owner-authenticated).

Typical fields:
- `asset_id`
- `amount`
- destination `to` identity

Session key cannot withdraw.

---

## 5. WebSocket Real-Time Data

### Connection

- Devnet: `wss://api.devnet.o2.app/v1/ws`
- Testnet: `wss://api.testnet.o2.app/v1/ws`

### Subscribe to Order Book Depth

Topic: depth updates keyed by `market_id` and precision settings.

### Subscribe to Your Orders

Topic: order lifecycle events keyed by account identity (contract/trade account).

### Subscribe to Trades

Topic: market trade stream keyed by `market_id`.

### Subscribe to Balances

Topic: balance updates keyed by address/contract identity.

### Subscribe to Nonce Updates

Topic: account nonce changes for replay protection tracking.

### Unsubscribe

Use matching `unsubscribe` message with same topic and identity.

### Identity Format

Common identity keys:
- `market_id`
- `contract` / `trade_account_id`
- optional stream-specific cursor/window fields

Example:
```json
{"action":"subscribe","topic":"orders","identity":{"market_id":"0x...","contract":"0x..."}}
```

Operational guidance:
- Use WS to avoid stale cancels and nonce drift.
- Keep REST polling as fallback for reconciliation.

---

## 6. Order Types Reference

Supported order types:
- `"Spot"`: resting order, partial fills allowed.
- `"Market"`: immediate execution at available prices.
- `{"Limit":["<price>","<timestamp>"]}`: explicit price + timestamped limit variant.
- `"FillOrKill"`: full fill immediately or cancel.
- `"PostOnly"`: maker-only placement.
- `{"BoundedMarket":{"max_price":"...","min_price":"..."}}`: market behavior with bound constraints.

### JSON Format for Each Order Type

```json
"Spot"
"Market"
"FillOrKill"
"PostOnly"
{"Limit":["1000000","1730000000"]}
{"BoundedMarket":{"max_price":"1100000","min_price":"900000"}}
```

---

## 7. Complete REST API Reference

All endpoints are JSON and rooted under `/v1` unless otherwise noted.

### Market Data
- `GET /v1/markets`
- `GET /v1/markets/summary?market_id=...`
- `GET /v1/markets/ticker?market_id=...`

### Order Book & Depth
- `GET /v1/depth?market_id=...&precision=...`
- Valid precision values include powers of ten from `10` up to `1000000000`.

### Trading Data
- `GET /v1/trades?market_id=...&direction=...&count=...`
- `GET /v1/trades_by_account?market_id=...&contract=...&direction=...&count=...`
- `GET /v1/bars?market_id=...&from=...&to=...&resolution=...`

Constraints:
- trades count max 50
- `direction` and `count` required for trades
- bars max 5000
- bar resolutions: `1s`, `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w`, `1M`

### Account & Balance
- `POST /v1/accounts`
- `GET /v1/accounts?owner=...`
- `GET /v1/accounts?owner_contract=...`
- `GET /v1/accounts?trade_account_id=...`
- `GET /v1/balance?asset_id=...&address=...`
- `GET /v1/balance?asset_id=...&contract=...`

Lookup rules:
- account lookup expects exactly one of `owner`, `owner_contract`, `trade_account_id`
- balance lookup uses either `address` or `contract` per endpoint contract

### Order Management
- `GET /v1/orders?market_id=...&contract=...&direction=...&count=...`
- `GET /v1/order?market_id=...&order_id=...`

Order query constraints:
- max 200 per orders request
- optional `is_open=true`
- pagination by `start_timestamp` + `start_order_id`

### Session Management
- `PUT /v1/session`
- `POST /v1/session/actions`

### Account Operations
- `POST /v1/accounts/actions`
- `POST /v1/accounts/upgrade`
- `POST /v1/accounts/withdraw`

### Analytics Endpoints
- `POST /analytics/v1/whitelist`
- `GET /analytics/v1/referral/code-info?code=...`

### Aggregator Endpoints

Uses `market_pair` notation (for example `FUEL_USDC`) instead of hex market IDs.

- `GET /v1/aggregated/assets`
- `GET /v1/aggregated/orderbook?market_pair=...&depth=...&level=...`
- `GET /v1/aggregated/summary`
- `GET /v1/aggregated/ticker`
- `GET /v1/aggregated/trades?market_pair=...`

### System
- `GET /v1/ws` (upgrade target for WebSocket connections)

Protected endpoint header:
- `O2-Owner-Id: 0x<owner_b256>` required for session/account operation routes.

---

## 8. Error Handling

### Error Response Format

Common structured response:
```json
{"code":2000,"message":"..."}
```

`POST /v1/session/actions` can also return on-chain revert style:
```json
{"message":"Revert(...)","reason":"...","receipts":[...]}
```

Success/failure check:
- success when `tx_id` exists
- failure when `message` exists and `tx_id` is absent

### Complete Error Code Table

General (1xxx):
- `1000 InternalError`
- `1001 InvalidRequest`
- `1002 ParseError`
- `1003 RateLimitExceeded`
- `1004 GeoRestricted`

Market (2xxx):
- `2000 MarketNotFound`
- `2001 MarketPaused`
- `2002 MarketAlreadyExists`

Order (3xxx):
- `3000 OrderNotFound`
- `3001 OrderNotActive`
- `3002 InvalidOrderParams`

Account/Session (4xxx):
- `4000 InvalidSignature`
- `4001 InvalidSession`
- `4002 AccountNotFound`
- `4003 WhitelistNotConfigured`

Trade (5xxx):
- `5000 TradeNotFound`
- `5001 InvalidTradeCount`

Subscription/WebSocket (6xxx):
- `6000 AlreadySubscribed`
- `6001 TooManySubscriptions`
- `6002 SubscriptionError`

Validation (7xxx):
- `7000 InvalidAmount`
- `7001 InvalidTimeRange`
- `7002 InvalidPagination`
- `7003 NoActionsProvided`
- `7004 TooManyActions`

Block/Events (8xxx):
- `8000 BlockNotFound`
- `8001 EventsNotFound`

### Common Error Scenarios and Solutions

`4000` on session creation:
- wrong signing mode (must be personalSign flow)
- wrong key (must be owner key)
- double hashing from signer misconfiguration

`4000` on session actions:
- wrong signing mode (must be raw sha256 signing)
- wrong key (must be session key)
- byte mismatch from nonce/action/ordering differences

`7004`:
- request exceeds 5 actions; split batches

`4001`:
- session missing/expired; rotate via `PUT /v1/session`

`1003`:
- apply backoff + jitter and request pacing

---

## 9. Common Bot Patterns

### Testnet Market Conditions

Test/dev books can be thin or one-sided. Check depth before placing crossing orders to avoid unexpected fills.

### Market Maker Skeleton

Loop:
1. fetch external reference
2. compute buy/sell quotes from spread policy
3. scale to chain integers with precision limits
4. fetch latest nonce
5. build atomic batch (cancel stale, settle, create new orders)
6. sign with session key and submit
7. record returned order IDs for next cycle

### Simple Taker / Sniper

Loop:
1. subscribe to WS depth/trades
2. wait for trigger condition
3. submit bounded market or targeted spot order
4. verify outcome and refresh state

### Order Management Best Practices

- Settle before creating orders in capital-constrained loops.
- Track nonce locally and increment after each action tx attempt.
- Use WS order updates to reduce stale-cancel reverts.
- Keep fallback reconciliation via REST.
- Proactively rotate sessions before expiry.

---

## 10. Appendix

### Fuel ABI Encoding Rules

Core encoding rules:
- selector bytes: `u64_be(len(name)) + utf8(name)`
- integers: u64 big-endian
- identities: discriminant + value bytes
- enums: discriminant + active variant payload

For order/session action bytes, the encoded layout must exactly match on-chain ABI expectations.

### Decimal Handling Rules

Use market metadata for:
- token decimals
- max precision constraints

Conversion pattern:
- `scaled = floor(human * 10^decimals)`
- apply precision truncation according to market limits
- ensure scaled outputs fit expected u64 ranges

### Fee Structure

Fee schedules vary by market/program and can differ by environment. Read current live data and avoid hardcoded assumptions.

### Rate Limit Summary

Implement:
- exponential backoff
- jitter
- burst throttling
- safe retry policy for idempotent paths

### Session Expiry Guidelines

- Track expiry when session is created.
- Renew proactively for long-running bots.
- Keep owner signing path available for immediate recovery.
