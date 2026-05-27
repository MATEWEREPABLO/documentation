---
title: "Wallet Management"
description: "Check your Yonne wallet balance, handle 402 Insufficient Funds errors gracefully, and build a top-up flow that unblocks dispatches automatically."
---

Yonne deducts the `delivery_fee` from your merchant wallet when each order is created. If your balance drops below the fee for a delivery, the create-order call fails with `402 Insufficient Funds`. This guide shows you how to check balance, respond to that error, and build a proactive top-up workflow.

---

## Check your wallet balance

Call `GET /api/v1/external/wallet` to get your current balance:

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/v1/external/wallet" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```

```javascript Node.js
const res = await fetch("https://api.yonne.app/api/v1/external/wallet", {
  headers: { "X-API-Key": process.env.YONNE_API_KEY }
});
const wallet = await res.json();
console.log(wallet.balance, wallet.currency);
```

```python Python
import requests, os

wallet = requests.get(
    "https://api.yonne.app/api/v1/external/wallet",
    headers={"X-API-Key": os.environ["YONNE_API_KEY"]}
).json()
print(wallet["balance"], wallet["currency"])
```
</CodeGroup>

**Response:**

```json
{
  "success": true,
  "balance": 12500,
  "currency": "MWK",
  "environment": "live"
}
```

---

## Handling a `402 Insufficient Funds` error

When `create-order` returns `402`, the response tells you exactly what you have and what you need:

```json
{
  "success": false,
  "error": "Insufficient Funds",
  "message": "Wallet balance is insufficient for this delivery. Please top up.",
  "balance": 1500,
  "required": 7350
}
```

**Your response should be:**

1. **Don't fail silently.** The customer's payment succeeded — their order must not simply disappear.
2. **Flag the order** in your system as `dispatch_pending_funds`.
3. **Alert your operations team** with the shortfall: `required - balance = 5850 MWK needed`.
4. **Tell the customer** that their order is confirmed and the delivery will be dispatched shortly.
5. **After topping up**, retry `create-order` with the **same `Idempotency-Key`** to dispatch.

---

## The full 402 recovery flow

<CodeGroup>
```javascript Node.js
async function dispatchWithFundsCheck(internalOrder) {
  const idempotencyKey = `order-${internalOrder.id}-attempt-1`;

  const res = await fetch("https://api.yonne.app/api/v1/external/create-order", {
    method: "POST",
    headers: {
      "X-API-Key": process.env.YONNE_API_KEY,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey
    },
    body: JSON.stringify({
      delivery_address: internalOrder.deliveryAddress,
      receiver_name: internalOrder.customerName,
      receiver_phone: internalOrder.customerPhone,
      delivery_fee: internalOrder.deliveryFee,
      delivery_lat: internalOrder.deliveryLat,
      delivery_lng: internalOrder.deliveryLng,
      item_name: internalOrder.itemName,
      product_type: "Package",
      weight_estimate: internalOrder.weightEstimate,
      merchant_reference_id: internalOrder.id
    })
  });

  const data = await res.json();

  if (res.status === 402) {
    // Flag for ops team
    await db.orders.update(internalOrder.id, { status: "dispatch_pending_funds" });
    await alertOpsTeam({
      type: "insufficient_funds",
      orderId: internalOrder.id,
      balance: data.balance,
      required: data.required,
      shortfall: data.required - data.balance,
      idempotencyKey // save so ops can retry with the same key
    });
    return { dispatched: false, reason: "insufficient_funds" };
  }

  if (!data.success) throw data;

  await db.orders.update(internalOrder.id, {
    status: "dispatched",
    yonne_order_id: data.order_id,
    yonne_tracking_id: data.tracking_id
  });
  return { dispatched: true, ...data };
}
```

```python Python
import requests, os

def dispatch_with_funds_check(internal_order):
    idempotency_key = f"order-{internal_order['id']}-attempt-1"

    response = requests.post(
        "https://api.yonne.app/api/v1/external/create-order",
        headers={
            "X-API-Key": os.environ["YONNE_API_KEY"],
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key
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
            "merchant_reference_id": internal_order["id"]
        }
    )
    data = response.json()

    if response.status_code == 402:
        # Flag for ops team
        db.orders.update(internal_order["id"], status="dispatch_pending_funds")
        alert_ops_team(
            order_id=internal_order["id"],
            balance=data["balance"],
            required=data["required"],
            shortfall=data["required"] - data["balance"],
            idempotency_key=idempotency_key
        )
        return {"dispatched": False, "reason": "insufficient_funds"}

    if not data.get("success"):
        raise Exception(data)

    return {"dispatched": True, **data}
```
</CodeGroup>

---

## Retrying after a top-up

Once your ops team tops up the wallet, retry the original create-order call using the **exact same `Idempotency-Key`** you used on the first attempt. Yonne will see the key, check that the original order was not yet created, and process it now.

```javascript Node.js
// Ops retries from the saved idempotency key
await retryDispatch({
  orderId: "WEB-100245",
  idempotencyKey: "order-WEB-100245-attempt-1"
});
```

Do not generate a new key — that would create a duplicate order.

---

## Proactive balance monitoring

To avoid `402` errors reaching customers at all, set up a daily or hourly balance check and alert your team when balance drops below a safe threshold:

<CodeGroup>
```javascript Node.js
async function checkWalletHealth(minBalanceMWK = 50000) {
  const res = await fetch("https://api.yonne.app/api/v1/external/wallet", {
    headers: { "X-API-Key": process.env.YONNE_API_KEY }
  });
  const { balance, currency } = await res.json();

  if (balance < minBalanceMWK) {
    await alertOpsTeam({
      type: "low_wallet_balance",
      balance,
      currency,
      threshold: minBalanceMWK
    });
  }
}
```

```python Python
import requests, os

def check_wallet_health(min_balance_mwk=50000):
    wallet = requests.get(
        "https://api.yonne.app/api/v1/external/wallet",
        headers={"X-API-Key": os.environ["YONNE_API_KEY"]}
    ).json()

    if wallet["balance"] < min_balance_mwk:
        alert_ops_team(
            balance=wallet["balance"],
            currency=wallet["currency"],
            threshold=min_balance_mwk
        )
```
</CodeGroup>

Run this on a schedule (e.g. every hour via cron) so your ops team knows to top up before orders start failing.

---

## Top-up process

Wallet top-ups are done through the Yonne merchant dashboard. The API does not expose a direct top-up endpoint — this is intentional to keep financial transactions in a controlled, audited channel.

After topping up in the dashboard, confirm the new balance before retrying pending dispatches:

```bash
curl --request GET "https://api.yonne.app/api/v1/external/wallet" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```
