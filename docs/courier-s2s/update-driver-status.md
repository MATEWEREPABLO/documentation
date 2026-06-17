---
title: "Update Driver Status"
description: "Mark a driver as externally busy or available so the platform dispatch algorithm can account for your own assignments."
---

When your system assigns a driver to a job outside of the Yonne platform, use this endpoint to mark them as busy. Yonne will skip that driver during dispatch until you mark them available again.

---

## Endpoint

```http
PUT /api/courier/s2s/riders/{rider_id}/status
```

**Requires:** `Authorization: Bearer <token>` — see [Authentication](/docs/courier-s2s/authentication).

---

## Mark a driver busy

<CodeGroup>
```bash cURL
curl --request PUT "https://api.yonne.app/api/courier/s2s/riders/99/status" \
  --header "Authorization: Bearer <token>" \
  --header "Content-Type: application/json" \
  --data '{
    "status": "busy_external",
    "reason": "Assigned to partner delivery",
    "external_reference": "EXT-456"
  }'
```

```javascript Node.js
async function markDriverBusy(riderId, externalRef) {
  const res = await fetch(
    `https://api.yonne.app/api/courier/s2s/riders/${riderId}/status`,
    {
      method: "PUT",
      headers: {
        "Authorization": `Bearer ${await getToken()}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        status: "busy_external",
        reason: "Assigned to partner delivery",
        external_reference: externalRef
      })
    }
  );
  return res.json();
}
```

```python Python
import requests

def mark_driver_busy(rider_id, external_ref, token):
    response = requests.put(
        f"https://api.yonne.app/api/courier/s2s/riders/{rider_id}/status",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        },
        json={
            "status": "busy_external",
            "reason": "Assigned to partner delivery",
            "external_reference": external_ref
        }
    )
    return response.json()
```
</CodeGroup>

## Mark a driver available

<CodeGroup>
```bash cURL
curl --request PUT "https://api.yonne.app/api/courier/s2s/riders/99/status" \
  --header "Authorization: Bearer <token>" \
  --header "Content-Type: application/json" \
  --data '{
    "status": "available"
  }'
```

```javascript Node.js
async function markDriverAvailable(riderId) {
  const res = await fetch(
    `https://api.yonne.app/api/courier/s2s/riders/${riderId}/status`,
    {
      method: "PUT",
      headers: {
        "Authorization": `Bearer ${await getToken()}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({ status: "available" })
    }
  );
  return res.json();
}
```

```python Python
def mark_driver_available(rider_id, token):
    response = requests.put(
        f"https://api.yonne.app/api/courier/s2s/riders/{rider_id}/status",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        },
        json={"status": "available"}
    )
    return response.json()
```
</CodeGroup>

---

## Request body

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | string | Yes | `"busy_external"` or `"available"` |
| `reason` | string | No | Human-readable reason, shown in the admin dashboard |
| `external_reference` | string | No | Your cross-system job ID — stored against the driver record |

---

## Success response

```json
{
  "driver_id": 99,
  "driver_reference_id": "DR-099",
  "status": "busy",
  "updated_at": "2026-06-17T10:00:00Z"
}
```

---

## Status mapping

| S2S value | Internal status | Effect on dispatch |
|---|---|---|
| `busy_external` | `busy` | Driver is skipped during Yonne dispatch |
| `available` | `active` | Driver is eligible for Yonne dispatch |

<Note>
  The response returns the internal `status` value (`"busy"` or `"active"`), not the S2S value you sent. This is expected.
</Note>

---

## When to use `external_reference`

If your system has a job ID for the external assignment (e.g. `"EXT-456"`), pass it as `external_reference`. Yonne stores it against the driver record — this helps admin staff cross-reference jobs when investigating driver availability issues.
