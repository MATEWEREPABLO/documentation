---
title: "Metrics Summary"
description: "Pull a combined snapshot of delivery counts, driver availability, and financial totals for your courier account."
---

The metrics summary endpoint returns a single response with delivery, driver, and financial data for a given date range. Use it to build dashboards, generate reports, or monitor your account health.

---

## Endpoint

```http
GET /api/courier/s2s/metrics/summary
```

**Requires:** `Authorization: Bearer <token>` — see [Authentication](/docs/courier-s2s/authentication).

---

## Query parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `start_date` | ISO 8601 date | No | 30 days ago | Start of the reporting period |
| `end_date` | ISO 8601 date | No | Today | End of the reporting period |
| `source` | string | No | — | Filter delivery counts by source. Does not affect financial figures. |
| `currency` | string | No | `MWK` | Currency for financial figures |

---

## Request examples

**Last 30 days (default):**

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/courier/s2s/metrics/summary" \
  --header "Authorization: Bearer <token>"
```

```javascript Node.js
const res = await fetch("https://api.yonne.app/api/courier/s2s/metrics/summary", {
  headers: { "Authorization": `Bearer ${await getToken()}` }
});
const metrics = await res.json();
```

```python Python
import requests

response = requests.get(
    "https://api.yonne.app/api/courier/s2s/metrics/summary",
    headers={"Authorization": f"Bearer {token}"}
)
metrics = response.json()
```
</CodeGroup>

**Custom date range:**

<CodeGroup>
```bash cURL
curl --request GET \
  "https://api.yonne.app/api/courier/s2s/metrics/summary?start_date=2026-06-01&end_date=2026-06-17" \
  --header "Authorization: Bearer <token>"
```

```javascript Node.js
const params = new URLSearchParams({
  start_date: "2026-06-01",
  end_date: "2026-06-17"
});

const res = await fetch(
  `https://api.yonne.app/api/courier/s2s/metrics/summary?${params}`,
  { headers: { "Authorization": `Bearer ${await getToken()}` } }
);
```

```python Python
response = requests.get(
    "https://api.yonne.app/api/courier/s2s/metrics/summary",
    headers={"Authorization": f"Bearer {token}"},
    params={"start_date": "2026-06-01", "end_date": "2026-06-17"}
)
```
</CodeGroup>

---

## Response

```json
{
  "period": {
    "start": "2026-05-18",
    "end": "2026-06-17"
  },
  "delivery_metrics": {
    "total_deliveries": 120,
    "completed": 95,
    "cancelled": 10,
    "pending": 15,
    "average_rating": 4.6
  },
  "driver_metrics": {
    "active_drivers": 12,
    "total_registered": 20,
    "busy_external": 3,
    "offline": 5
  },
  "financial_summary": {
    "gross_delivery_value": 450000.00,
    "courier_earnings": 90000.00,
    "platform_fees": 45000.00,
    "total_withdrawn": 60000.00,
    "average_delivery_value": 3750.00
  }
}
```

---

## Response fields

### `delivery_metrics`

| Field | Description |
|---|---|
| `total_deliveries` | All deliveries in the period |
| `completed` | Successfully delivered |
| `cancelled` | Cancelled before delivery |
| `pending` | In-progress or not yet dispatched |
| `average_rating` | Average customer rating across completed deliveries |

### `driver_metrics`

| Field | Description |
|---|---|
| `active_drivers` | Drivers currently available for dispatch |
| `total_registered` | All drivers registered under your courier account |
| `busy_external` | Drivers you've marked busy via [Update Driver Status](/docs/courier-s2s/update-driver-status) |
| `offline` | Drivers who are offline |

### `financial_summary`

| Field | Description |
|---|---|
| `gross_delivery_value` | Total value of all deliveries billed to customers |
| `courier_earnings` | Your net earnings after platform fees |
| `platform_fees` | Intercity's fee deducted from gross value |
| `total_withdrawn` | Amount already withdrawn from your account |
| `average_delivery_value` | Average delivery value per completed order |

<Note>
  Financial figures always reflect your **entire** courier account, regardless of the `source` filter. The `source` parameter only affects delivery counts.
</Note>

---

## Using the `source` filter

If your deliveries come from multiple channels (e.g. your own app, partner platforms), pass `source` to filter the delivery count breakdown to one channel:

```bash
GET /api/courier/s2s/metrics/summary?source=partner_app&start_date=2026-06-01&end_date=2026-06-17
```

This narrows `delivery_metrics` to only deliveries originating from `partner_app`. Financial figures remain courier-wide.
