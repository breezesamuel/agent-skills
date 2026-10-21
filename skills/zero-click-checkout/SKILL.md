---
name: zero-click-checkout
description: Build a checkout that never asks the customer to do anything — popup payment window, automatic return to the app, automatic order confirmation, automatic entitlement unlock, automatic receipt email, and automatic server-side reconciliation. Use when a user asks for "零点击支付", "popup checkout", "customer never leaves the page", "auto-confirm payment", "payment webhook handling", "idempotent order ledger", or wants a PayPal/Alipay/Stripe/x402 flow that completes without manual accounting.
---

# Zero-click checkout

Design a payment flow where the **customer** performs zero actions beyond
clicking "pay", and the **operator** performs zero manual bookkeeping.

The customer never switches back to the browser, never clicks a confirmation,
and never waits for a human to verify the payment. The operator never opens a
spreadsheet.

## The five moving parts

```
customer clicks Pay
   → popup checkout window opens (PayPal / Alipay / Stripe / x402)
   → provider redirects back to your return-URL inside the popup
   → return handler posts the result to the opener window
   → popup closes itself
   → order state machine advances: created → paid → activated → receipt sent
   → server-side reconciler is the source of truth (not the browser)
```

The single most important design decision: **the browser is never trusted to
report a payment**. It only *nudges* the server; the server verifies.

## 1. Popup + self-closing return (no "switch back to your browser")

```js
// open
const w = window.open(CHECKOUT_URL, "pay", "width=520,height=720");
window.__payWindow = w;

// on the return page (same origin or postMessage-capable)
function onReturn(result) {
  window.opener?.postMessage({ type: "bds:pay", result }, ORIGIN);   // exact origin, not "*"
  window.close();                                                     // self-close
}
window.addEventListener("message", (e) => {
  if (e.origin !== ORIGIN) return;          // origin check is mandatory
  if (e.data?.type === "bds:pay") handleResult(e.data.result);
});
```

Return URL must be a dedicated route that can be loaded standalone — it must
work when the opener is gone (user closed it) and degrade to a plain
"check your email" message.

## 2. Polling fallback (pop-up blockers kill `window.open`)

```js
async function pollStatus(outTradeNo) {
  const res = await fetch(`/api/paypal/status?out_trade_no=${encodeURIComponent(outTradeNo)}`);
  const j = await res.json();
  if (j?.paid) { unlock(); return; }
  setTimeout(() => pollStatus(outTradeNo), 2000);   // 2s cadence
}
```

Never use `window.opener` as the only channel — popup blockers and mobile
webviews break it.

## 3. Order state machine with an idempotent ledger

States: `created → paid → activated → receipt_sent`, plus `failed`, `refunded`.

The ledger write must be **atomic and idempotent** — providers retry
notifications, users double-click, and the reconciler polls on top of the
webhook. All three can fire for the same order.

```python
def record(self, order_id: str, **fields) -> bool:
    """Idempotent upsert. Returns True if this call changed the state."""
    with self._lock:
        cur = self._load()
        prev = cur.get(order_id)
        merged = {**(prev or {}), **fields}
        merged["updated_at"] = utcnow_iso()
        cur[order_id] = merged
        self._atomic_write(cur)      # write temp file + os.replace
        return prev != merged
```

Rules that keep it clean:

- Key by provider order id (`out_trade_no`), never by timestamp
- Store amounts as integer minor units or `Decimal` — never float
- Write via temp file + `os.replace` so a crash can't truncate the ledger
- Keep a `source` field (`webhook` / `poll` / `manual`) and a `receipt_sent`
  flag so the receipt is sent exactly once
- Counters (`created/paid/activated/failed/refunded`) live next to the data

## 4. Webhook verification (never trust the browser, never trust the payload)

Verify the signature *before* parsing business fields:

- **PayPal**: verify the webhook transmission with the API, check
  `transmission.sig_verified === "SUCCESS"` and that the `webhook_id` is one you
  created. Do not treat a `PAYMENT.CAPTURE.COMPLETED` payload as proof without
  also calling `GET /v2/checkout-orders/{id}` and checking
  `status === "COMPLETED"` and the amount/currency match.
- **Alipay**: `alipay_trade_no` + `out_trade_no` + `total_amount` must match
  what you stored; verify `sign` against Alipay's public key.
