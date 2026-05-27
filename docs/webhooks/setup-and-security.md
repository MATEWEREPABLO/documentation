---
title: "Webhook Setup & Security"
description: "Configure your Yonne webhook endpoint and verify every payload with HMAC-SHA256 signature validation."
---

Yonne delivers real-time order status updates to your server via webhooks. Every payload is signed with HMAC-SHA256 — you must verify the signature on every request to ensure the payload came from Yonne and hasn't been tampered with.

---

## Overview

Rather than polling the Yonne API for order status changes, webhooks let Yonne push updates to your server the moment something happens. This enables:

- **Real-time order tracking** — update your OMS as soon as a driver is assigned or an order is delivered.
- **Automated customer notifications** — send SMS or email confirmations without polling.
- **Fulfilment automation** — trigger warehouse or logistics workflows the instant an order changes state.

---

## Setting up your endpoint

### 1. Configure your webhook URL and secret

In your **Yonne merchant dashboard**, navigate to **Settings → Webhooks** and fill in two fields:

| Field | Description |
|---|---|
| `webhook_url` | Your publicly reachable HTTPS endpoint that accepts `POST` requests (e.g. `https://yourapp.com/webhooks/yonne`) |
| `webhook_secret` | A secret string shared between Yonne and your server, used to sign and verify every request |

> **No `webhook_url`, no webhooks.** If this field is not set for your merchant account, Yonne will not send any webhook notifications for your orders.

### 2. Expose your endpoint

Your endpoint must be reachable from the public internet. During development, use a tunnelling tool to expose your local server:

```bash
ngrok http 3000
# Register https://abc123.ngrok.io/webhooks/yonne in your dashboard
```

### 3. Respond with `200 OK` within 10 seconds

