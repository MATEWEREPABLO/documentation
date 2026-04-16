# Guide: Create Your First External API Order

This guide shows the normal merchant flow from validation to order creation.

## Step 1. Validate the API key

Call:

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

## Step 2. Request a quote

Call:

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
  "environment": "live",
  "breakdown": {
    "courier_base_rate": 450,
    "platform_fee_per_km": 150,
    "distance_km": 3.9,
    "courier_base_cost": 1755,
    "courier_weighted_cost": 1755,
    "courier_total": 1755,
    "platform_fee": 585,
    "weight_multiplier": 1,
    "speed_multiplier": 1,
    "subtotal": 2340,
    "minimum_amount": 2500,
    "minimum_applied": false,
    "total_amount": 2340
  }
}
```

## Step 3. Create the order

Call:

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
  "pickup_address": "Area 47, Lilongwe",
  "delivery_address": "Mchesi, Lilongwe",
  "merchant_reference_id": "WEB-100245",
  "metadata": {
    "channel": "woocommerce"
  },
  "environment": "live"
}
```

## Step 4. Track the order

Use:

```http
GET /api/v1/external/orders/{order_id}
GET /api/v1/external/orders/{order_id}/tracking
```

Use `/tracking` when you want rider coordinates and ETA. Use `/orders/{order_id}` when you want the broader order snapshot.

## Step 5. Cancel if needed

If the order has not reached pickup, transit, or completion:

```http
POST /api/v1/external/orders/{order_id}/cancel
```

If cancellation is allowed, the response includes the updated wallet balance.

## Important integration notes

- Always use `Idempotency-Key` on order creation.
- Treat `delivery_fee` as required input for create-order.
- Use the same canonical snake_case field style throughout your integration.
- Save both `order_id` and `tracking_id`.
