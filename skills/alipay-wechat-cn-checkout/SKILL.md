---
name: alipay-wechat-cn-checkout
description: Ship Alipay and WeChat Pay checkout for a Chinese-market web product — order creation, notify/callback verification, refund, and reconciliation without a licensed aggregator. Use when a user asks for "支付宝支付接入", "微信支付", "扫码支付", "china payment integration", "alipay trade query", "wechat pay v3 notify", or wants to take domestic payments with an appid/mchid instead of a foreign gateway.
---

# Alipay & WeChat Pay checkout for a Chinese-market product

Domestic checkout is a different problem from Stripe/PayPal: the gateway is
Alipay/WeChat itself, the money settles in CNY, and the merchant needs a
verified appid/mchid rather than a merchant onboarding flow.

This skill covers the integration surface that actually breaks in practice:
signature verification, the notify callback, order-state convergence, and
refunds.

## Credentials and their exact shapes

| Item | Where it comes from | Shape |
|---|---|---|
| Alipay appid | Alipay Open Platform → app | 16 digits (e.g. `2021006197622702`) |
| Alipay private key | Open Platform → key → RSA2 | **PEM**, `-----BEGIN PRIVATE KEY-----` … |
| Alipay public key | Open Platform → key → Alipay public key | **PEM** |
| WeChat appid | mp.weixin.qq.com → appid | 18 chars, starts with `wx` |
| WeChat mchid | merchant platform → mchid | 10 digits |
| WeChat API v3 key | merchant platform → APIv3 key | exactly 32 chars |
| WeChat private key | merchant platform → API certificate | **PEM**, `-----BEGIN PRIVATE KEY-----` … |

**A PEM that does not start with `-----BEGIN` is not a key.** A truncated key,
a base64 blob without the header, or a JSON-escaped string pasted into a
`Config` env var will fail at signature time with an opaque error. Validate
the shape at startup, not at request time:

```python
def _valid_pem(v: str | None) -> bool:
    if not v: return False
    s = v.replace("\\n", "\n").strip()
    return s.startswith("-----BEGIN") and "PRIVATE KEY-----" in s
```

If a PEM is not valid, **disable that channel explicitly** and log why — do not
let it fail halfway through a customer's payment.

## The PEM-in-env-var trap

Environment variables cannot hold newlines cleanly. Always normalize:

```python
key = (os.environ.get("WECHAT_PRIVATE_KEY") or "").replace("\\n", "\n")
```

and keep the PEM in a secret store, never in a repo or a `Config` enum.

## Alipay: the two-request dance

### Request (create order)

```
POST https://openapi.alipay.com/gateway.do
  method=alipay.trade.page.pay  (or alipay.trade.app.pay / alipay.trade.precreate for QR)
  app_id=<appid>
  method=<signed method>
  charset=utf-8
  sign_type=RSA2
  timestamp=<ISO8601 with timezone>
  version=1.0
  biz_content=<JSON: out_trade_no, total_amount, subject …>
  sign=<base64 RSA2 signature over the sorted param string>
```

**Signature construction** (this is where it always breaks):

1. Take all **non-empty** params except `sign` and `sign_type`
2. Sort by key **ASCII ascending**
3. Join as `k=v&k=v` (raw values, **not** URL-encoded at this step)
4. Sign with the merchant private key, `SHA256withRSA`, base64
5. URL-encode only when you put it in the request body

For `alipay.trade.app.pay` and `page.pay` the browser receives a **form POST or
a redirect URL with `sign`** — do not try to call these APIs server-side and
expect a JSON response. For server-side flows use:

- `alipay.trade.precreate` → returns `qr_code` (QR code URL)
- `alipay.trade.query` → returns order status

### Notify (the async callback)

Alipay calls your notify URL with the same signed params. Verify, then respond
with the plain string `success` (any other body makes Alipay keep retrying for
up to 24 hours).

```python
def verify_alipay(params: dict) -> bool:
    sign = params.pop("sign", "")
    params.pop("sign_type", None)
    sign_str = "&".join(f"{k}={v}" for k, v in sorted(params.items()) if v not in (None, ""))
    expected = base64.b64encode(
        private_key.sign(sign_str.encode(), padding.PKCS1v15(), hashes.SHA256())
    ).decode()
    return hmac.compare_digest(expected, sign)
```

Check `trade_status` ∈ {`TRADE_SUCCESS`, `TRADE_FINISHED`}, and that
`out_trade_no` matches what you stored **and** `total_amount` matches what you
charged. A callback whose amount differs is either a bug or an attack.

## WeChat Pay v3: the parts that bite

- Requests are signed with **API v3** (HMAC-SHA256 over
  `timestamp\nnonce\nbody\n`) using the **APIv3 key**; the *certificate* is used
  only to decrypt the platform certificate and to verify platform
  notifications.
- The response header is `Wechatpay-Signature`; construct the string as
  `timestamp\nnonce\nbody\n` and verify with the platform public key.
- **AES-256-GCM** decrypts the notification resource:
  `associated_data + nonce + ciphertext` with the APIv3 key; the auth tag is
  the last 16 bytes of the base64-decoded `ciphertext`.
- The notify body carries `resource`; the plain fields (transaction id,
  out_trade_no) are **inside** the encrypted resource.
- WeChat's notification must be answered with HTTP **200** and an empty body.
- `out_trade_no` must be unique and ≤ 32 chars; `total` is in **fen** (integer).

## Order state convergence (the part users feel)

Alipay/WeChat are asynchronous and both retry notifications. So the notify
handler and the query API must converge on the same state machine:

```
created → paid → activated → receipt_sent
```

- The notify handler is the fast path; the query API is the truth.
- Write the ledger idempotently, keyed by `out_trade_no`.
- When the ledger says `paid` but the customer did not get access, the bug is
  almost always in the **entitlement** step, not the payment step. Test the
  entitlement lookup separately, with the exact store you deployed.

## Refund

- Alipay: `alipay.trade.refund` with `out_request_no` (unique per refund
  attempt — it is the idempotency key) and `refund_amount`
- WeChat: `POST /v3/refund/domestic/refunds` with `out_refund_no`
- Both are asynchronous: poll the refund query until `SUCCESS`
- Never refund from a client-side button — always from a server-side,
  admin-authenticated path

## Reconciliation without an aggregator

Run a pull-based reconciler on a timer:

```
every N minutes:
  for each order in state created and older than 30s:
      alipay.trade.query(out_trade_no)  /  wechat v3 query by out_trade_no
      advance state, activate entitlement, send receipt
```

This is what makes "no manual accounting" true. Notifications get lost; a
pull loop does not.

## Environment checklist

- [ ] Alipay private key is a real PEM (validate the header)
- [ ] Alipay public key is the **Alipay** public key, not your own
- [ ] WeChat private key is a real PEM, and APIv3 key is exactly 32 chars
- [ ] `notify_url` is publicly reachable over HTTPS and its cert is valid
- [ ] Alipay app is in production state (not sandbox) before real traffic
- [ ] Sandbox and production credentials are never both present in one env
- [ ] Currency is CNY and amounts are integers (fen for WeChat, yuan with 2dp
      for Alipay)

## Anti-patterns

- Treating the notify callback as the only source of truth
- Returning JSON from the WeChat notify handler instead of an empty 200
- Reusing `out_request_no` across refund attempts (double refunds)
- A PEM stored with literal `\n` escapes and never normalized
- Amount comparison with floats
- Enabling a channel whose credentials are invalid and discovering it at checkout time
