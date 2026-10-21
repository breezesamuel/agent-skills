---
name: onchain-approval-audit
description: Audit EVM wallets for unlimited or stale token approvals and emit ready-to-sign revoke calldata. Use when a user asks to check "token approvals", "revoke.cash style audit", "is my wallet safe from approval drain", "which contracts can spend my tokens", or wants to find unlimited ERC-20 allowances across Uniswap/Permit2/1inch/0x/CoW/Aave style spenders. Covers eth_getLogs + Multicall3 batch reading, chain-verified constants, and the exact calldata encoding traps.
---

# On-chain Approval Audit

Find every live token approval on an EVM wallet, flag the dangerous ones
(`unlimited` = allowance ≥ 95% of 2²⁵⁶−1, `stale` = granted long ago and never
revoked), and return `approve(spender, 0)` calldata the user can sign.

Read-only: it never touches keys and never moves funds.

## When to use this skill

- "Check my wallet's token approvals" / "is my USDT allowance safe?"
- "Which contracts can spend my tokens?" / "who has approval on my address?"
- "Give me the transactions to revoke old approvals"
- Auditing a treasury / multisig signer set before a token migration
- Building an approval-risk report tool or an Apify Actor / MCP server

## The five traps that make naive auditors silently wrong

Every one of these was hit in production while building this. All five fail
**silently** — the tool returns "no risk found" while missing real exposure.

### 1. A wrong event topic0 returns zero logs and raises no error

`eth_getLogs` with a malformed `topic0` returns `[]` — no exception, no warning.
Your audit then reports a clean wallet.

The correct `Approval(address,address,uint256)` topic0 is:

```
0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
```

**Never trust a topic constant from memory or from a blog post.** Verify it
against the chain at runtime (see step 0 below).

### 2. ABI addresses must be padded to 32 bytes (64 hex chars), not 20 bytes

`padStart(40, "0")` produces calldata whose fields are **misaligned**. The
revoke transaction either gets rejected or — worse — targets the wrong address.
Correct: `padStart(64, "0")`.

A revoke payload is `selector(4) + address(32) + value(32)` = 132 bytes = 264 hex
chars after `0x`. Assert on that length before shipping.

### 3. Hand-rolled ABI encoding for `aggregate3` reverts

Multicall3's `aggregate3((address,bool,bytes)[])` has a dynamic array head
(offset `0x20`) **and** a per-element tuple offset (`0x60`), plus 32-byte
alignment on the `bytes` payload. Miss any of them and every node answers
`execution reverted`.

Do not hand-encode. Use `viem`:

```js
import { encodeFunctionData, decodeFunctionResult } from "viem";

const AGGREGATE3_ABI = [{
  name: "aggregate3", type: "function", stateMutability: "view",
  inputs: [{ name: "calls", type: "tuple[]", components: [
    { name: "target", type: "address" },
    { name: "allowFailure", type: "bool" },
    { name: "callData", type: "bytes" },
  ] }],
  outputs: [{ name: "returnData", type: "tuple[]", components: [
    { name: "success", type: "bool" },
    { name: "returnData", type: "bytes" },
  ] }],
}];

const data = encodeFunctionData({
  abi: AGGREGATE3_ABI, functionName: "aggregate3",
  args: [{ target, allowFailure: true, callData: "0xdd62ed3e..." }],
});
const decoded = decodeFunctionResult({
  abi: AGGREGATE3_ABI, functionName: "aggregate3", data: result,
});
```

### 4. A 20-byte checksum typo makes the whole RPC call fail

viem validates EIP-55 checksums. One bad checksum in your hardcoded spender
list aborts the entire batch with `InvalidAddressError`, so you lose *all*
allowance results, not just one.

Recompute every checksum from the lowercase form and verify on-chain:

```js
import { getAddress } from "viem";
getAddress(addr.toLowerCase());      // canonical EIP-55
await rpc("eth_getCode", [addr, "latest"]); // must not be "0x"
```

### 5. The log window is short — old unlimited grants are invisible

Public RPCs serve `eth_getLogs` over roughly the last 600 blocks (≈2 hours).
An unlimited grant from 8 months ago is simply **not in the window**, and if its
spender isn't on your curated list you will never see it. The most dangerous
exposure is usually the oldest one.

Cover it two ways:
- **Exact current allowances** via Multicall3 for a curated spender list
  (Permit2, Uniswap V2/V3/Universal routers, 1inch V6, 0x, ParaSwap, CoW GPv2,
  Aave V3, Lido stETH) — this is not window-limited.
- **Full history** via Etherscan `account/tokentapproval` when an API key is
  available.

