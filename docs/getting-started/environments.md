---
title: "Environments"
description: "Yonne has two environments — test and live — selected by your API key prefix. The host is identical for both."
---

Yonne uses a single API host for both environments:

```text
https://api.yonne.app
```

Your API key prefix is the only switch between them. Use `yonne_test_` keys while building and `yonne_live_` keys in production.

---

## Side-by-side comparison

| Behavior | Live (`yonne_live_`) | Test (`yonne_test_`) |
|---|---|---|
| Wallet balance checked | Yes — real deduction | Sandbox context only |
| Real rider dispatch | Yes | No production riders assigned |
| `ERR_NO_CAPACITY_AVAILABLE` possible | Yes | Possible in sandbox |
| Webhooks dispatched | Yes | Yes (plus simulation endpoint available) |
| `environment` field in responses | `"live"` | `"test"` |
| Tracking IDs generated | Real IDs | Test IDs |

---

## Test mode

Use test mode to build your integration, run your CI pipeline, and validate webhook handling — without touching your live wallet or dispatching real riders.

### Recommended test workflow

**1. Validate your test key**

```bash
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_test_xxxxxxxxx"
```

**2. Quote and create orders normally** — the same payloads work in both environments.

**3. Simulate status changes** to test your webhook handler without waiting for a real delivery:

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/test/simulate-status" \
  --header "X-API-Key: yonne_test_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --data '{
    "order_id": "ORD-123456",
    "status": "In Transit"
  }'
```

```javascript Node.js
await fetch("https://api.yonne.app/api/v1/external/test/simulate-status", {
  method: "POST",
  headers: {
    "X-API-Key": "yonne_test_xxxxxxxxx",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ order_id: "ORD-123456", status: "In Transit" })
});
```

```python Python
import requests

requests.post(
    "https://api.yonne.app/api/v1/external/test/simulate-status",
    headers={
        "X-API-Key": "yonne_test_xxxxxxxxx",
        "Content-Type": "application/json"
    },
    json={"order_id": "ORD-123456", "status": "In Transit"}
)
```
</CodeGroup>

---

## Live mode checklist

Before switching to `yonne_live_`, confirm each of these:

- [ ] Merchant pickup address is set in the dashboard
- [ ] Wallet has sufficient balance for expected order volume
- [ ] Your integration sends `Idempotency-Key` on every `create-order` call
- [ ] Your webhook receiver is live, reachable, and validates HMAC signatures
- [ ] `402 Insufficient Funds` and `422 ERR_NO_CAPACITY_AVAILABLE` are handled separately in your error logic
- [ ] Live and test keys are stored in separate environment variables

---

## Common mistakes

**Using a live key in development** — your real wallet gets debited. Keep keys in `.env` files and never hardcode them.

**Treating test orders as real fulfillments** — test mode validates your integration logic, not real courier operations. Do not assume a rider will be assigned.

**Wrong key for the environment** — if you see `order_not_found` or unexpected wallet values, you're likely querying an order that exists in one environment using a key from the other.
