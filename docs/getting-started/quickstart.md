---
title: "Quickstart"
description: "Make your first Yonne API call in under 5 minutes. You'll validate your key, get a delivery quote, and create a test order."
---

# Quickstart

You'll need a Yonne API key. Use a `yonne_test_` key while building — it behaves identically to production without affecting your live wallet.

<Note>
  Get your API key from [merchant.yonne.app](https://merchant.yonne.app). If you don't have an account yet, sign up there first.
</Note>

---

## Step 1 — Validate your API key

Before you build anything, confirm your key works and your pickup location is configured.

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_test_xxxxxxxxx"
```

```javascript Node.js
const response = await fetch("https://api.yonne.app/api/v1/external/validate", {
  headers: { "X-API-Key": "yonne_test_xxxxxxxxx" }
});
const data = await response.json();
console.log(data);
```

```python Python
import requests

response = requests.get(
    "https://api.yonne.app/api/v1/external/validate",
    headers={"X-API-Key": "yonne_test_xxxxxxxxx"}
)
print(response.json())
```
</CodeGroup>

**Success response:**

```json
{
  "success": true,
  "balance": 125000,
  "currency": "MWK",
  "hasPickup": true,
  "pickupAddress": "Area 3, Lilongwe",
  "pickupLatitude": -13.9626,
  "pickupLongitude": 33.7741,
  "environment": "test"
}
```

<Warning>
  If `hasPickup` is `false`, go to your dashboard and set your store's pickup address before continuing. Order creation will fail without it.
</Warning>

---

## Step 2 — Get a delivery quote

Yonne calculates the delivery fee based on the distance between your pickup location and the customer's exact delivery coordinates. Your frontend must request the customer's location using the browser Geolocation API and pass those coordinates to your server, which then calls `/quote`.

<Note>
  A typed address alone is not enough — Yonne requires `delivery_lat` and `delivery_lng`. Use the browser Geolocation API or a geocoding service to convert addresses to coordinates before calling `/quote`.
</Note>

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/quote" \
  --header "X-API-Key: yonne_test_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --data '{
    "delivery_lat": -13.9831,
    "delivery_lng": 33.7812,
    "product_type": "Package",
    "weight_estimate": "0-5kg",
    "delivery_speed": "standard"
  }'
```

```javascript Node.js
const response = await fetch("https://api.yonne.app/api/v1/external/quote", {
  method: "POST",
  headers: {
    "X-API-Key": "yonne_test_xxxxxxxxx",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    delivery_lat: -13.9831,
    delivery_lng: 33.7812,
    product_type: "Package",
    weight_estimate: "0-5kg",
    delivery_speed: "standard"
  })
});
const data = await response.json();
console.log(data);
```

```python Python
import requests

response = requests.post(
    "https://api.yonne.app/api/v1/external/quote",
    headers={
        "X-API-Key": "yonne_test_xxxxxxxxx",
        "Content-Type": "application/json"
    },
    json={
        "delivery_lat": -13.9831,
        "delivery_lng": 33.7812,
        "product_type": "Package",
        "weight_estimate": "0-5kg",
        "delivery_speed": "standard"
    }
)
print(response.json())
```
</CodeGroup>

**Success response:**

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
  "environment": "test"
}
```

Save the `delivery_fee` — you'll pass it directly into the create-order call.

---

## Step 3 — Create your first order

Now dispatch a rider. The `Idempotency-Key` header is **required** — it prevents duplicate orders if your request is retried.

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/create-order" \
  --header "X-API-Key: yonne_test_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: quickstart-order-001" \
  --data '{
    "delivery_address": "Mchesi, Lilongwe",
    "receiver_name": "Jane Banda",
    "receiver_phone": "+265991234567",
    "delivery_fee": 7350,
    "delivery_lat": -13.9831,
    "delivery_lng": 33.7812,
    "item_name": "Printer cartridge",
    "product_type": "Package",
    "weight_estimate": "0-5kg",
    "merchant_reference_id": "QS-001"
  }'
```

```javascript Node.js
const response = await fetch("https://api.yonne.app/api/v1/external/create-order", {
  method: "POST",
  headers: {
    "X-API-Key": "yonne_test_xxxxxxxxx",
    "Content-Type": "application/json",
    "Idempotency-Key": "quickstart-order-001"
  },
  body: JSON.stringify({
    delivery_address: "Mchesi, Lilongwe",
    receiver_name: "Jane Banda",
    receiver_phone: "+265991234567",
    delivery_fee: 7350,
    delivery_lat: -13.9831,
    delivery_lng: 33.7812,
    item_name: "Printer cartridge",
    product_type: "Package",
    weight_estimate: "0-5kg",
    merchant_reference_id: "QS-001"
  })
});
const data = await response.json();
console.log(data);
```

```python Python
import requests

response = requests.post(
    "https://api.yonne.app/api/v1/external/create-order",
    headers={
        "X-API-Key": "yonne_test_xxxxxxxxx",
        "Content-Type": "application/json",
        "Idempotency-Key": "quickstart-order-001"
    },
    json={
        "delivery_address": "Mchesi, Lilongwe",
        "receiver_name": "Jane Banda",
        "receiver_phone": "+265991234567",
        "delivery_fee": 7350,
        "delivery_lat": -13.9831,
        "delivery_lng": 33.7812,
        "item_name": "Printer cartridge",
        "product_type": "Package",
        "weight_estimate": "0-5kg",
        "merchant_reference_id": "QS-001"
    }
)
print(response.json())
```
</CodeGroup>

**Success response:**

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
  "environment": "test"
}
```

**Save both `order_id` and `tracking_id`** in your database. You'll use `order_id` for API operations (cancel, status checks) and `tracking_id` for customer-facing tracking links.

---

## You're done

You've just validated your key, gotten a quote, and created your first order. Here's what to build next:

- **Wire up checkout** → [E-commerce Checkout Workflow](/docs/integration-workflows/ecommerce-checkout)
- **Show live tracking to customers** → [Real-time Tracking Workflow](/docs/integration-workflows/real-time-tracking)
- **Handle webhooks for order updates** → [Webhook Setup & Security](/docs/webhooks/setup-and-security)
