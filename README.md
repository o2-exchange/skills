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
| [o2-sdk-typescript](./skills/o2-sdk/typescript/SKILL.md) | Building with the TypeScript SDK: accounts, sessions, trading, standard withdrawals, and Fast Bridge proxy access. |
| [o2-sdk-python](./skills/o2-sdk/python/SKILL.md) | Building with the Python SDK: accounts, sessions, trading, standard withdrawals, and Fast Bridge proxy access. |
| [o2-sdk-rust](./skills/o2-sdk/rust/SKILL.md) | Building with the Rust SDK: accounts, sessions, trading, standard withdrawals, streams, and Fast Bridge proxy access. |
| [o2-fast-bridge-deposits](./skills/fast-bridge/deposits/SKILL.md) | SDK-first EVM-to-Fuel deposits through the Fast Bridge proxy, with TypeScript, Python, and Rust flows. |
| [o2-fast-bridge-withdrawals](./skills/fast-bridge/withdrawals/SKILL.md) | SDK-first withdrawals from a funded Fuel wallet to EVM through the Fast Bridge proxy, including inspection, fees, signing, and status. |

## Usage Examples

#### TypeScript Market Making Bot

```text
I am building a TypeScript market-making bot on O2. Use o2-sdk-typescript to set up the owner signer, create a session key, fetch market metadata, and place post-only orders.
```

#### Base Native ETH Funding Flow

```text
I run a bot from Base and want to bridge native ETH to a Fuel wallet. Use o2-fast-bridge-deposits and show the FastBridgeClient prepare, inspect, sign, submit, and status flow.
```

#### Cross-chain multi-asset Flow

```text
I need to bridge USDC from Ethereum to Fuel, then later move O2 profits through my Fuel wallet to a Base address. Use both Fast Bridge skills and explain the O2-account-to-wallet step before the Fuel-to-EVM withdrawal.
```

## Notes

- The shared O2 reference docs live inside `o2-reference`, so the SDK skills do not repeat the same API and signing details.
- Fast Bridge ABI files remain under `skills/fast-bridge/*/abis/` for advanced
  proxy-bypass work; normal integrations should use the SDK proxy clients.
