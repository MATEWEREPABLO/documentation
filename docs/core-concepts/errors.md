---
title: "Error Catalog"
description: "Every Yonne error code, what causes it, and exactly what to do next."
---

All Yonne error responses follow this shape:

```json
{
  "success": false,
  "error": "ErrorLabel",
  "message": "Human-readable explanation"
}
```

Some errors include extra fields like `balance`, `required`, `required_weight_kg`, `status`, or `retry_after`.

---

## Authentication errors

### `401 Unauthorized`

```json
{
  "success": false,
  "error": "Unauthorized",
  "message": "API key required. Send X-API-Key header or Authorization: Bearer <key>."
}
```

**Causes:** `X-API-Key` header missing · key malformed · key inactive · wrong environment prefix

**Fix:** Confirm the header is present. Validate the key with `GET /api/v1/external/validate`. Check your dashboard for key status.

---

## Coordinate and pickup errors

### `400 Missing coordinates`

```json
{
  "success": false,
  "error": "Missing coordinates",
  "message": "delivery_lat and delivery_lng are required."
}
```

**Fix:** Ensure both `delivery_lat` and `delivery_lng` are present in your request body.

---

### `400 Invalid coordinates`

```json
{
  "success": false,
  "error": "Invalid coordinates",
  "message": "Coordinates must be valid numbers."
}
```

**Fix:** Validate that coordinates are numeric before sending. Reject string inputs like `"13.98N"`.

---

### `400 Pickup not set`

```json
{
  "success": false,
  "error": "Pickup not set",
  "message": "Merchant pickup location is not set. Set it in your dashboard."
}
```

**Fix:** Go to your merchant dashboard and configure your store's pickup address and coordinates. No order can be created without a pickup location.

---

### `400 Invalid pickup`

**Causes:** Pickup coordinates are `(0, 0)` or out of range.

**Fix:** Update the merchant pickup location to valid coordinates in the dashboard.

---

### `400 Invalid delivery`

**Causes:** Destination coordinates are `(0, 0)` or out of range.

**Fix:** Collect a valid customer delivery location before calling create-order.

---

### `400 Invalid distance`

```json
{
  "success": false,
  "error": "Invalid distance",
  "message": "Pickup and delivery are the same or too close. Set your store pickup in the dashboard and ensure delivery uses the customer's location."
}
```

**Fix:** Verify that pickup and delivery coordinates are not the same point.

---

## Pricing and wallet errors

### `400 Calculation failed`

**Causes:** Internal pricing validation failure · malformed weight or speed inputs.

**Fix:** Inspect the `message` field. Verify `weight_estimate` and `delivery_speed` are valid values. Contact support if it persists.

---

### `402 Insufficient Funds`

```json
{
  "success": false,
  "error": "Insufficient Funds",
  "message": "Wallet balance is insufficient for this delivery. Please top up.",
  "balance": 1500,
  "required": 7350
}
```

**Causes:** Merchant wallet balance is below `delivery_fee`.

**Fix:** Top up the wallet, confirm the new balance with `GET /api/v1/external/wallet`, then retry. Do not retry blindly in a loop.

See the full top-up flow in [Wallet Management](/docs/integration-workflows/wallet-management).

---

## Capacity errors

### `422 ERR_NO_CAPACITY_AVAILABLE`

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

**Causes:** No online rider can carry the shipment at the computed `required_weight_kg`. BIKE riders are excluded above 50 kg.

**Fix:**
- Show a user-facing message: "No delivery vehicles available right now — please try again shortly."
- Do not silently downgrade vehicle class requirements.
- Do not retry in a tight loop.
- Keep the order in a `pending` state until the merchant manually retries.

See [Capacity & Weight](/docs/core-concepts/capacity-and-weight) for the full dispatch model.

---

## Order lifecycle errors

### `404 Order not found`

```json
{
  "success": false,
  "error": "Order not found"
}
```

**Causes:** Wrong `order_id` · order belongs to a different merchant · querying a live order with a test key (or vice versa).

**Fix:** Confirm `order_id` is correct. Check you're using the right environment key.

---

### `400 Order cannot be cancelled at this stage`

```json
{
  "success": false,
  "error": "Order cannot be cancelled at this stage",
  "status": "In Transit"
}
```

**Causes:** Order is already picked up, in transit, delivered, completed, or cancelled.

**Fix:** Do not retry cancellation. Handle this as a terminal business-state error in your UI.

---

### `400 POD is available only after delivery`

**Causes:** Proof of delivery requested before the order is in a terminal delivery state.

**Fix:** Poll order status first. Request POD only after `delivered` or `completed` status.

---

## Test simulation errors

### `400 order_id and status are required`

**Causes:** Missing fields in `POST /api/v1/external/test/simulate-status`.

**Fix:** Send both `order_id` and `status` in the request body.

---

### `500 Webhook dispatch failed`

**Causes:** Simulation updated order status but the internal webhook dispatch failed.

**Fix:** Retry the simulation if safe. Check your webhook receiver logs.

---

## Retry reference

| Error | Retry? |
|---|---|
| Network timeout on create-order | Yes — same `Idempotency-Key` |
| Any `GET` request | Yes — always safe |
| `401 Unauthorized` | No — fix credentials |
| `400` validation errors | No — fix the payload |
| `402 Insufficient Funds` | Only after topping up wallet |
| `422 ERR_NO_CAPACITY_AVAILABLE` | Only after capacity becomes available |
| `400` lifecycle errors (cancel, POD) | No — terminal state |

---

## What to log on every failure

Always log:
- Request path
- `merchant_reference_id`
- `order_id`
- `Idempotency-Key`
- HTTP status code
- `error` and `message` fields
- Full response body when safe to store
