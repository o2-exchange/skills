# O2 Skills

Skills for helping agents guide O2 traders through SDK usage and fast-bridge funding/withdrawal flows.

## Reference

### O2 SDK

- **[o2-sdk-typescript](./skills/o2-sdk/typescript/SKILL.md)** — TypeScript reference for O2 SDK setup, owner signers, trading accounts, sessions, markets, balances, orders, nonce recovery, and account actions.
- **[o2-sdk-rust](./skills/o2-sdk/rust/SKILL.md)** — Rust reference for O2 SDK setup, owner wallets, trading accounts, sessions, markets, balances, orders, nonce recovery, stock withdrawals, and streams.

### Fast Bridge

- **[o2-fast-bridge-deposits](./skills/fast-bridge/deposits/SKILL.md)** — Explain EVM source-chain deposits into O2/Fuel, including O2 `trade_account_id` contract recipients, `recipientIsContract`, source-token decimals, caps, whitelists, and relayer timing.
- **[o2-fast-bridge-withdrawals](./skills/fast-bridge/withdrawals/SKILL.md)** — Explain O2 trading-account withdrawals to EVM chains, including how to replicate a `withdrawToChain`-style helper until native SDK support ships, owner signer vs session key, `trade_account_id`, fees, 9-decimal amounts, recipient encoding, and unwrap vs bridge.

### References

- **[o2-skills.md](./references/o2-skills.md)** — O2 trading integration workflow.
- **[o2-reference.md](./references/o2-reference.md)** — O2 API/session/signing/order reference.

### Fast Bridge ABIs

- **[fast-bridge-abis/](./fast-bridge-abis)** — Minimal ABI fragments used by the examples:
  `Messenger.json`, `IERC20Metadata.json`, `IERC20Permit.json`, `AssetRegistry-abi.json`, and `GasOracle-abi.json`.
