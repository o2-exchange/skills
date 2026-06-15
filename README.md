# O2 Skills

Skills for helping agents guide O2 traders through SDK usage and fast-bridge funding/withdrawal flows.

## Install

```bash
npx skills add o2-exchange/skills
```

## Skills Included

| Skill | Use This For |
| --- | --- |
| [o2-reference](./skills/o2-reference/SKILL.md) | Shared O2 API, session, and signing reference material used by the SDK and bridge skills. |
| [o2-sdk-typescript](./skills/o2-sdk/typescript/SKILL.md) | Building with the O2 SDK in TypeScript: account setup, owner signers, sessions, markets, balances, orders, nonce recovery, and account actions. |
| [o2-sdk-python](./skills/o2-sdk/python/SKILL.md) | Building with the O2 SDK in Python: owner wallets, trading accounts, sessions, balances, orders, withdrawals, and batch actions. |
| [o2-sdk-rust](./skills/o2-sdk/rust/SKILL.md) | Building with the O2 SDK in Rust: owner wallets, trading accounts, sessions, balances, orders, withdrawals, and streams. |
| [o2-fast-bridge-deposits](./skills/fast-bridge/deposits/SKILL.md) | Moving funds from an EVM chain into an O2 trading account, with TypeScript, Python, and Rust reference flows for deposits. |
| [o2-fast-bridge-withdrawals](./skills/fast-bridge/withdrawals/SKILL.md) | Moving funds from an O2 trading account back to an EVM chain, with TypeScript, Python, and Rust reference flows for fee quotes and withdrawals. |

## Usage Examples

### TypeScript Market Making Bot

```text
I am building a TypeScript market-making bot on O2. Use o2-sdk-typescript to set up the owner signer, create a session key, fetch market metadata, and place post-only orders.
```

### Base Native ETH Funding Flow

```text
I run a bot from Base and want to fund its O2 trading account with native ETH before rotating into trading collateral on O2. Use o2-fast-bridge-deposits and show the depositETH flow for my O2 trading account (trade_account_id).
```

### Cross-chain multi-asset Flow

```text
I manage multiple trading bots and need to bridge USDC from Ethereum mainnet into one trading account, then withdraw profits later to a Base EVM address. Use o2-fast-bridge-deposits for the Ethereum USDC funding flow and o2-fast-bridge-withdrawals for the owner-signed USDC withdrawal flow.
```

## Notes

- The shared O2 reference docs live inside `o2-reference`, so the SDK skills do not repeat the same API and signing details.
- Fast-bridge ABI files are bundled with the bridge skills that use them:
  `skills/fast-bridge/deposits/abis/` and `skills/fast-bridge/withdrawals/abis/`.