- **Stripe**: `stripe-signature` HMAC over the raw body (do not JSON-parse
  before verifying) and compare with a constant-time function.
- **x402**: the facilitator's settlement receipt is the receipt — verify the
  `x-payment-response` header's decoded payload against the expected amount
  and payee before granting access.

Always respond `200` **after** verification, and always make the handler
idempotent (the same event can arrive twice).

## 5. Automatic reconciliation (the operator's zero-touch guarantee)

The webhook can be lost. So a **pull-based reconciler** runs on a timer and is
the real source of truth:

```
every N minutes:
  for each order in state "created" and older than 30s:
      query the provider for the authoritative status
      if paid  → advance state, activate entitlement, send receipt
      if failed → mark failed, release reservation
```

Pull-based beats push here: push depends on an inbound tunnel/webhook staying
alive; pull only needs outbound network, which is far more reliable. Keep the
push path too, as a low-latency hint, but let pull be authoritative.

**Fail closed on missing credentials**: if the reconciler has no valid API
token, it must run **read-only** — log what it would have done, change
nothing. Never let a missing secret look like "no payments happened".

## 6. Automatic receipt

Send the receipt from the server the moment the order reaches `paid`:

- receipt goes to the address captured at checkout, not to a hardcoded one
- include: provider order id, your order id, amount + currency, plan, timestamp
- mark `receipt_sent` in the same ledger write so it never sends twice
- use an SMTP relay you control, with the **16-character app password**, not
  the account login password (QQ/163 mailboxes require this; a wrong password
  fails with `535` and looks like a server bug)

## 7. Entitlement unlock must not depend on the payment store you don't have

A very common bug: the unlock endpoint reads orders from a database that the
deployment does not actually have configured. Result: the customer paid, the
order exists in the KV store, and "unlocked tools" is permanently empty.

Fix with a fallback chain and a per-ref index:

```js
// on saveOrder: maintain a ref -> order-ids index (idempotent, dedupe)
const idx = (await dbGet('orders_by_ref_' + rec.ref)) || [];
if (!idx.includes(rec.out_trade_no)) {
  idx.push(rec.out_trade_no);
  await dbSet('orders_by_ref_' + rec.ref, idx);
}

// on listByRef: try primary store, then the index
async function listByRef(ref) {
  const primary = await queryPrimaryStore(ref);
  if (primary.length) return primary;
  const idx = (await dbGet('orders_by_ref_' + ref)) || [];
  const out = [];
  for (const id of idx) {
    const rec = await getOrder(id);
    if (rec && rec.status === 'paid') out.push(rec);
  }
  return out;
}
```

## 8. Referral / discount ladders are a revenue leak — cap them

A common ladder is `[[1,10],[2,30],[3,50],[5,70],[10,100]]` meaning "invite 10
friends → 100% off". At 100% off the platform receives **¥0** and the referrer's
commission is **also ¥0** — both sides lose, and the only winners are people who
never pay.

Cap the ladder where the *server* computes the price (client-side caps are
decorative) and state the economics:

```
net_revenue = list_price × (1 − max_discount) × platform_share
```

With a 30% cap and a 70% platform share: `list × 0.7 × 0.7 = 49% of list`.

Apply the cap in **both** places: the client ladder (for display) and the
server ladder (for the authoritative amount).

## Definition of done

- [ ] Customer clicks one button and never returns to the tab manually
- [ ] Return page closes its own popup and posts to the opener with an origin check
- [ ] A polling fallback exists for blocked popups
- [ ] The ledger is idempotent, atomic, and survives a crash mid-write
- [ ] Webhook signature verification happens before business logic
- [ ] A pull-based reconciler advances orders the webhook missed
- [ ] Receipts are sent exactly once and recorded as sent
- [ ] The unlock endpoint works with the store you actually deployed
- [ ] The discount ladder is capped on the server and the net-revenue formula is written down
- [ ] Nothing reports success without a verifiable record (ledger entry, provider id, HTTP status)

## Anti-patterns

- Trusting a `?success=true` query parameter
- Sending the receipt from the browser
- Reconciling only via webhook
- Writing the ledger with `json.dump` straight to the target path
- Shipping a client-side discount ladder the server doesn't enforce
- A "test payment" mode that can silently become live
