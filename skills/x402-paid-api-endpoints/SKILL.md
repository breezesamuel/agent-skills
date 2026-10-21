---
name: x402-paid-api-endpoints
description: Turn an HTTP endpoint into a paid API using the x402 protocol — 402 challenge, per-call pricing, facilitator settlement, EIP-3009 signed authorizations, and a client that pays automatically. Use when a user asks to "monetize an API endpoint", "pay-per-call HTTP", "x402", "charge for API requests with crypto", "agent-to-agent payments", or wants an agent service sold per invocation.
---

# x402 paid API endpoints

x402 turns a normal HTTP endpoint into a paid one: the server answers
`402 Payment Required` with a price and payment details, the client pays a
facilitator, retries with a payment header, and gets the real response.

The appeal for agent services: no subscriptions, no seat minimum, no card
onboarding — a machine can pay per call.

## The request/response cycle

```
client ──▶ POST /entrypoints/audit/invoke
client ◀── 402 + { accepts: [{ scheme, network, maxAmountRequired, ... }],
                   + PAYMENT-REQUIRED header (base64 JSON) }
client ──▶ facilitator settles the payment (signs with the client's key)
client ──▶ POST /entrypoints/audit/invoke  +  X-PAYMENT: <signed payload>
server  ──▶ verify → settle → 200 { result }
```

## Server side

### 1. Declare the price and the payee

```
network: base-sepolia
asset:   USDC (or the EVM address of your token)
price:   0.0025 USDC per call  → atomic units (×10^6) = 2500
payee:   0x12A2b19eFA9D8BC48ac156Cc8FdfC7cC0Dff36aB
```

Keep a **price guardrail** in the code, not just in the config:

```ts
if (priceUsd > 0.05) throw new Error("price guardrail: refusing to serve above $0.05");
// and assert the same constant in the UI that renders the price
```

A fat-fingered `0.25` instead of `0.0025` is a 100× price change on a live
paid endpoint. The guardrail is cheap insurance.

### 2. Answer 402 correctly

The 402 body must tell the client exactly how to pay: which network, which
asset address, how much (atomic), where the payee is, and the resource being
bought. Clients parse this; a vague 402 fails closed with a confusing error.

### 3. Verify, then settle

Never trust the client's claim that it paid. The verification order:

1. Decode the client's payment header
2. Check the amount ≥ your price, the network matches, the payee is yours
3. Check the signature (EIP-3009 `transferWithAuthorization` over
   `TransferWithAuthorization` typed data) against the client's address
4. **Settle through the facilitator** and read back the confirmed receipt
5. Only then run the paid work and return `200`

Include a receipt header in the response so the client can prove it paid.

### 4. Guard the work itself

The paid handler must be safe to run twice (the client may retry). Either the
work is idempotent (a read, an audit) or you dedupe on the payment id.

Also cap the input: a paid endpoint is still a public endpoint. Validate the
wallet address format before spending RPC calls.

## Client side

```ts
import { wrapFetchWithPayment, createSigner, decodeXPaymentResponse } from "x402-fetch";
const signer = await createSigner("base-sepolia", process.env.PRIVATE_KEY!);
const fetchWithPayment = wrapFetchWithPayment(fetch, signer);
const res = await fetchWithPayment(`${BASE_URL}/entrypoints/audit/invoke`, {
  method: "POST", headers: { "content-type": "application/json" },
  body: JSON.stringify({ input: { wallet: "0x…" } }),
});
const receipt = decodeXPaymentResponse(res.headers.get("x-payment-response")!);
```

The wrapper handles the 402 → pay → retry loop. Read the decoded receipt and
**record it**: the payment id and the settled amount are the only evidence
that a call was paid for.

## Publishing a paid endpoint so agents can find it

A paid endpoint nobody can discover earns nothing.

- Publish an **agent card / manifest** at `/.well-known/agent.json` (or
  `agent-card.json`): name, description, entry points, input/output JSON
  Schema, pricing, network, and an invoke URL.
- Every entry point page must show: what it does, the price in USD, the
  network, the exact `curl` invocation, and both input and output schemas.
- Put a human-readable landing page at the root — most visitors will never
  send a `X-PAYMENT` header; they need to see what it is and what it costs.
- List the endpoint in a directory/marketplace so agents can discover it by
  capability rather than by URL.

## Testnet before mainnet

Run the full cycle on a testnet before charging real money:

- the 402 handshake completes
- the facilitator settles
- the paid work returns the right result
- a request with **no** payment header is rejected
- a request with an **underpaid** header is rejected
- a request with a **tampered** payload is rejected

Write these as automated checks, not as manual steps. A paid endpoint that
serves free content is a hole in your business; a paid endpoint that rejects
valid payments is a dead product.

## Honest limits to document

- Payment is per call — buyers with tiny budgets need a floor, and a per-call
  price of $0.0025 means ~400 calls per $1
- The client pays a facilitator fee on top of your price
- Settlement is not instant on every network; design for "paid, processing"
- Testnet prices must be labelled as such — never show a $0.00 mainnet price
  for a testnet-only endpoint

## Anti-patterns

- Trusting an `X-PAYMENT` header without facilitator verification
- A price constant defined in two places that can drift apart
- No price guardrail
- Exposing the paid endpoint without a 402 path (it becomes free)
- Publishing the endpoint but no schema — agents cannot use what they cannot parse
