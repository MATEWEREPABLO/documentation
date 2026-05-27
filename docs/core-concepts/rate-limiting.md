---
title: "Rate Limiting"
description: "Yonne enforces rate limits to protect platform stability. Use exponential backoff and avoid polling when webhooks can do the job."
---

Yonne enforces rate limits across all merchants to maintain platform stability. Design your integration to send predictable, controlled traffic.

---

## General guidance

- **Use webhooks over polling.** Don't repeatedly call `/track/:id` to check order status — let Yonne push status updates to you instead. See [Real-time Tracking](/docs/integration-workflows/real-time-tracking).
- **Cache stable data.** The response from `GET /api/v1/external/validate` rarely changes. Cache `pickupLatitude`, `pickupLongitude`, and `pickupAddress` instead of fetching them on every order.
- **Use exponential backoff** on transient failures (`5xx`, network errors). Don't retry at a fixed 100ms interval.
- **Never use tight retry loops** on business errors like `402` or `422`. These require human or system-level resolution, not hammering the API.

---

## Handling a `429 Too Many Requests` response

If you exceed the rate limit, the API returns `429`. Honor it — back off and retry after the delay indicated in the response headers.

<CodeGroup>
```javascript Node.js
async function callWithBackoff(fn, maxRetries = 4) {
  for (let i = 0; i < maxRetries; i++) {
    const response = await fn();
    if (response.status !== 429) return response;

    const retryAfter = parseInt(response.headers.get("Retry-After") || "2", 10);
    await new Promise(r => setTimeout(r, retryAfter * 1000 * Math.pow(2, i)));
  }
  throw new Error("Rate limit exceeded after max retries");
}
```

```python Python
import time, requests

def call_with_backoff(fn, max_retries=4):
    for attempt in range(max_retries):
        response = fn()
        if response.status_code != 429:
            return response
        retry_after = int(response.headers.get("Retry-After", 2))
        time.sleep(retry_after * (2 ** attempt))
    raise Exception("Rate limit exceeded after max retries")
```
</CodeGroup>

---

## Order creation is the most sensitive endpoint

`POST /api/v1/external/create-order` is the only mutating endpoint that should ever be retried, and only with the **same `Idempotency-Key`**. Retrying with a new key creates a new order.

See [Idempotency & Retries](/docs/core-concepts/idempotency-and-retries) for the full retry decision tree.

---

## Batch and import jobs

If you're replaying a backlog of orders or running an import job, throttle your own send rate. Don't burst hundreds of requests at once — stagger them with a short delay between each call.