## Procedure

### Step 0 — verify your constants against the chain (before anything else)

```js
const probe = await rpc("eth_getLogs", [{
  address: USDC, fromBlock: "0x" + (head - 20).toString(16), toBlock: "0x" + head.toString(16),
}]);                       // no topic filter on purpose
const seen = new Set(probe.map(l => l.topics[0].toLowerCase()));
if (!seen.has(APPROVAL_TOPIC0)) throw new Error("topic0 mismatch — logs path is blind");
```

Cache the result per process. If it fails, **say so in the output** instead of
returning an empty result.

### Step 1 — read the approval events

One request for many tokens, chunked (public RPCs reject large `address` arrays):

```js
for (const chunk of chunks(addresses, 4)) {
  const logs = await rpc("eth_getLogs", [{
    address: chunk,
    topics: [APPROVAL_TOPIC0, padTopic(owner)],   // topic1 = owner
    fromBlock: "0x" + from.toString(16), toBlock: "0x" + head.toString(16),
  }]);
}
```

Batch the block-timestamp lookups — one HTTP round trip instead of N:

```js
const res = await rpcBatch(blockNums.map(bn => ({
  method: "eth_getBlockByNumber", params: ["0x" + bn.toString(16), false],
})));
```

### Step 2 — read exact current allowances with Multicall3

One `eth_call` for `tokens × spenders`. This is the only window-independent
source of truth for "can this contract still spend my tokens right now".

Treat the returned allowance as **authoritative** and let it override the
historical event value: if the current allowance is 0, the grant was revoked —
do not report it.

### Step 3 — merge, keeping the newest state per (token, spender)

Key by `token:spender`. Keep the record with the higher block number. The
current allowance, when known, wins over any historical value.

### Step 4 — classify

| Risk | Rule | Severity |
|---|---|---|
| `unlimited` | allowance ≥ 95% × (2²⁵⁶−1) | high |
| `stale` | last approval older than the threshold (default 90 days) and allowance still > 0 | medium |
| `direct` | any other non-zero allowance | info |

Only flag `stale` when you actually have a timestamp. Entries found purely
through the allowance path have no approval date — report them as `direct`
with an explicit note rather than inventing a date.

### Step 5 — emit revoke calldata

```js
function approveZeroCalldata(spender) {
  return "0x095ea7b3"
    + spender.toLowerCase().replace(/^0x/, "").padStart(64, "0")   // 64, not 40
    + "0".repeat(64);
}
```

`to` is the **token** contract, `data` is the approve-to-zero payload.

### Step 6 — verify before returning

- revoke payload length == 264 hex chars after `0x`
- payload matches `/^0x095ea7b3[0-9a-f]{128}$/`
- at least one approval record was found for a wallet you expect to have some
- the topic self-check from step 0 passed

## Output shape

```json
{
  "wallet": "0x…",
  "mode": "logs",
  "scanned": { "from_block": 1, "to_block": 2, "tokens": 12, "spenders": 11,
               "coverage_note": "only recent window; use mode=etherscan for full history" },
  "approvals": [
    { "token_symbol": "USDC", "spender": "0x…", "value": "115792089237316195423570985008687907853269984665640564039457584007913129639935",
      "unlimited": true, "last_approved_at": 1792587515, "source": "allowance" }
  ],
  "risk_flags": [
    { "risk": "unlimited", "severity": "high",
      "detail": "allowance ≥ 95% of 2^256-1, spender can drain this asset" }
  ],
  "revoke_tx_data": [
    { "token": "0xa0b8…", "spender": "0x68b3…", "to": "0xa0b8…",
      "data": "0x095ea7b3…", "human_readable": "approve(spender, 0)" }
  ],
  "notes": []
}
```

Always include `notes` — state the coverage limit, the missing API key, or a
failed check. An audit that hides its blind spots is worse than no audit.

## Performance notes

- 599-block log window + Multicall3 for 12 tokens × 11 spenders ≈ 11–15s cold
- Batching block timestamps took a naive 124s implementation down to 15s
- Public RPC failover list that works today: `ethereum-rpc.publicnode.com`,
  `eth.drpc.org`, `1rpc.io/eth`, `cloudflare-eth.com`
- `rpc.ankr.com/eth` now demands an API key — do not put it in a no-key list

## Honest limits to state to the user

- Log-window coverage without an Etherscan key is roughly the last 2 hours
- Permit2 allowances are not plain ERC-20 allowances — label them, don't guess
- A spender that was granted and then fully revoked is correctly absent
- This reads chain state only; it never signs and never spends
