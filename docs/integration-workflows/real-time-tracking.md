---
title: "Real-time Tracking"
description: "Use Tracking IDs and webhooks to show customers live delivery status without polling."
---

# Real-time Tracking

After an order is created, Yonne provides two mechanisms to track it: a **tracking link** you surface to customers, and **webhooks** that push status changes to your backend in real time.

---

## The two tracking tools

| Tool | Who it's for | How it works |
|---|---|---|
| `tracking_link` | Your customer | A hosted Yonne page showing live rider location |
| Webhooks | Your server | HTTP `POST` to your endpoint on every status change |

Use both. The tracking link gives customers a zero-effort view. Webhooks let your backend react to status changes without polling.

---

## The tracking link

Every successful `create-order` response includes a `tracking_link`:

```json
{
  "tracking_id": "YON-TRK-123456",
  "tracking_link": "https://yonne.app/track/YON-TRK-123456"
}
```

Show this on your order confirmation page and in your post-purchase email:

```html
<a href="{{ tracking_link }}">Track your delivery in real time →</a>
```

---

## Polling (fallback only)

If you need order status and missed a webhook event, poll the tracking endpoint:

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/v1/external/track/YON-TRK-123456" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```

```javascript Node.js
const res = await fetch(
  "https://api.yonne.app/api/v1/external/track/YON-TRK-123456",
  { headers: { "X-API-Key": process.env.YONNE_API_KEY } }
);
const status = await res.json();
console.log(status.status); // "In Transit", "Delivered", etc.
```

```python Python
import requests, os

status = requests.get(
    "https://api.yonne.app/api/v1/external/track/YON-TRK-123456",
    headers={"X-API-Key": os.environ["YONNE_API_KEY"]}
).json()
print(status["status"])
```
</CodeGroup>

<Warning>
  Do not poll on a tight interval. Use webhooks as your primary update mechanism and polling only for reconciliation or support lookups.
</Warning>

---

## Webhooks as the primary update source

Configure a webhook endpoint in your Yonne dashboard. Yonne will `POST` a JSON payload to your URL on every order status change.

Example payload:

```json
{
  "event": "order.status_updated",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "status": "In Transit",
  "merchant_reference_id": "WEB-100245",
  "metadata": {
    "channel": "woocommerce"
  },
  "timestamp": "2025-05-18T10:32:00Z"
}
```

Your webhook handler should:

1. Verify the HMAC-SHA256 signature (see [Webhook Setup & Security](/docs/webhooks/setup-and-security))
2. Look up the order by `order_id` or `merchant_reference_id`
3. Update the order status in your database
4. Trigger any downstream logic (customer notification, OMS update, support tool refresh)
5. Return `200 OK`

---

## Order status lifecycle

```
searching → assigned → picked_up → in_transit → delivered
                                                      │
                                              (terminal state)
                                              
Any state → cancelled (if eligible)
```

| Status | What it means | Suggested UX |
|---|---|---|
| `searching` | Looking for an available rider | "Finding your rider…" |
| `assigned` | Rider confirmed | "Rider is heading to the store" |
| `picked_up` | Item collected from merchant | "Your order is on the way!" |
| `in_transit` | Rider en route to customer | Show real-time tracking link |
| `delivered` | Order handed to customer | "Delivered — enjoy!" |
| `cancelled` | Order cancelled | Refund or retry flow |

---

## Testing your webhook handler

Use the simulate-status endpoint to push status changes in test mode without waiting for a real rider:

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
    "X-API-Key": process.env.YONNE_TEST_API_KEY,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ order_id: "ORD-123456", status: "In Transit" })
});
```

```python Python
import requests, os

requests.post(
    "https://api.yonne.app/api/v1/external/test/simulate-status",
    headers={
        "X-API-Key": os.environ["YONNE_TEST_API_KEY"],
        "Content-Type": "application/json"
    },
    json={"order_id": "ORD-123456", "status": "In Transit"}
)
```
</CodeGroup>

Walk through every status in sequence during QA to confirm your handler updates state correctly at each transition.

---

## Reconciliation strategy

If your webhook receiver goes down and misses events:

1. On recovery, query `GET /api/v1/external/track/:tracking_id` for any orders in a non-terminal state.
2. Update your database to match the current Yonne status.
3. Trigger any missed downstream actions (notifications, OMS updates).

Keep both `order_id` and `tracking_id` in your database from the moment the order is created — you'll need them for this reconciliation.
