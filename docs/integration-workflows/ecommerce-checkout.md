---
title: "E-commerce Checkout Integration"
description: "A step-by-step guide to integrating Yonne delivery fees and order dispatch into your e-commerce checkout flow."
---

This guide walks you through the exact sequence of API calls needed to add Yonne delivery to your checkout — from capturing the customer's location and displaying a real-time fee, to dispatching a rider the moment they pay.

---

## The complete flow

```
Customer adds item to cart
         │
         ▼
┌─────────────────────────┐
│  Step 1: VALIDATE       │  On startup / merchant onboarding
│  GET /validate          │  Confirm key is active + pickup is set
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Step 2: QUOTE          │  Request customer location via browser
│  POST /quote            │  Geolocation API → get delivery_fee + ETA
└──────────┬──────────────┘
           │
           ▼
  Customer reviews fee
  and confirms order
           │
           ▼
┌─────────────────────────┐
│  Step 3: COLLECT        │  Your payment gateway (Paychangu, Stripe, etc.)
│  Customer pays          │  Capture payment including delivery_fee
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Step 4: CREATE ORDER   │  Server-side, after payment succeeds
│  POST /create-order     │  Dispatch the rider with Idempotency-Key
└──────────┬──────────────┘
           │
           ▼
  Return tracking_link
  to customer confirmation page
```

---

## Step 1 — Validate on startup

Before your checkout can work, confirm your API key is active and your merchant pickup address is configured. Call this once on server startup (or when onboarding a new merchant).

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```

```javascript Node.js
const res = await fetch("https://api.yonne.app/api/v1/external/validate", {
  headers: { "X-API-Key": process.env.YONNE_API_KEY }
});
const merchant = await res.json();

if (!merchant.hasPickup) {
  throw new Error("Merchant pickup location not configured. Set it in the Yonne dashboard.");
}

// Cache these — they won't change often
const pickupCoords = {
  lat: merchant.pickupLatitude,
  lng: merchant.pickupLongitude
};
```

```python Python
import requests, os

merchant = requests.get(
    "https://api.yonne.app/api/v1/external/validate",
    headers={"X-API-Key": os.environ["YONNE_API_KEY"]}
).json()

if not merchant["hasPickup"]:
    raise RuntimeError("Merchant pickup location not configured.")

pickup_coords = {
    "lat": merchant["pickupLatitude"],
    "lng": merchant["pickupLongitude"]
}
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
  "environment": "live"
}
```

<Warning>
  If `hasPickup` is `false`, stop here. Go to your Yonne dashboard and configure the pickup address. Order creation will fail until this is set.
</Warning>

---

## Step 2 — Capture the customer's location, then quote

Yonne calculates the delivery fee based on the real distance between your pickup point and the customer's location. A typed address is not sufficient — you must send precise GPS coordinates (`delivery_lat`, `delivery_lng`).

**Get the customer's coordinates first**, using the browser Geolocation API on your frontend:

```javascript
// Frontend — request browser location, then POST to your own checkout API
navigator.geolocation.getCurrentPosition(
  (position) => {
    const { latitude, longitude } = position.coords;
    fetchDeliveryQuote(latitude, longitude);
  },
  (error) => {
    // Fall back to asking the customer to enter coordinates manually,
    // or use a geocoding service to convert a typed address to lat/lng
    console.error("Location access denied:", error.message);
  }
);
```

Once you have the coordinates, call `POST /api/v1/external/quote` from your server and display the returned `delivery_fee` and `eta` on the checkout page.

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/quote" \
  --header "X-API-Key: yonne_live_xxxxxxxxx" \
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
// Call this from your checkout API route when the customer enters their address
async function getDeliveryQuote(deliveryLat, deliveryLng, cartWeightEstimate) {
  const res = await fetch("https://api.yonne.app/api/v1/external/quote", {
    method: "POST",
    headers: {
      "X-API-Key": process.env.YONNE_API_KEY,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      delivery_lat: deliveryLat,
      delivery_lng: deliveryLng,
      product_type: "Package",
      weight_estimate: cartWeightEstimate, // e.g. "0-5kg"
      delivery_speed: "standard"
    })
  });

  const quote = await res.json();
  if (!quote.success) throw quote;
  return quote;
}

// Returns: { delivery_fee: 7350, eta: "24m", suggested_vehicle_class: "BIKE", ... }
```

```python Python
import requests, os

def get_delivery_quote(delivery_lat, delivery_lng, weight_estimate):
    response = requests.post(
        "https://api.yonne.app/api/v1/external/quote",
        headers={
            "X-API-Key": os.environ["YONNE_API_KEY"],
            "Content-Type": "application/json"
        },
        json={
            "delivery_lat": delivery_lat,
            "delivery_lng": delivery_lng,
            "product_type": "Package",
            "weight_estimate": weight_estimate,
            "delivery_speed": "standard"
        }
    )
    quote = response.json()
    if not quote.get("success"):
        raise Exception(quote)
    return quote
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
  "environment": "live"
}
```

**What to do with this response:**
- Display `delivery_fee` (in MWK) and `eta` on your checkout page.
- Store `delivery_fee` in your checkout session — you'll pass it unchanged to create-order.
- Optionally show `suggested_vehicle_class` ("A bike will deliver your order").

---

## Step 3 — Collect payment

