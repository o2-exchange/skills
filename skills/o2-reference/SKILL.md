---
name: o2-reference
description: "Reference skill for O2 public docs: trading integration workflow, REST API behavior, session signing, account-action signing, byte layout rules, endpoint payloads, error codes, and identifier rules. Use when another O2 skill or user request needs deeper O2 API/session/signing details beyond SDK quick usage."
---

# O2 Reference

Use this skill when an O2 SDK or bridge task needs deeper protocol/API details.

Read only the file needed for the task:

```text
./references/o2-skills.md     O2 trading integration workflow
./references/o2-reference.md  O2 API, session, signing, byte layout, endpoints, errors
```

Keep answers grounded in these references when discussing:

- session creation signing
- session action signing
- account-action signing
- owner B256 vs `trade_account_id`
- API payload shapes
- error codes and recovery
- byte layout rules
