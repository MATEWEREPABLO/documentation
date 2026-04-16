# API overview

Use the Yonne External API to validate merchant credentials, quote deliveries, create orders, track fulfillment, cancel eligible orders, inspect wallet state, and test webhook flows.

This section is the main API entry point. Read this page first, then move into authentication, webhooks, the error catalog, and the endpoint reference generated from `api-1.json`.

## Base URL

```text
https://api.yonne.app
```

The same host is used for both sandbox and production. Your API key prefix determines the environment:

- `yonne_test_...` for sandbox
- `yonne_live_...` for production

## Recommended request flow

1. Validate the API key with `GET /api/v1/external/validate`.
2. Confirm the merchant pickup location is configured.
3. Quote the delivery with `POST /api/v1/external/quote`.
4. Create the order with `POST /api/v1/external/create-order`.
5. Track status with the order and tracking endpoints.
6. Handle webhook updates in your backend.

## First order walkthrough

### 1. Validate the API key

```http
GET /api/v1/external/validate
X-API-Key: yonne_live_xxxxxxxxx
```

You should receive merchant context including:

- `balance`
- `hasPickup`
- `pickupAddress`
- `pickupLatitude`
- `pickupLongitude`

If `hasPickup` is `false`, configure the merchant pickup location before continuing.

### 2. Request a quote

```http
POST /api/v1/external/quote
Content-Type: application/json
X-API-Key: yonne_live_xxxxxxxxx
```

Example request:

```json
{
  "delivery_lat": -13.9831,
  "delivery_lng": 33.7812,
  "product_type": "Package",
  "weight_estimate": "0-5kg",
  "delivery_speed": "standard"
}
```

Example response:

```json
{
  "success": true,
  "price": 7350,
  "delivery_fee": 7350,
  "currency": "MWK",
  "eta": "24m",
  "eta_minutes": 24,
  "service_id": "bike_std",
  "suggested_vehicle_class": "BIKE",
  "required_weight_kg": 5,
  "distance_km": 3.9,
  "environment": "live"
}
```

### 3. Create the order

```http
POST /api/v1/external/create-order
Content-Type: application/json
X-API-Key: yonne_live_xxxxxxxxx
Idempotency-Key: order-100245-attempt-1
```

Example request:

```json
{
  "delivery_address": "Mchesi, Lilongwe",
  "receiver_name": "Jane Banda",
  "receiver_phone": "+265991234567",
  "delivery_fee": 7350,
  "delivery_lat": -13.9831,
  "delivery_lng": 33.7812,
  "pickup_lat": -13.9626,
  "pickup_lng": 33.7741,
  "item_name": "Printer cartridge",
  "product_type": "Package",
  "weight_estimate": "0-5kg",
  "merchant_reference_id": "WEB-100245",
  "metadata": {
    "channel": "woocommerce"
  }
}
```

Example response:

```json
{
  "success": true,
  "tracking_id": "YON-TRK-123456",
  "order_id": "ORD-123456",
  "status": "searching",
  "tracking_link": "https://yonne.app/track/YON-TRK-123456",
  "delivery_fee": 7350,
  "wallet_balance": 117650,
  "currency": "MWK",
  "environment": "live"
}
```

## Important integration rules

- Always send `Idempotency-Key` on order creation.
- Treat `delivery_fee` as a required create-order input.
- Use canonical `snake_case` field names.
- Save both `order_id` and `tracking_id`.
- Separate sandbox and production keys in your systems and logs.

## What belongs where

- Use `docs/api-reference/authentication` for request authentication details.
- Use `docs/api-reference/webhooks` for webhook delivery behavior.
- Use `docs/api-reference/error-catalog` for business and integration failure reference.
- Use the endpoint docs generated from `api-1.json` for the full operation-level contract, including the Mintlify method styling for `GET`, `POST`, and other operations.
