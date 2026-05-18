---
title: "Capacity & Weight"
description: "Understand how Yonne selects vehicle classes based on shipment weight, and why ERR_NO_CAPACITY_AVAILABLE happens."
---

# Capacity & Weight

Yonne uses **capacity-qualified dispatch** — the platform does not simply assign the nearest rider. It first filters the rider pool by vehicle capacity to ensure your shipment can actually be carried.

---

## How dispatch filters riders

When an order is created, Yonne runs this sequence:

1. Find all online riders
2. Filter by courier affiliation
3. Filter by vehicle capacity — riders whose vehicle cannot carry the shipment are excluded
4. Apply vehicle-class rules (see Bike Rule below)
5. Prefer the smallest valid vehicle class
6. Sort remaining candidates by distance and assign the nearest

---

## How weight is calculated

Yonne computes `required_weight_kg` from your inputs:

| Input | Value used |
|---|---|
| `weight_estimate` only | Upper bound of the bucket (see table below) |
| `weight_estimate` + `total_weight_kg` | The **greater** of the two values |

### Weight bucket upper bounds

| `weight_estimate` | `required_weight_kg` used for routing |
|---|---|
| `0-5kg` | 5 |
| `5-10kg` | 10 |
| `10-20kg` | 20 |
| `20-50kg` | 50 |
| `50kg+` | > 50 (use `total_weight_kg` for accuracy) |

For any shipment above 50 kg, always send `total_weight_kg` with the actual weight so routing and pricing agree on the correct vehicle class.

---

## The bike rule

**BIKE riders are excluded for any shipment with `required_weight_kg` above 50.**

This is a hard system rule. Even if a specific bike rider has a custom capacity override, they will not be assigned to shipments above the threshold.

If your product weighs 55 kg, only `CAR`, `VAN`, or `TRUCK` class vehicles are eligible.

---

## The quote response tells you the vehicle class

Call `POST /api/v1/external/quote` before creating an order. The response includes the routing recommendation:

```json
{
  "success": true,
  "delivery_fee": 18500,
  "currency": "MWK",
  "suggested_vehicle_class": "VAN",
  "required_weight_kg": 95,
  "service_id": "van_std",
  "eta": "38m"
}
```

Surface `suggested_vehicle_class` in your checkout UI so customers know what type of vehicle to expect.

---

## What happens when no vehicle is available

If no online rider has sufficient capacity, the create-order call returns `422`:

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

### What to do

- **Do not silently downgrade** the vehicle class and retry — that changes the shipment contract.
- **Do not retry in a tight loop** — capacity will not appear instantly.
- **Surface a clear message** to your customer: "No delivery vehicles are available for this shipment right now. Please try again in a few minutes or contact support."
- Keep the order in a `pending` or `failed` state in your system until the merchant confirms they want to retry.

---

## Heavy-order example

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/create-order" \
  --header "X-API-Key: yonne_live_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: order-APPL-001-attempt-1" \
  --data '{
    "delivery_address": "Area 18, Lilongwe",
    "receiver_name": "John Phiri",
    "receiver_phone": "+265991234567",
    "delivery_fee": 18500,
    "delivery_lat": -13.954,
    "delivery_lng": 33.792,
    "item_name": "Commercial freezer",
    "product_type": "Appliance",
    "weight_estimate": "50kg+",
    "total_weight_kg": 95
  }'
```

```javascript Node.js
await fetch("https://api.yonne.app/api/v1/external/create-order", {
  method: "POST",
  headers: {
    "X-API-Key": process.env.YONNE_API_KEY,
    "Content-Type": "application/json",
    "Idempotency-Key": "order-APPL-001-attempt-1"
  },
  body: JSON.stringify({
    delivery_address: "Area 18, Lilongwe",
    receiver_name: "John Phiri",
    receiver_phone: "+265991234567",
    delivery_fee: 18500,
    delivery_lat: -13.954,
    delivery_lng: 33.792,
    item_name: "Commercial freezer",
    product_type: "Appliance",
    weight_estimate: "50kg+",
    total_weight_kg: 95
  })
});
```

```python Python
import requests, os

requests.post(
    "https://api.yonne.app/api/v1/external/create-order",
    headers={
        "X-API-Key": os.environ["YONNE_API_KEY"],
        "Content-Type": "application/json",
        "Idempotency-Key": "order-APPL-001-attempt-1"
    },
    json={
        "delivery_address": "Area 18, Lilongwe",
        "receiver_name": "John Phiri",
        "receiver_phone": "+265991234567",
        "delivery_fee": 18500,
        "delivery_lat": -13.954,
        "delivery_lng": 33.792,
        "item_name": "Commercial freezer",
        "product_type": "Appliance",
        "weight_estimate": "50kg+",
        "total_weight_kg": 95
    }
)
```
</CodeGroup>
