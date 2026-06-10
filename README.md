# O2 Skills

Skills for helping agents guide O2 traders through SDK usage and fast-bridge funding/withdrawal flows.

## Reference

### O2 Reference

- **[o2-reference](./skills/o2-reference/SKILL.md)** — Shared O2 public docs for API/session/signing details used by the SDK and bridge skills.

### O2 SDK

- **[o2-sdk-typescript](./skills/o2-sdk/typescript/SKILL.md)** — TypeScript reference for O2 SDK setup, owner signers, trading accounts, sessions, markets, balances, orders, nonce recovery, and account actions.
- **[o2-sdk-rust](./skills/o2-sdk/rust/SKILL.md)** — Rust reference for O2 SDK setup, owner wallets, trading accounts, sessions, markets, balances, orders, nonce recovery, stock withdrawals, and streams.

### Fast Bridge

- **[o2-fast-bridge-deposits](./skills/fast-bridge/deposits/SKILL.md)** — Explain EVM source-chain deposits into O2/Fuel, including O2 `trade_account_id` contract recipients, `recipientIsContract`, source-token decimals, caps, whitelists, and relayer timing.
- **[o2-fast-bridge-withdrawals](./skills/fast-bridge/withdrawals/SKILL.md)** — Explain O2 trading-account withdrawals to EVM chains, including how to replicate a `withdrawToChain`-style helper until native SDK support ships, owner signer vs session key, `trade_account_id`, fees, 9-decimal amounts, recipient encoding, and unwrap vs bridge.

### References

- O2 public reference docs are bundled inside the `o2-reference` skill so SDK skills do not duplicate them.

### Fast Bridge ABIs

- Bridge ABI fragments live inside the bridge skill folders that use them:
  `skills/fast-bridge/deposits/abis/` and `skills/fast-bridge/withdrawals/abis/`.
