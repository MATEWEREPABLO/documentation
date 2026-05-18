---
title: "Event Catalog"
description: "Every webhook event Yonne sends, with full payload examples for each."
---

# Event Catalog

Yonne sends a signed HTTP `POST` to your registered webhook URL whenever an order changes state. This page lists every event type and shows the exact payload shape you'll receive.

See [Webhook Setup & Security](/docs/webhooks/setup-and-security) for how to register your endpoint and verify signatures.

---

## Event envelope

Every event uses the same outer envelope:

```json
{
  "event": "order.status_updated",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "merchant_reference_id": "WEB-100245",
  "status": "In Transit",
  "metadata": {
    "channel": "woocommerce"
  },
  "timestamp": "2025-05-18T10:32:00Z"
}
```

| Field | Type | Description |
|---|---|---|
| `event` | string | The event type (see catalog below) |
| `order_id` | string | Yonne's internal order identifier |
| `tracking_id` | string | Customer-facing tracking identifier |
| `merchant_reference_id` | string | Your own order ID, echoed back from create-order |
| `status` | string | Current order status |
| `metadata` | object | Metadata you sent on create-order, echoed back |
| `timestamp` | ISO 8601 | When the event occurred |

---

## Events

### `order.status_updated`

Fired whenever the order status changes. This is the most common event — your handler will receive it at each stage of the delivery lifecycle.

```json
{
  "event": "order.status_updated",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "merchant_reference_id": "WEB-100245",
  "status": "In Transit",
  "metadata": { "channel": "woocommerce" },
  "timestamp": "2025-05-18T10:32:00Z"
}
```

**Status values you'll see in this event:**

| Status | Meaning |
|---|---|
| `searching` | Yonne is looking for an available rider |
| `assigned` | A rider has accepted the order |
| `picked_up` | Rider has collected the item from your pickup location |
| `in_transit` | Rider is en route to the customer |
| `delivered` | Order successfully handed to the customer |
| `completed` | Delivery confirmed and finalized |

---

### `order.created`

Fired immediately after a successful `create-order` call. Useful for audit logging and confirming that Yonne received the order.

```json
{
  "event": "order.created",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "merchant_reference_id": "WEB-100245",
  "status": "searching",
  "delivery_fee": 7350,
  "currency": "MWK",
  "metadata": { "channel": "woocommerce" },
  "timestamp": "2025-05-18T10:00:00Z"
}
```

---

### `order.cancelled`

Fired when an order is cancelled — either by your system via the cancel endpoint, or by Yonne operations.

```json
{
  "event": "order.cancelled",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "merchant_reference_id": "WEB-100245",
  "status": "cancelled",
  "metadata": { "channel": "woocommerce" },
  "timestamp": "2025-05-18T10:15:00Z"
}
```

When you receive this event, update your internal order to `cancelled` and trigger any refund or retry logic your business requires.

---

### `order.delivered`

Fired when the order reaches the `delivered` status. This is a good trigger for post-delivery actions: sending a customer review request, releasing escrow, or updating your OMS.

```json
{
  "event": "order.delivered",
  "order_id": "ORD-123456",
  "tracking_id": "YON-TRK-123456",
  "merchant_reference_id": "WEB-100245",
  "status": "delivered",
  "metadata": { "channel": "woocommerce" },
  "timestamp": "2025-05-18T11:02:00Z"
}
```

---

## Handling unknown event types

New event types may be added to this catalog over time. Your handler should log and ignore events it doesn't recognize rather than throwing an error — this prevents future event additions from breaking your integration.

```javascript Node.js
function handleWebhookEvent(event) {
  switch (event.event) {
    case "order.created": /* ... */ break;
    case "order.status_updated": /* ... */ break;
    case "order.cancelled": /* ... */ break;
    case "order.delivered": /* ... */ break;
    default:
      console.log("Unknown event type — ignoring:", event.event);
  }
}
```

---

## Retry behavior

If your endpoint returns a non-`2xx` status, Yonne will retry delivery with exponential backoff. Design your handler to be idempotent — receiving the same event twice should not cause duplicate side effects.

See [Webhook Setup & Security](/docs/webhooks/setup-and-security) for the deduplication pattern.
