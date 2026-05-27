---
title: "Event Catalog"
description: "Every webhook event Yonne sends, with full payload examples for each."
---

# Event Catalog

Yonne sends a signed HTTP `POST` to your registered webhook URL whenever an order changes state. This page lists every event type and shows the exact payload shape you'll receive.

See [Webhook Setup & Security](/docs/webhooks/setup-and-security) for how to register your endpoint and verify signatures.

---

## Supported events

| Event | When it fires |
|---|---|
| `order.driver_assigned` | A driver accepts and is assigned to the order |
| `order.picked_up` | The driver marks the order as picked up from your location |
| `order.delivered` | The driver marks the order as delivered to the customer |
| `order.failed` | The order exhausted all driver dispatch attempts with no driver found |
| `order.status_changed` | Any other status change not covered by the events above (legacy fallback) |

---

## Event envelope

Every event uses the same outer envelope:

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

| Field | Type | Description |
|---|---|---|
| `event` | string | The event type (see catalog above) |
| `created_at` | ISO 8601 | When the event occurred |
| `data.order_id` | string | Yonne's internal order identifier |
| `data.tracking_number` | string | Customer-facing tracking reference |
| `data.tracking_url` | string | Public tracking link — safe to share directly with your customer |
| `data.merchant_reference_id` | string | Your own order ID, echoed back from `create-order` |
| `data.environment` | string | `live` or `test` |
| `data.delivery_fee` | number | Delivery fee in the smallest currency unit (e.g. tambala for MWK) |
| `data.courier` | object \| null | Driver details — `null` until a driver is assigned (see below) |
| `data.metadata` | object | Custom key/value object you attached at order creation, echoed back |

---

## Events

### `order.driver_assigned`

Fires when a driver accepts and is assigned to the order. This is the first event that includes a populated `courier` object.

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

**Suggested action:** Update your order status to show a driver has been found. Use `tracking_url` to notify your customer so they can follow the rider in real time.

---

### `order.picked_up`

Fires when the driver marks the order as collected from your pickup location.

```json
{
  "event": "order.picked_up",
  "created_at": "2026-05-26T14:45:00.000Z",
  "data": {
    "order_id": "ORD123456",
    "tracking_number": "YON-ABC123",
    "tracking_url": "https://yonne.app/track/YON-ABC123",
    "status": "In Transit",
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

**Suggested action:** Mark the order as "In Transit" in your system and send an estimated delivery notification to your customer.

---

### `order.delivered`

Fires when the driver marks the order as successfully delivered to the customer.

```json
{
  "event": "order.delivered",
  "created_at": "2026-05-26T15:10:00.000Z",
  "data": {
    "order_id": "ORD123456",
    "tracking_number": "YON-ABC123",
    "tracking_url": "https://yonne.app/track/YON-ABC123",
    "status": "Delivered",
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

**Suggested action:** Mark the order as complete, release any escrow, and trigger a post-delivery review request to your customer.

---

### `order.failed`

Fires when the order exhausted all driver dispatch attempts and no driver was found. The `courier` field is `null` for this event.

```json
{
  "event": "order.failed",
  "created_at": "2026-05-26T14:50:00.000Z",
  "data": {
    "order_id": "ORD123456",
    "tracking_number": "YON-ABC123",
    "tracking_url": "https://yonne.app/track/YON-ABC123",
    "status": "Failed",
    "environment": "live",
    "delivery_fee": 3500.0,
    "pickup_address": "Area 18, Lilongwe",
    "delivery_address": "Area 3, Lilongwe",
    "receiver_name": "John Doe",
    "merchant_reference_id": "your-internal-order-id",
    "metadata": {},
    "courier": null
  }
}
```

**Suggested action:** Mark the order as failed in your system, notify the customer, and trigger any retry or refund logic your business requires.

---

### `order.status_changed`

A legacy fallback event that fires on any status transition not covered by the specific events above. Treat it as a general-purpose status update.

```json
{
  "event": "order.status_changed",
  "created_at": "2026-05-26T14:32:00.000Z",
  "data": {
    "order_id": "ORD123456",
    "tracking_number": "YON-ABC123",
    "tracking_url": "https://yonne.app/track/YON-ABC123",
    "status": "Pending",
    "environment": "live",
    "delivery_fee": 3500.0,
    "pickup_address": "Area 18, Lilongwe",
    "delivery_address": "Area 3, Lilongwe",
    "receiver_name": "John Doe",
    "merchant_reference_id": "your-internal-order-id",
    "metadata": {},
    "courier": null
  }
}
```

---

## Handling unknown event types

New event types may be added to this catalog over time. Your handler should log and ignore events it doesn't recognise rather than throwing an error — this prevents future additions from breaking your integration.

```javascript
function handleWebhookEvent(event) {
  switch (event.event) {
    case 'order.driver_assigned': /* ... */ break;
    case 'order.picked_up':       /* ... */ break;
    case 'order.delivered':       /* ... */ break;
    case 'order.failed':          /* ... */ break;
    case 'order.status_changed':  /* ... */ break;
    default:
      console.log('Unknown event type — ignoring:', event.event);
  }
}
```

---

## Retry behavior

If your endpoint returns a non-`2xx` status or times out, Yonne retries with exponential backoff (1 min → 15 min → 60 min, up to 3 retries). Design your handler to be idempotent — receiving the same event twice should not cause duplicate side effects.

See [Webhook Setup & Security](/docs/webhooks/setup-and-security) for the full retry table and deduplication pattern.
