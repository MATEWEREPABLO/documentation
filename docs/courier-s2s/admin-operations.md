---
title: "Admin Operations"
description: "How Yonne admins generate, enable, rotate, and revoke S2S credentials for a courier company."
---

All credential management is performed by Yonne admin staff — not by the courier's system. This page documents the full lifecycle so courier engineering teams understand what to expect and what to ask for.

---

## Credential lifecycle at a glance

```
1. Admin generates credentials  (OTP required)
         │
         ▼
2. Admin enables the integration
         │
         ▼
3. Courier system authenticates + calls endpoints
         │
         ▼
4. Admin rotates credentials when needed  (OTP required)
         │    or
         └──▶ Admin revokes / disables at any time
```

---

## Step 1 — Generate credentials

The admin triggers an OTP to their verified email, verifies it, then calls generate. This must happen before your system can do anything.

**Admin endpoint:**
```http
POST /admin/couriers/{courier_id}/s2s/generate-credentials
```

On first call, the platform creates three credentials and returns them **in plaintext exactly once**:

| Credential | Prefix | Purpose |
|---|---|---|
| `api_key` | `ck_live_` | Identifies your courier account |
| `api_secret` | `sk_live_` | Proves ownership of the key — treated like a password |
| `webhook_secret` | `whsec_` | Used to verify webhook payload signatures |

<Warning>
  The admin must copy these values and hand them to your team immediately. After this single reveal, the `api_secret` and `webhook_secret` are stored only as hashes — they cannot be retrieved again. If they are lost, the admin must rotate.
</Warning>

**After generate:** your integration is still disabled (`s2s_enabled = false`). The admin must explicitly enable it before your JWTs will be accepted.

---

## Step 2 — Enable the integration

```http
PUT /admin/couriers/{courier_id}/s2s/toggle
```

```json
{ "enabled": true }
```

This flips `s2s_enabled` to `true`. From this point, your system can authenticate and call S2S endpoints.

---

## OTP requirement for credential actions

Generate and rotate both require the admin to complete an OTP challenge within the last 5 minutes. The flow:

| Step | Endpoint | Detail |
|---|---|---|
| 1. Request OTP | `POST /admin/couriers/{id}/s2s/send-otp` | 6-digit code sent to courier's email. Rate-limited: 3 sends per 10 minutes. |
| 2. Verify OTP | `POST /admin/couriers/{id}/s2s/verify-otp` | Code valid for 10 minutes. 5 failed attempts triggers a 15-minute lockout. |
| 3. Perform action | Generate or rotate | Must happen within 5 minutes of successful OTP verification. |

---

## Rotate credentials

If credentials are exposed or need to be cycled, the admin rotates them:

```http
POST /admin/couriers/{courier_id}/s2s/rotate-credentials
```

Rotation requires OTP verification. It:
- Generates a new `api_key`, `api_secret`, and `webhook_secret` immediately.
- Revokes the old credentials — any tokens issued with the old key stop working.
- Preserves the existing webhook URL.

**What your team needs to do after a rotate:**
1. Receive the new credentials from the Yonne admin.
2. Update `COURIER_API_KEY` and `COURIER_API_SECRET` in your environment variables.
3. Restart or redeploy your service so the new credentials take effect.
4. Verify by calling `/api/courier/s2s/auth/token` — a successful response confirms the new credentials are working.

---

## Revoke (disable) the integration

```http
POST /admin/couriers/{courier_id}/s2s/revoke
```

Sets `s2s_enabled = false`. Your credentials are preserved — no new generate is needed to re-enable. The admin can restore access by calling toggle with `{ "enabled": true }`.

Your system will receive `403 S2S_DISABLED` on all endpoint calls while revoked.

---

## Check integration status

The admin can check the current state of the integration at any time:

```http
GET /admin/couriers/{courier_id}/s2s/status
```

**Response:**

```json
{
  "status_summary": "Active and healthy",
  "s2s_enabled": true,
  "api_key": "ck_live_xxxx...5678",
  "s2s_created_at": "2026-06-15T09:00:00Z",
  "s2s_last_used_at": "2026-06-17T10:00:00Z"
}
```

**Status summaries:**

| Summary | Meaning |
|---|---|
| `"Not configured"` | No credentials have been generated yet |
| `"Disabled"` | Credentials exist but the integration is turned off |
| `"Configured but inactive"` | Enabled, but no recent API calls have been made |
| `"Active and healthy"` | Enabled and recently used |

The `api_key` is returned redacted (first 8 + last 4 characters).

---

## Set a webhook URL

```http
PUT /admin/couriers/{courier_id}/s2s/webhook
```

Sets the URL where Yonne will deliver webhook events for this courier. The `webhook_secret` (generated with credentials) is used to sign payloads — verify it on your end using HMAC.

---

## All admin endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/admin/couriers/{id}/s2s/send-otp` | Send OTP to courier email |
| `POST` | `/admin/couriers/{id}/s2s/verify-otp` | Verify OTP, open 5-minute action window |
| `POST` | `/admin/couriers/{id}/s2s/generate-credentials` | Generate (or reveal) credentials |
| `POST` | `/admin/couriers/{id}/s2s/rotate-credentials` | Rotate all credentials |
| `POST` | `/admin/couriers/{id}/s2s/revoke` | Disable the integration |
| `GET` | `/admin/couriers/{id}/s2s/status` | View integration status |
| `PUT` | `/admin/couriers/{id}/s2s/webhook` | Set webhook URL |
| `PUT` | `/admin/couriers/{id}/s2s/toggle` | Enable or disable the integration |
