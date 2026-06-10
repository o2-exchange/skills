Skills are organized into domain folders under `skills/`.

- `o2-reference/` — shared O2 API/session/signing reference docs
- `o2-sdk/` — TypeScript and Rust O2 SDK usage
- `fast-bridge/` — O2 fast-bridge deposits and withdrawals

Every skill must have:

- a reference in the top-level `README.md`
- an entry in `.claude-plugin/plugin.json`
- a domain entry in `skills/README.md`

Keep this repo minimal. Prefer crisp, source-grounded skills focused on O2 reference docs, SDK usage, and fast-bridge flows. Avoid adding irrelevant buckets, setup flows, or extra root files.
