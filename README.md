# O2 Skills

Skills for helping agents guide O2 traders through SDK usage and fast-bridge funding/withdrawal flows.

## Install

```bash
npx skills add o2-exchange/skills
```

## Skills Included

### Core O2 Docs

- **[o2-reference](./skills/o2-reference/SKILL.md)**
  Shared O2 API, session, and signing reference material used by the SDK and bridge skills.

### O2 SDK Guides

- **[o2-sdk-typescript](./skills/o2-sdk/typescript/SKILL.md)**
  Use this when building with the O2 SDK in TypeScript: account setup, owner signers, sessions, markets, balances, orders, nonce recovery, and account actions.

- **[o2-sdk-rust](./skills/o2-sdk/rust/SKILL.md)**
  Use this when building with the O2 SDK in Rust: owner wallets, trading accounts, sessions, balances, orders, withdrawals, and streams.

### Fast-Bridge Guides

- **[o2-fast-bridge-deposits](./skills/fast-bridge/deposits/SKILL.md)**
  Use this for EVM-to-O2 deposits: `trade_account_id` recipients, `recipientIsContract`, token decimals, caps, whitelists, and relayer timing.

- **[o2-fast-bridge-withdrawals](./skills/fast-bridge/withdrawals/SKILL.md)**
  Use this for O2-to-EVM withdrawals: owner signer vs session key, `trade_account_id`, fee quoting, 9-decimal amounts, recipient encoding, and unwrap vs bridge.

## Notes

- The shared O2 reference docs live inside `o2-reference`, so the SDK skills do not repeat the same API and signing details.
- Fast-bridge ABI files are bundled with the bridge skills that use them:
  `skills/fast-bridge/deposits/abis/` and `skills/fast-bridge/withdrawals/abis/`.