Yonne considers any response outside the 10-second window as a failure and will schedule a retry. Return `200` as fast as possible and process the event asynchronously (see [Best Practices](#best-practices)).

---

## The webhook payload

Every event uses the same envelope structure:

```json
{
  "event": "order.driver_assigned",
  "created_at": "2026-05-26T14:30:00.000Z",
  "data": {
    "order_id": "ORD123456",
    "tracking_number": "YON-ABC123",
    "tracking_url": "https://yonne.app/track/YON-ABC123",
    "status": "Awaiting Acceptance",
    "environment": "live",
    "delivery_fee": 3500.0,
    "pickup_address": "Area 18, Lilongwe",
    "delivery_address": "Area 3, Lilongwe",
    "receiver_name": "John Doe",
    "merchant_reference_id": "your-internal-order-id",
    "metadata": {},
    "courier": {
      "driver_name": "James Banda",
      "driver_phone": "+265991234567",
      "vehicle_type": "BIKE"
    }
  }
}
```

### Key fields

| Field | Description |
|---|---|
| `event` | The event type that triggered this delivery (see [Event Catalog](/docs/webhooks/event-catalog)) |
| `created_at` | ISO 8601 timestamp of when the event occurred |
| `data.order_id` | Yonne's internal order identifier |
| `data.merchant_reference_id` | **Your own order ID**, echoed back from the `create-order` call — use this to look up the order in your system |
| `data.tracking_url` | A public link you can forward directly to your customer so they can track their rider in real time |
| `data.courier` | Driver details — `null` for events that fire before a driver is assigned (e.g. `order.failed`) |
| `data.metadata` | Any custom key/value object you attached to the order at creation time, echoed back |

---

## The signature header

Yonne signs every webhook request and includes the signature in the `X-Yonne-Signature` header:

```http
POST /webhooks/yonne HTTP/1.1
Content-Type: application/json
X-Yonne-Signature: 3b4f2c1d8e...
```

The value is the HMAC-SHA256 hex digest of the **raw request body**, computed using your `webhook_secret` as the key.

---

## Verifying the signature

**Always verify the signature before processing the payload.** Reject any request where the signature does not match.

> **Critical: read raw bytes first.** Compute the HMAC over the exact raw bytes Yonne sent. If you parse the JSON body first and then re-serialize it, the byte sequence will differ and the signature will never match — even for legitimate requests.

<CodeGroup>
```javascript Node.js (Express)
const crypto = require('crypto');

app.post('/webhooks/yonne', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-yonne-signature'];
  const expected = crypto
    .createHmac('sha256', process.env.YONNE_WEBHOOK_SECRET)
    .update(req.body) // req.body must be a raw Buffer — use express.raw(), not express.json()
    .digest('hex');

  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const event = JSON.parse(req.body);

  // Acknowledge immediately, process asynchronously
  res.status(200).send('OK');
  processEventAsync(event);
});
```

```php PHP
<?php
$rawBody = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_YONNE_SIGNATURE'] ?? '';
$expected = hash_hmac('sha256', $rawBody, $_ENV['YONNE_WEBHOOK_SECRET']);

if (!hash_equals($expected, $signature)) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode($rawBody, true);

// Acknowledge immediately, process asynchronously
http_response_code(200);
echo 'OK';
processEventAsync($event);
```

```python Python (Flask)
import hmac
import hashlib
import os
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/webhooks/yonne', methods=['POST'])
def webhook():
    raw_body = request.get_data()  # raw bytes — do NOT use request.json here
    signature = request.headers.get('X-Yonne-Signature', '')
    expected = hmac.new(
        os.environ['YONNE_WEBHOOK_SECRET'].encode('utf-8'),
        raw_body,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        return {'error': 'Invalid signature'}, 401

    event = request.get_json(force=True)

    # Acknowledge immediately, process asynchronously
    process_event_async(event)
    return {'ok': True}, 200
```
</CodeGroup>

> **Use constant-time comparison.** Always use `timingSafeEqual` (Node.js) or `hmac.compare_digest` (Python) / `hash_equals` (PHP) when comparing signatures. A regular string equality check is vulnerable to timing attacks.

---

## Best practices

### Fast ACKs — respond immediately, process asynchronously

Your endpoint must return any `2xx` status code within **10 seconds**. Yonne does not use the response body. If your processing logic (database writes, sending notifications, calling third-party APIs) might take longer, acknowledge first and do the work in a background job:

```javascript
app.post('/webhooks/yonne', express.raw({ type: 'application/json' }), (req, res) => {
  // ... verify signature ...

  res.status(200).send('OK'); // acknowledge fast
  processEventAsync(JSON.parse(req.body)); // do your work after
});
```

### Idempotency — handle duplicates gracefully

The same event can be delivered more than once (e.g. if your server returned `200` but the network dropped before Yonne received the response). Use the combination of `order_id` + `event` as a deduplication key:

```javascript
const key = `${event.data.order_id}:${event.event}`;

if (await alreadyProcessed(key)) {
  return; // safe to skip — already handled
}

await markAsProcessed(key);
// ... process the event
```

---

## Delivery guarantees & retries

Yonne guarantees at-least-once delivery. If your endpoint is unavailable or returns a non-`2xx` status, Yonne will retry with exponential backoff:

| Attempt | Delay after failure |
|---|---|
| 1st retry | 1 minute |
| 2nd retry | 15 minutes |
| 3rd retry | 60 minutes |
| After 3rd failure | Marked `failed` — no further retries |

| Scenario | What happens |
|---|---|
| Your endpoint returns `2xx` | Marked `delivered` — no retries |
| Your endpoint returns `4xx` / `5xx` | Retried up to 3 times with backoff |
| Your endpoint times out (> 10 s) | Treated as failure — retried |
| Yonne server restarts mid-delivery | Event stays `pending` in the database and is picked up on the next scheduler tick |
| All 3 retries exhausted | Marked `failed` — visible in your webhook event log for manual inspection |

---

## Testing your handler in development

Use the simulate-status endpoint with your test API key to trigger webhook deliveries without waiting for a real rider:

```bash
curl --request POST "https://api.yonne.app/api/v1/external/test/simulate-status" \
  --header "X-API-Key: yonne_test_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --data '{ "order_id": "ORD-123456", "status": "In Transit" }'
```
