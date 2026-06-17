---
title: "Authentication"
description: "Exchange your api_key and api_secret for a short-lived JWT, then attach it to every S2S request."
---

S2S authentication is a two-step process: exchange your credentials for a JWT, then attach that JWT as a Bearer token on every subsequent request.

<Note>
  Your `api_key` and `api_secret` are shown **once** when the Yonne admin generates them. Store them securely in environment variables immediately — they cannot be retrieved again (only regenerated via rotate).
</Note>

---

## Step 1 — Exchange credentials for a JWT

```http
POST /api/courier/s2s/auth/token
```

<CodeGroup>
```bash cURL
curl --request POST "https://api.yonne.app/api/courier/s2s/auth/token" \
  --header "Content-Type: application/json" \
  --data '{
    "api_key": "ck_live_xxxxxxxxxxxx",
    "api_secret": "sk_live_xxxxxxxxxxxx"
  }'
```

```javascript Node.js
const res = await fetch("https://api.yonne.app/api/courier/s2s/auth/token", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    api_key: process.env.COURIER_API_KEY,
    api_secret: process.env.COURIER_API_SECRET
  })
});

const { token, expires_in } = await res.json();
// Store token in memory — reuse it until expiry
```

```python Python
import requests, os

response = requests.post(
    "https://api.yonne.app/api/courier/s2s/auth/token",
    json={
        "api_key": os.environ["COURIER_API_KEY"],
        "api_secret": os.environ["COURIER_API_SECRET"]
    }
)
data = response.json()
token = data["token"]
# Store token in memory — reuse it until expiry
```
</CodeGroup>

**Success response:**

```json
{
  "token": "<jwt>",
  "courier_id": 123,
  "courier_name": "Acme Logistics",
  "expires_in": 86400
}
```

| Field | Description |
|---|---|
| `token` | The JWT to attach to all S2S requests |
| `courier_id` | Your courier ID on the platform |
| `expires_in` | Seconds until the token expires — always `86400` (24 hours) |

---

## Step 2 — Attach the JWT to every request

All protected S2S endpoints require the token in the `Authorization` header:

```http
Authorization: Bearer <token>
```

<CodeGroup>
```bash cURL
curl --request GET "https://api.yonne.app/api/courier/s2s/metrics/summary" \
  --header "Authorization: Bearer <token>"
```

```javascript Node.js
const res = await fetch("https://api.yonne.app/api/courier/s2s/metrics/summary", {
  headers: { "Authorization": `Bearer ${token}` }
});
```

```python Python
response = requests.get(
    "https://api.yonne.app/api/courier/s2s/metrics/summary",
    headers={"Authorization": f"Bearer {token}"}
)
```
</CodeGroup>

---

## Token lifetime and refresh

The JWT is valid for **24 hours**. Implement a simple refresh strategy in your integration:

```javascript Node.js
let token = null;
let tokenExpiry = null;

async function getToken() {
  if (token && Date.now() < tokenExpiry - 60_000) {
    return token; // reuse while still valid (with 1-min buffer)
  }

  const res = await fetch("https://api.yonne.app/api/courier/s2s/auth/token", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      api_key: process.env.COURIER_API_KEY,
      api_secret: process.env.COURIER_API_SECRET
    })
  });

  const data = await res.json();
  token = data.token;
  tokenExpiry = Date.now() + data.expires_in * 1000;
  return token;
}
```

---

## Credential prefixes

| Prefix | Credential |
|---|---|
| `ck_live_` | `api_key` |
| `sk_live_` | `api_secret` |

---

## Authentication errors

| Code | HTTP | Meaning | Fix |
|---|---|---|---|
| `TOKEN_MISSING` | 401 | No `Authorization` header | Add `Authorization: Bearer <token>` to your request |
| `TOKEN_EXPIRED` | 401 | Token is older than 24 hours | Re-authenticate to get a fresh token |
| `TOKEN_INVALID` | 401 | Bad signature or wrong token type | Ensure you're using a token from `/api/courier/s2s/auth/token` |
| `COURIER_NOT_FOUND` | 404 | The `courier_id` in the token no longer exists | Contact Yonne support |
| `S2S_DISABLED` | 403 | Integration has been disabled by admin | Contact your Yonne account manager |
| `DB_ERROR` | 503 | Platform database unreachable | Retry with exponential backoff |

<Warning>
  A `403 S2S_DISABLED` response means your integration was turned off on the admin side. Your credentials are still intact — the Yonne admin just needs to re-enable the integration via toggle.
</Warning>