This step happens entirely in your existing payment flow. Yonne is not involved.

Ensure the total your payment gateway charges includes the `delivery_fee` from the quote. When the payment succeeds, proceed to Step 4.

<Note>
  Never create a Yonne order before the payment is confirmed. Create it in your payment success webhook or confirmation callback, not in the checkout page render.
</Note>

---

## Step 4 — Create the order (dispatch the rider)

Call `POST /api/v1/external/create-order` **server-side**, immediately after payment is confirmed. The `Idempotency-Key` header is **required** — generate it from your internal order ID so retries are safe.

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/v1/external/create-order" \
  --header "X-API-Key: yonne_live_xxxxxxxxx" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: order-WEB-100245-attempt-1" \
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
    "merchant_reference_id": "WEB-100245",
    "metadata": {
      "channel": "woocommerce",
      "checkout_session_id": "cs_abc123"
    }
  }'
```

```javascript Node.js
async function dispatchOrder(internalOrder) {
  const res = await fetch("https://api.yonne.app/api/v1/external/create-order", {
    method: "POST",
    headers: {
      "X-API-Key": process.env.YONNE_API_KEY,
      "Content-Type": "application/json",
      "Idempotency-Key": `order-${internalOrder.id}-attempt-1`
    },
    body: JSON.stringify({
      delivery_address: internalOrder.deliveryAddress,
      receiver_name: internalOrder.customerName,
      receiver_phone: internalOrder.customerPhone,
      delivery_fee: internalOrder.deliveryFee, // from the quote — do not recalculate
      delivery_lat: internalOrder.deliveryLat,
      delivery_lng: internalOrder.deliveryLng,
      item_name: internalOrder.itemName,
      product_type: "Package",
      weight_estimate: internalOrder.weightEstimate,
      merchant_reference_id: internalOrder.id,
      metadata: {
        channel: "web",
        checkout_session_id: internalOrder.checkoutSessionId
      }
    })
  });

  const data = await res.json();
  if (!data.success) throw data;

  // Save both — you'll need them for tracking and support
  await db.orders.update(internalOrder.id, {
    yonne_order_id: data.order_id,
    yonne_tracking_id: data.tracking_id,
    tracking_link: data.tracking_link
  });

  return data;
}
```

```python Python
import requests, os

def dispatch_order(internal_order):
    response = requests.post(
        "https://api.yonne.app/api/v1/external/create-order",
        headers={
            "X-API-Key": os.environ["YONNE_API_KEY"],
            "Content-Type": "application/json",
            "Idempotency-Key": f"order-{internal_order['id']}-attempt-1"
        },
        json={
            "delivery_address": internal_order["delivery_address"],
            "receiver_name": internal_order["customer_name"],
            "receiver_phone": internal_order["customer_phone"],
            "delivery_fee": internal_order["delivery_fee"],
            "delivery_lat": internal_order["delivery_lat"],
            "delivery_lng": internal_order["delivery_lng"],
            "item_name": internal_order["item_name"],
            "product_type": "Package",
            "weight_estimate": internal_order["weight_estimate"],
            "merchant_reference_id": internal_order["id"],
            "metadata": {
                "channel": "web",
                "checkout_session_id": internal_order["checkout_session_id"]
            }
        }
    )
    data = response.json()
    if not data.get("success"):
        raise Exception(data)
    return data
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
  "environment": "live"
}
```

**What to do with this response:**
- Save `order_id` — use it for cancellations, status checks, and support lookups.
- Save `tracking_id` — use it in customer-facing tracking links.
- Show `tracking_link` on the order confirmation page so the customer can track in real time.

---

## Handling create-order failures

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

The customer's payment has already gone through, but your Yonne wallet is empty. Handle this gracefully:

1. Flag the order internally as `dispatch_failed_insufficient_funds`.
2. Alert your operations team immediately — they need to top up the wallet.
3. After topping up, retry create-order with the same `Idempotency-Key`.
4. Do not leave the customer without a response — send them an "order confirmed, we'll dispatch soon" message.

See [Wallet Management](/docs/integration-workflows/wallet-management) for the full top-up flow.

### `422 ERR_NO_CAPACITY_AVAILABLE`

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

No rider is available right now. Do not silently downgrade the vehicle class. Flag the order as `dispatch_failed_no_capacity` and retry later.

---

## Step 5 — Show the customer their tracking link

After `create-order` succeeds, surface `tracking_link` on the order confirmation page:

```html
<p>Your order is confirmed! <a href="{{ tracking_link }}">Track your delivery →</a></p>
```

From this point, Yonne will push status updates to your webhook endpoint as the order progresses. See [Real-time Tracking](/docs/integration-workflows/real-time-tracking) to wire up the full tracking experience.

---

## Integration checklist

Before going live with this flow:

- [ ] `GET /validate` is called on startup and `hasPickup` is confirmed
- [ ] Customer location is collected via the browser Geolocation API (or geocoding) before calling `/quote`
- [ ] `delivery_fee` from the quote is passed unchanged into create-order (not recalculated)
- [ ] `Idempotency-Key` is generated from your internal order ID on every create-order call
- [ ] `order_id` and `tracking_id` are saved to your database
- [ ] `402` and `422` errors alert your ops team rather than failing silently
- [ ] Webhook receiver is live to receive status updates without polling
