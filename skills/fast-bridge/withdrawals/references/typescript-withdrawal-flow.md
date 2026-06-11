# TypeScript Fast-Bridge Withdrawal Flow

These snippets are the tested withdrawal path without CLI/env boilerplate.

Setup the Fuel read contract and reusable imports:

```ts
import { createHash } from "node:crypto";
import { parseUnits, zeroPadValue } from "ethers";
import {
  Contract as FuelContract,
  Provider,
  Wallet as FuelWallet,
  getMintedAssetId,
} from "fuels";
import GasOracleAbi from "./abis/GasOracle-abi.json";

const fuelProvider = new Provider(fuelRpcUrl);
const viewWallet = FuelWallet.fromAddress(ownerSigner.b256Address, fuelProvider);
const gasOracle = new FuelContract(gasOracleContractId, GasOracleAbi, viewWallet);
```

Resolve the bridge asset. User-facing `USDC` becomes Fuel/O2 bridge asset `uwUSDC`.

```ts
const bridgeAssetSymbol = normalizeBridgeAssetSymbol(asset); // "USDC" -> "uwUSDC"
const assetSubId = sha256Hex(Buffer.from(bridgeAssetSymbol, "utf8"));
const wrappedAssetId = getMintedAssetId(wrappedAssetsMinterContractId, assetSubId);
```

Quote the bridge fee from `GasOracle`, then compute the gross debit. The user receives roughly `desiredNetAmount`; O2 debits `desiredNetAmount + feeQuote`.

```ts
const feeQuote = await gasOracle.functions
  .get_withdrawal_fee(destinationChainId, { bits: wrappedAssetId })
  .get()
  .then((r) => asBigInt(r.value));

const desiredNetAmount = parseUnits(amountText, 9);
const grossDebit = desiredNetAmount + feeQuote;
```

Fetch the O2 nonce and O2 signing chain id. Do not replace `o2ChainId` with a raw provider chain id.

```ts
const nonce = await fetchTradeAccountNonce(apiBase, tradeAccountId);
const o2ChainId = await fetchO2ChainId(apiBase);
```

Encode the AssetRegistry call data manually. This is the 76-byte payload for `withdraw_via_fast_bridge_with_fee(sub_id, destination_chain, recipient, fee_quote)`.

```ts
const paddedRecipient = zeroPadValue(destinationEvmAddress, 32);

const callData = concatBytes(
  hexToBytes(assetSubId),
  u32BE(destinationChainId),
  hexToBytes(paddedRecipient),
  u64BE(feeQuote),
);
```

Build and owner-sign the O2 account-action payload bytes. `withdrawFunctionSelector` is Fuel `selectorBytes`, not the 8-byte method hash.

```ts
const withdrawFunctionSelector = actionSelector("withdraw_via_fast_bridge_with_fee");
const signingBytes = concatBytes(
  u64BE(nonce),
  u64BE(o2ChainId),
  actionSelector("call_contracts"),
  buildActionsSigningBytes(nonce, [
    {
      contractId: hexToBytes(assetRegistryContractId),
      functionSelector: withdrawFunctionSelector,
      amount: grossDebit,
      assetId: hexToBytes(wrappedAssetId),
      gas: GAS_MAX,
      callData,
    },
  ]).slice(8),
);

const signatureHex = normalizeSignatureHex(
  await ownerSigner.personalSign(signingBytes),
);
```

If generated TradeAccount bindings are available, simulate the same call before submitting the API action. The standalone flow above is still the source of truth for the signed payload bytes.

```ts
await tradeAccount.functions
  .call_contracts(
    { Secp256k1: { bits: Array.from(hexToBytes(signatureHex)) } },
    [
      {
        contract_id: { bits: assetRegistryContractId },
        function_selector: withdrawFunctionSelector,
        call_params: {
          coins: grossDebit.toString(),
          asset_id: { bits: wrappedAssetId },
          gas: GAS_MAX.toString(),
        },
        call_data: callData,
      },
    ],
  )
  .get();
```

Submit the O2 account action directly:

```ts
const payload = {
  actions: [
    {
      WithdrawViaFastBridgeWithFee: {
        amount: grossDebit.toString(),
        fee_quote: feeQuote.toString(),
        asset: {
          sub_id: assetSubId,
          universal: wrappedAssetId,
        },
        recipient: {
          Evm: {
            chain_id: destinationChainId.toString(),
            recipient: {
              address: destinationEvmAddress.toLowerCase(),
            },
          },
        },
      },
    },
  ],
  signature: { Secp256k1: signatureHex },
  nonce: nonce.toString(),
  trade_account_id: tradeAccountId,
  variable_outputs: 1,
  contracts: [assetRegistryContractId],
};

const response = await fetch(`${apiBase}/v1/accounts/actions`, {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "O2-Owner-Id": ownerSigner.b256Address,
  },
  body: JSON.stringify(payload),
});

if (!response.ok) {
  throw new Error(`O2 withdrawal action failed (${response.status}): ${await response.text()}`);
}
```

