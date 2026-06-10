# o2 Exchange Skills

o2 is a fully on-chain order book exchange built on the Fuel Network. Every order, fill, and cancellation is a blockchain transaction. This file tells you how to build trading integrations on o2.

This skills file works together with the **o2 API Reference** (`o2-reference.md`). For Claude Code, place both files in `.claude/rules/` so they are auto-loaded. For other AI tools, refer to their documentation for loading custom context files. The reference contains signing byte layouts, endpoint catalog, error codes, ABI encoding rules, and complete Python/TypeScript code examples. This skills file tells you _what to do_ and _when_. The reference tells you _how_.

Full reference: [https://docs.o2.app/o2-reference.md](https://docs.o2.app/o2-reference.md)

---

## Integration Workflow

Follow these steps in order. Do not skip any step.

### 1. Set Up Environment

Pick a network and set the base URL. See **Reference Section 2** for full URL tables (WebSocket, Fuel RPC, Faucet).

| Network | API Base URL |
|---------|-------------|
| Devnet | `https://api.devnet.o2.app` |
| Testnet | `https://api.testnet.o2.app` |
| Mainnet | `https://api.o2.app` |

All REST endpoints are prefixed with `/v1/`. All request/response bodies are JSON. See **Reference Section 2** for required cryptographic dependencies by language.

### 2. Generate a Wallet

Generate a secp256k1 keypair. Derive the B256 address (32-byte SHA-256 hash of the uncompressed public key, displayed as `0x`-prefixed 64-char hex). EVM wallets are also supported (keccak256 derivation, zero-padded to B256). See **Reference Section 4.1** for Python reference implementation.

### 3. Fund Your Wallet

On devnet/testnet/sandbox: `POST https://fuel-o2-faucet.vercel.app/api/<network>/mint-v2` with `{"address":"0x<your_b256>"}`. The faucet is not available on mainnet. See **Reference Section 4.2** for details.

### 4. Create a Trading Account

`POST /v1/accounts` with `{"identity":{"Address":"0x<your_b256>"}}`. Save the returned `trade_account_id`. One owner wallet maps to one trading account. See **Reference Section 4.3** for the full request/response format.

### 5. Deposit Funds

On devnet/testnet/sandbox, the faucet mints directly to your trading account. On mainnet, send an on-chain Fuel transfer to the `trade_account_id` contract. See **Reference Section 4.4** for both options.

### 6. Fetch Market Info

`GET /v1/markets`. Save the `market_id`, `contract_id`, asset IDs, and decimal configs. You need these for every order. See **Reference Section 4.5** for the full response structure and how to read decimal configs.

### 7. Whitelist (Testnet/Devnet Only)

`POST /analytics/v1/whitelist` with `{"tradeAccount":"0x<trade_account_id>"}`. Required before placing orders. Without this, orders fail with `TraderNotWhiteListed`. See **Reference Section 4.5b**.

### 8. Create a Session

Generate a second keypair (the session key). Construct `set_session` signing bytes, sign with the owner wallet using `personalSign` format, and send `PUT /v1/session`. The session key can place/cancel orders but cannot withdraw funds. The owner key only signs once here. o2 sponsors gas for all session-based actions.

See **Reference Section 3** for the three-layer identity model (owner, trading account, session) and **Reference Section 4.6** for the signing byte construction, signature format, and complete Python reference implementation.

### 9. Fetch Nonce

`GET /v1/accounts?trade_account_id=0x<your_trade_account>`. Read the `nonce` field. The nonce is a replay-protection counter that increments on-chain with every action, even on reverts.

### 10. Place Orders

Construct `POST /v1/session/actions` with `CreateOrder` actions. Sign the payload with the session key using raw `sha256` (no prefix). Always prepend a `SettleBalance` action to move filled proceeds into your available balance before placing new orders.

See **Reference Section 4.7** for the signing byte format, payload structure, and complete Python code example.

### 11. Monitor and Manage

- Check orders: `GET /v1/orders?market_id=0x...&contract=0x<trade_account_id>`. See **Reference Section 4.8**.
- Cancel orders: `CancelOrder` action via `POST /v1/session/actions`. See **Reference Section 4.9**.
- Check balances: `GET /v1/balance`. See **Reference Section 4.10**.
- Withdraw: `POST /v1/accounts/withdraw`. See **Reference Section 4.11**.

---

## Order Type Decision Guide

See **Reference Section 6** for the full JSON format of each order type.

| When you want to... | Use |
|---------------------|-----|
| Standard limit order (partial fills OK, rests on book) | `"Spot"` |
| Execute immediately at best price, no resting | `"Market"` |
| Guaranteed price with time priority | `{"Limit":["<price>","<timestamp>"]}` |
| Fill the entire amount or nothing at all | `"FillOrKill"` |
| Only provide liquidity (never take) | `"PostOnly"` |
| Market order with slippage protection | `{"BoundedMarket":{"max_price":"...","min_price":"..."}}` |

---

## Key Rules and Gotchas

1. **Always settle before creating orders.** Include `SettleBalance` as the first action in every `session/actions` request.

2. **Nonce increments on reverts.** Track the nonce locally. Increment after every `session/actions` call regardless of success. Re-fetch from the API if you get out of sync.

3. **Max 5 actions per request.** A typical market maker cycle: cancel(buy) + cancel(sell) + settle + create(buy) + create(sell) = 5 actions.

4. **Cancelling a filled order reverts the entire batch.** If an order was fully filled between cycles, the cancel action reverts the whole transaction (including new placements). Use WebSocket `subscribe_orders` to track fills in real-time and skip cancels for filled orders. See **Reference Section 5** for WebSocket subscription formats.

5. **Signing conventions differ by context:**
   - Owner signing (session creation): `personalSign` format, `sha256("\x19Fuel Signed Message:\n" + len + message)`. See **Reference Section 4.6**.
   - Session signing (orders): raw, `sha256(message)` with `prehash: false`. See **Reference Section 4.7**.

6. **`@noble/secp256k1` v3 critical change:** v3 defaults to `prehash: true` (double-hashes). You must pass `prehash: false` since this guide's flow already pre-hashes. Incorrect prehash silently recovers to the wrong address (error 4000). See **Reference Section 2** for library setup.

7. **Decimal scaling.** Prices and quantities are u64 integers. Multiply by `10^decimals` and truncate to `max_precision` digits. Get these from the market info response. See **Reference Section 10** (Appendix: Decimal Handling Rules).

8. **Use `collect_orders: true`** in `session/actions` to get order IDs back for tracking and cancellation.

---

## Common Patterns

See **Reference Section 9** for full pseudocode and detailed explanations.

### Market Maker

Each cycle: fetch external reference price, cancel stale orders, settle balance, place symmetric buy/sell around the spread, sleep, repeat. All actions go in a single `session/actions` request for atomic execution. See **Reference Section 9 (Market Maker Skeleton)** for the complete algorithm.

### Simple Taker

Subscribe to WebSocket depth updates, wait for target price, submit `BoundedMarket` order with slippage bounds, repeat. See **Reference Section 9 (Simple Taker / Sniper)**.

### Error Recovery

See **Reference Section 8** for the full error code table and handling recommendations.

- **Nonce mismatch (5006):** Re-fetch nonce from API.
- **Rate limited (1003):** Exponential backoff.
- **Session expired:** Create a new session (requires owner key). Track `expiry` timestamp and renew proactively.
- **Batch revert from stale cancel:** Remove the failed cancel from the action list and resubmit.

---

## Quick Reference: API Reference Sections

| Section | Content |
|---------|---------|
| **1** | Quick Start (TL;DR): 11-step checklist |
| **2** | Environment Setup: URLs, dependencies by language |
| **3** | Authentication Architecture: owner, trading account, session model |
| **4** | Step-by-Step Walkthrough: wallet, account, session, orders, management |
| **5** | WebSocket Real-Time Data: depth, orders, balances, nonce subscriptions |
| **6** | Order Types Reference: JSON formats for all 6 order types |
| **7** | Complete REST API Reference: all endpoints by category |
| **8** | Error Handling: error codes, response formats, troubleshooting |
| **9** | Common Bot Patterns: market maker, taker, best practices |
| **10** | Appendix: ABI encoding, decimal handling, fees, rate limits |

---

## Reference Links

- [o2 API Reference](https://docs.o2.app/o2-reference.md): the full knowledge base this skills file references
- [MCP Server](https://docs.o2.app/mcp-server): trade via natural language with AI assistants (no code needed)
- [Developer Quick Start (TypeScript)](https://docs.o2.app/developer-quick-start): working code walkthrough
- [Developer Quick Start (Python)](https://docs.o2.app/developer-quick-start-python): working code walkthrough
