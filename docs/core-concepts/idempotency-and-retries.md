---
title: "Idempotency & Retries"
description: "Use Idempotency-Key on every create-order call to prevent duplicate orders when networks fail or requests time out."
---

# Idempotency & Retries

Networks fail. Clients time out. Users double-click checkout buttons. The `Idempotency-Key` header is how you guarantee that no matter how many times you retry a request, Yonne creates the order exactly once.

---

## How it works

Send a unique string in the `Idempotency-Key` header on every `POST /api/v1/external/create-order` call:

```http
POST /api/v1/external/create-order
X-API-Key: yonne_live_xxxxxxxxx
Idempotency-Key: order-100245-attempt-1
Content-Type: application/json
```

If Yonne receives a second request with the same key within **24 hours**, it returns the original success response instead of creating a second order. The response may include:

```json
{
  "idempotent_replay": true
}
```

---

## Generating a good idempotency key

A good key encodes your internal order ID plus a retry attempt counter so you can distinguish retries from genuinely new orders:

```text
order-{your_internal_order_id}-attempt-{n}
```

Examples:
- `order-WEB-100245-attempt-1` — first attempt
- `order-WEB-100245-attempt-2` — retry after timeout

<Warning>
  Never generate a new random key on each retry. If you do, Yonne will create a new order every time, and you'll have duplicates.
</Warning>

---

## The retry decision tree

```
Request failed or timed out?
│
├─ Network error / timeout / 5xx
│   └─ Retry with the SAME Idempotency-Key ✓
│
├─ 401 Unauthorized
│   └─ Fix your API key — do not retry ✗
│
├─ 400 Validation error (missing coordinates, etc.)
│   └─ Fix the payload — do not retry ✗
│
├─ 402 Insufficient Funds
│   └─ Top up wallet, then retry ✗ (do not retry blindly)
│
└─ 422 ERR_NO_CAPACITY_AVAILABLE
    └─ Wait for capacity, then retry ✗ (do not retry blindly)
```

---

## Safe vs. unsafe retries

| Request | Safe to retry? | Condition |
|---|---|---|
| `GET /validate` | Yes | Always safe |
| `GET /quote` | Yes | Always safe |
| `POST /create-order` | Yes, with conditions | Only with the **same** `Idempotency-Key` |
| `POST /create-order` | No | Do not retry on `402` or `422` without resolving the underlying issue |
| `GET /track/:id` | Yes | Always safe |

---

## Handling a timed-out create-order

If your client times out before receiving a response, you don't know whether Yonne created the order. Here's what to do:

<CodeGroup>
```javascript Node.js
async function createOrderWithRetry(payload, internalOrderId, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const response = await fetch("https://api.yonne.app/api/v1/external/create-order", {
        method: "POST",
        headers: {
          "X-API-Key": process.env.YONNE_API_KEY,
          "Content-Type": "application/json",
          "Idempotency-Key": `order-${internalOrderId}-attempt-${attempt}`
        },
        body: JSON.stringify(payload),
        signal: AbortSignal.timeout(10000) // 10s timeout
      });

      const data = await response.json();
      if (data.success) return data;

      // Business errors — do not retry
      if (response.status === 402 || response.status === 422) throw data;

    } catch (err) {
      if (attempt === maxAttempts) throw err;
      await new Promise(r => setTimeout(r, 1000 * attempt)); // backoff
    }
  }
}
```

```python Python
import time
import requests

def create_order_with_retry(payload, internal_order_id, max_attempts=3):
    for attempt in range(1, max_attempts + 1):
        try:
            response = requests.post(
                "https://api.yonne.app/api/v1/external/create-order",
                headers={
                    "X-API-Key": os.environ["YONNE_API_KEY"],
                    "Content-Type": "application/json",
                    "Idempotency-Key": f"order-{internal_order_id}-attempt-{attempt}"
                },
                json=payload,
                timeout=10
            )
            data = response.json()
            if data.get("success"):
                return data
            # Business errors — do not retry
            if response.status_code in (402, 422):
                raise Exception(data)
        except requests.Timeout:
            if attempt == max_attempts:
                raise
            time.sleep(attempt)  # linear backoff
```
</CodeGroup>

---

## What to log on every order attempt

Always log these fields so you can reconcile duplicate concerns in support:

- `merchant_reference_id` — your internal order ID
- `Idempotency-Key` — the exact key you sent
- `order_id` — from the Yonne response
- `tracking_id` — from the Yonne response
- HTTP status code
- Full error payload on failure
