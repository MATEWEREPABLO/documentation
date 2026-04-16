# Error catalog

This document lists the most important business and integration errors for the Yonne External API, what they mean, and what client applications should do next.

## How to read errors

Most error responses follow this shape:

```json
{
  "success": false,
  "error": "Some error label",
  "message": "Human-readable explanation"
}
```

Some errors also include extra fields such as:

- `balance`
- `required`
- `required_weight_kg`
- `status`
- `retry_after`

## Authentication errors

### `401 Unauthorized`

Typical causes:

- `X-API-Key` header missing
- API key malformed
- API key inactive
- wrong environment key format

Example:

```json
{
  "success": false,
  "error": "Unauthorized",
  "message": "API key required. Send X-API-Key header or Authorization: Bearer <key>."
}
```

Recommended recovery:

- confirm `X-API-Key` is being sent
- verify the key in merchant dashboard
- call `GET /api/v1/external/validate`

## Pickup and coordinate errors

### `400 Missing coordinates`

Typical causes:

- `delivery_lat` missing
- `delivery_lng` missing

Example:

```json
{
  "success": false,
  "error": "Missing coordinates",
  "message": "delivery_lat and delivery_lng are required."
}
```

Recommended recovery:

- validate required destination coordinates before sending

### `400 Invalid coordinates`

Typical causes:

- coordinate values are not numbers
- invalid request payload formatting

Example:

```json
{
  "success": false,
  "error": "Invalid coordinates",
  "message": "Coordinates must be valid numbers."
}
```

Recommended recovery:

- validate numeric coordinates client-side
- ensure lat/lng are not strings with invalid characters

### `400 Pickup not set`

Typical causes:

- merchant has no pickup address saved
- merchant has no pickup coordinates configured

Example:

```json
{
  "success": false,
  "error": "Pickup not set",
  "message": "Merchant pickup location is not set. Set it in your dashboard."
}
```

Recommended recovery:

- configure pickup address and coordinates in dashboard before retrying

### `400 Invalid pickup`

Typical causes:

- merchant pickup is `(0,0)`
- merchant pickup coordinates are out of range

Recommended recovery:

- update merchant pickup location to valid coordinates

### `400 Invalid delivery`

Typical causes:

- destination coordinates are `(0,0)`
- destination coordinates are out of range

Recommended recovery:

- collect a valid customer delivery location before retrying

### `400 Invalid distance`

Typical causes:

- pickup and delivery are effectively the same point

Example:

```json
{
  "success": false,
  "error": "Invalid distance",
  "message": "Pickup and delivery are the same or too close. Set your store pickup in the dashboard and ensure delivery uses the customer's location."
}
```

Recommended recovery:

- ensure pickup and destination are different locations

## Pricing and wallet errors

### `400 Calculation failed`

Typical causes:

- internal pricing validation failure
- courier pricing misconfiguration
- malformed weight or delivery inputs

Recommended recovery:

- inspect the `message`
- verify pricing inputs
- if persistent, contact support

### `402 Insufficient Funds`

Typical causes:

- merchant wallet balance is below `delivery_fee`

Example:

```json
{
  "success": false,
  "error": "Insufficient Funds",
  "message": "Wallet balance is insufficient for this delivery. Please top up.",
  "balance": 1500,
  "required": 7350
}
```

Recommended recovery:

- top up wallet
- call `GET /api/v1/external/wallet`
- retry only after sufficient balance exists

## Capacity-aware dispatch errors

### `422 ERR_NO_CAPACITY_AVAILABLE`

Typical causes:

- no online rider can carry the shipment
- order weight requires `CAR`, `VAN`, or `TRUCK`, but only smaller vehicles are available
- shipment is above the bike threshold and no larger vehicle is online

Example:

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

Recommended recovery:

- do not silently downgrade vehicle requirements
- show a clear retry-later or contact-support message
- reduce shipment size only if operationally valid

### Dispatch rule behind this error

Important routing rule:

- `BIKE` riders are excluded for shipments above **50 kg required weight**

## Order lifecycle errors

### `404 Order not found`

Typical causes:

- wrong `order_id`
- order belongs to a different merchant
- wrong environment key used (`live` vs `test`)

Example:

```json
{
  "success": false,
  "error": "Order not found"
}
```

Recommended recovery:

- confirm the correct order ID
- confirm the correct merchant key and environment

### `400 Order cannot be cancelled at this stage`

Typical causes:

- order already picked up
- order already in transit
- order already delivered, completed, or cancelled

Example:

```json
{
  "success": false,
  "error": "Order cannot be cancelled at this stage",
  "status": "In Transit"
}
```

Recommended recovery:

- do not retry cancellation automatically
- handle as a business-state error in UI

### `400 POD is available only after delivery`

Typical causes:

- proof of delivery requested before order is delivered or completed

Recommended recovery:

- poll order status first
- request POD only after terminal delivery states

## Testing errors

### `400 order_id and status are required`

Typical causes:

- missing fields in `/api/v1/external/test/simulate-status`

Recommended recovery:

- send both `order_id` and `status`

### `500 Webhook dispatch failed`

Typical causes:

- simulation updated the order but webhook dispatch failed internally

Recommended recovery:

- retry test simulation if safe
- inspect webhook receiver and backend logs

## Retry guidance

### Safe to retry

- `GET` requests
- `POST /api/v1/external/create-order` only with the same `Idempotency-Key`
- temporary webhook simulation failures in test workflows

### Do not blindly retry

- `402 Insufficient Funds`
- `422 ERR_NO_CAPACITY_AVAILABLE`
- validation failures like missing or invalid coordinates
- lifecycle-state failures like cancellation after transit

## Best operational logging fields

When handling failures, log:

- request path
- `merchant_reference_id`
- `order_id`
- `Idempotency-Key`
- HTTP status code
- `error`
- `message`
- full response body when safe