Helper: constants and bridge asset symbol normalization.

```ts
const GAS_MAX = (1n << 64n) - 1n;

function normalizeBridgeAssetSymbol(value: string) {
  const normalized = value.trim();
  const upper = normalized.toUpperCase();
  if (upper === "USDC" || upper === "UWUSDC") return "uwUSDC";
  if (upper === "ETH" || upper === "UWETH") return "uwETH";
  if (upper === "FUEL" || upper === "UWFUEL") return "uwFUEL";
  if (/^uw[A-Za-z0-9]+$/.test(normalized)) return normalized;
  throw new Error(`Unsupported bridge withdrawal asset: ${value}`);
}
```

Helper: fetch nonce and O2 signing chain id.

```ts
async function fetchTradeAccountNonce(apiBase: string, tradeAccountId: string) {
  const url = new URL(`${apiBase}/v1/accounts`);
  url.searchParams.set("trade_account_id", tradeAccountId);
  const response = await fetch(url);
  const body = await response.text();
  if (!response.ok) throw new Error(`Failed to fetch nonce: ${body}`);

  const parsed = JSON.parse(body) as {
    nonce?: string | number;
    trade_account?: { nonce?: string | number } | null;
  };
  const nonce = parsed.nonce ?? parsed.trade_account?.nonce;
  if (nonce === undefined) throw new Error(`Account response did not include nonce: ${body}`);
  return BigInt(nonce);
}

async function fetchO2ChainId(apiBase: string) {
  const response = await fetch(`${apiBase}/v1/markets`);
  const body = await response.text();
  if (!response.ok) throw new Error(`Failed to fetch markets: ${body}`);

  const parsed = JSON.parse(body) as { chain_id?: string | number };
  if (parsed.chain_id === undefined) {
    throw new Error(`Markets response did not include chain_id: ${body}`);
  }
  return BigInt(parsed.chain_id);
}
```

Helper: selector bytes and action signing bytes.

```ts
function actionSelector(name: string) {
  const nameBytes = Buffer.from(name, "utf8");
  return concatBytes(u64BE(BigInt(nameBytes.length)), Uint8Array.from(nameBytes));
}

function buildActionsSigningBytes(
  nonce: bigint,
  calls: Array<{
    contractId: Uint8Array;
    functionSelector: Uint8Array;
    amount: bigint;
    assetId: Uint8Array;
    gas: bigint;
    callData?: Uint8Array;
  }>,
) {
  return concatBytes(
    u64BE(nonce),
    u64BE(BigInt(calls.length)),
    ...calls.map((call) =>
      concatBytes(
        call.contractId,
        u64BE(BigInt(call.functionSelector.length)),
        call.functionSelector,
        u64BE(call.amount),
        call.assetId,
        u64BE(call.gas),
        encodeOptionalBytes(call.callData),
      ),
    ),
  );
}

function encodeOptionalBytes(value?: Uint8Array) {
  return value
    ? concatBytes(u64BE(1n), u64BE(BigInt(value.length)), value)
    : u64BE(0n);
}
```

Helper: byte primitives.

```ts
function concatBytes(...parts: Uint8Array[]) {
  const out = new Uint8Array(parts.reduce((n, p) => n + p.length, 0));
  let offset = 0;
  for (const part of parts) {
    out.set(part, offset);
    offset += part.length;
  }
  return out;
}

function u32BE(value: number) {
  const buffer = Buffer.alloc(4);
  buffer.writeUInt32BE(value);
  return Uint8Array.from(buffer);
}

function u64BE(value: bigint) {
  const buffer = Buffer.alloc(8);
  buffer.writeBigUInt64BE(value);
  return Uint8Array.from(buffer);
}

function hexToBytes(value: string) {
  const hex = value.startsWith("0x") ? value.slice(2) : value;
  return Uint8Array.from(Buffer.from(hex, "hex"));
}

function sha256Hex(data: Uint8Array | Buffer) {
  return `0x${createHash("sha256").update(data).digest("hex")}`;
}
```

Helper: normalize API/SDK return types.

```ts
function asBigInt(value: unknown) {
  return BigInt(String(value));
}

function normalizeSignatureHex(signature: unknown) {
  if (typeof signature === "string") {
    return signature.startsWith("0x") ? signature : `0x${signature}`;
  }
  if (signature instanceof Uint8Array) {
    return `0x${Buffer.from(signature).toString("hex")}`;
  }
  if (Array.isArray(signature)) {
    return `0x${Buffer.from(signature).toString("hex")}`;
  }
  throw new Error("Unsupported signature format returned by ownerSigner.personalSign");
}
```
