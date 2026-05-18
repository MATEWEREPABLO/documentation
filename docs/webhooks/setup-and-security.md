---
title: "Webhook Setup & Security"
description: "Configure your Yonne webhook endpoint and verify every payload with HMAC-SHA256 signature validation."
---

# Webhook Setup & Security

Yonne delivers order status updates to your server via webhooks. Every payload is signed with HMAC-SHA256 — you must verify the signature on every request to ensure the payload came from Yonne and hasn't been tampered with.

---

## Setting up your endpoint

1. Build an HTTP `POST` endpoint on your server (e.g. `https://yourapp.com/webhooks/yonne`).
2. Register the URL in your Yonne merchant dashboard.
3. Copy the **webhook secret** shown in the dashboard — you'll use it to verify signatures.
4. Return `200 OK` for every valid request, even if you've already processed it.

Your endpoint must be reachable from the public internet. Yonne does not support localhost endpoints in production — use a tunnel like ngrok during development.

---

## The signature header

Yonne signs every webhook payload and sends the signature in the `X-Yonne-Signature` header:

```http
POST /webhooks/yonne HTTP/1.1
Content-Type: application/json
X-Yonne-Signature: sha256=3d2f5e...
```

The value is `sha256=` followed by the HMAC-SHA256 hex digest of the raw request body, computed using your webhook secret as the key.

---

## Verifying the signature

**Always verify the signature before processing the payload.** Reject any request where the signature does not match.

<CodeGroup>
```javascript Node.js (Express)
const crypto = require("crypto");

function verifyYonneSignature(req, secret) {
  const signature = req.headers["x-yonne-signature"];
  if (!signature) return false;

  const expected = "sha256=" + crypto
    .createHmac("sha256", secret)
    .update(req.rawBody) // must be the raw Buffer, not parsed JSON
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// Express webhook route — use express.raw() to preserve the raw body
app.post("/webhooks/yonne", express.raw({ type: "application/json" }), (req, res) => {
  const secret = process.env.YONNE_WEBHOOK_SECRET;

  if (!verifyYonneSignature(req, secret)) {
    return res.status(401).json({ error: "Invalid signature" });
  }

  const event = JSON.parse(req.body);
  handleWebhookEvent(event);
  res.status(200).json({ received: true });
});
```

```python Python (Flask)
import hmac
import hashlib
import os
from flask import Flask, request, jsonify

app = Flask(__name__)

def verify_yonne_signature(payload_bytes, signature_header, secret):
    if not signature_header:
        return False
    expected = "sha256=" + hmac.new(
        secret.encode("utf-8"),
        payload_bytes,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)

@app.route("/webhooks/yonne", methods=["POST"])
def yonne_webhook():
    secret = os.environ["YONNE_WEBHOOK_SECRET"]
    signature = request.headers.get("X-Yonne-Signature", "")

    if not verify_yonne_signature(request.data, signature, secret):
        return jsonify({"error": "Invalid signature"}), 401

    event = request.get_json()
    handle_webhook_event(event)
    return jsonify({"received": True}), 200
```
</CodeGroup>

<Warning>
  Always use a constant-time comparison (`timingSafeEqual` / `hmac.compare_digest`) when comparing signatures. A regular string equality check is vulnerable to timing attacks.
</Warning>

---

## Processing the event

After verifying the signature, parse and handle the event:

<CodeGroup>
```javascript Node.js
async function handleWebhookEvent(event) {
  switch (event.event) {
    case "order.status_updated":
      await db.orders.updateStatus(event.order_id, event.status);
      if (event.status === "delivered") {
        await notifications.sendDeliveryConfirmation(event.order_id);
      }
      break;

    case "order.cancelled":
      await db.orders.markCancelled(event.order_id);
      break;

    default:
      console.log("Unhandled event type:", event.event);
  }
}
```

```python Python
def handle_webhook_event(event):
    if event["event"] == "order.status_updated":
        db.orders.update_status(event["order_id"], event["status"])
        if event["status"] == "delivered":
            notifications.send_delivery_confirmation(event["order_id"])

    elif event["event"] == "order.cancelled":
        db.orders.mark_cancelled(event["order_id"])

    else:
        print(f"Unhandled event: {event['event']}")
```
</CodeGroup>

---

## Idempotent webhook processing

Yonne may deliver the same event more than once due to network retries or internal replay. Your handler must be idempotent — processing the same event twice should have no side effects.

Use the combination of `order_id` + `event` + `status` as a deduplication key:

```javascript Node.js
// Skip if already processed
const alreadyProcessed = await db.webhookEvents.exists({
  order_id: event.order_id,
  event: event.event,
  status: event.status
});

if (alreadyProcessed) {
  return; // Safe to skip
}

await db.webhookEvents.insert({ order_id: event.order_id, event: event.event, status: event.status });
// ... process the event
```

---

## Responding correctly

| Response | Meaning |
|---|---|
| `200 OK` | Event received and processed (or safely skipped) |
| `401` | Signature invalid — Yonne will log but may retry |
| `5xx` | Processing failed — Yonne will retry |

Always return `200` for duplicate events you've already processed. If you return a non-2xx status, Yonne treats it as a failure and will retry.

---

## Testing your handler in development

Use the simulate-status endpoint with your test key to trigger webhook deliveries without waiting for a real rider:

```bash
curl --request POST "https://api.yonne.app/api/v1/external/test/simulate-status" \
  --header "X-API-Key: yonne_test_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --data '{ "order_id": "ORD-123456", "status": "In Transit" }'
```

Use [ngrok](https://ngrok.com) to expose your local server during development:

```bash
ngrok http 3000
# Then register https://abc123.ngrok.io/webhooks/yonne in your dashboard
```
