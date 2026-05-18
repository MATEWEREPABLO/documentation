---
title: "Authentication"
description: "Authenticate every Yonne API request using your API key in the X-API-Key header."
---

# Authentication

Every request to the Yonne API must include your API key. There is no OAuth flow — your key is your credential.

---

## The required header

```http
X-API-Key: yonne_live_xxxxxxxxx
```

Send this on every request. The API also accepts the key as a Bearer token if your HTTP client forces that format:

```http
Authorization: Bearer yonne_live_xxxxxxxxx
```

Pick one approach and use it consistently. `X-API-Key` is the standard.

---

## Key prefixes and what they mean

| Prefix | Environment | Behavior |
|---|---|---|
| `yonne_live_` | Production | Real wallet deductions, real rider dispatch, live webhooks |
| `yonne_test_` | Sandbox | Safe for development — see [Environments](/docs/getting-started/environments) |

The API host is identical in both environments — only the key prefix changes the behavior.

---

## Validate your key on startup

Call `GET /api/v1/external/validate` during application boot or when debugging. It returns your merchant context in one shot:

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```

```javascript Node.js
const response = await fetch("https://api.yonne.app/api/v1/external/validate", {
  headers: { "X-API-Key": "yonne_live_xxxxxxxxx" }
});
const merchant = await response.json();
// merchant.hasPickup, merchant.balance, merchant.pickupAddress
```

```python Python
import requests

merchant = requests.get(
    "https://api.yonne.app/api/v1/external/validate",
    headers={"X-API-Key": "yonne_live_xxxxxxxxx"}
).json()
# merchant["hasPickup"], merchant["balance"]
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

---

## Authentication errors

### `401 Unauthorized` — key missing

```json
{
  "success": false,
  "error": "Unauthorized",
  "message": "API key required. Send X-API-Key header or Authorization: Bearer <key>."
}
```

Fix: confirm the `X-API-Key` header is present in every request.

### `401 Unauthorized` — key invalid or inactive

Same HTTP status, same error shape. Check that the key is copied correctly from your dashboard and is marked active.

---

## Security best practices

- Store your key in an environment variable, never in source code.
- Use your `yonne_test_` key in local development and CI.
- Rotate your `yonne_live_` key immediately if it is exposed — do this from [merchant.yonne.app](https://merchant.yonne.app).
- Never log the full key — log only the prefix (e.g. `yonne_live_***`) if you need a reference in your logs.
