---
title: "Courier S2S Integration"
description: "Connect your logistics system directly to Intercity using server-to-server API calls — no browser, no dashboard login required."
---

The S2S (server-to-server) integration lets your courier company's backend talk directly to the Intercity platform. You can update driver availability, pull delivery metrics, and retrieve financial summaries entirely through API calls.

<Note>
  This is a separate integration from the Merchant API. S2S credentials, base paths, and authentication flow are all different. Do not mix them.
</Note>

---

## How it works

There are two actors in this system:

- **Intercity admin** — manages your credentials (generate, rotate, revoke, enable).
- **Your system** — exchanges credentials for a short-lived JWT, then calls S2S endpoints with that token.

```
Admin generates credentials
         │
         ▼
Your system sends api_key + api_secret → receives JWT (valid 24 h)
         │
         ▼
Your system calls S2S endpoints with JWT in Authorization header
         │
         ▼
Platform validates JWT → checks integration is enabled → serves response
```

---

## Before your first call

The Intercity admin must complete these steps before your system can authenticate:

| Step | Who does it | What happens |
|---|---|---|
| Generate credentials | Admin (with OTP) | Creates your `api_key` and `api_secret` |
| Enable integration | Admin | Flips the S2S integration on — your JWT will be rejected until this is done |

Once both steps are done, your system can authenticate and start calling endpoints. See [Admin Operations](/docs/courier-s2s/admin-operations) for the full admin workflow.

---

## Base URL

All S2S endpoints share the same host as the rest of the platform:

```text
https://api.yonne.app
```

Courier S2S paths are under `/api/courier/s2s/`. Admin management paths are under `/admin/couriers/{courier_id}/s2s/`.

---

## What to read next

| You want to… | Go here |
|---|---|
| Get a JWT and make your first call | [Authentication](/docs/courier-s2s/authentication) |
| Update a driver's availability | [Update Driver Status](/docs/courier-s2s/update-driver-status) |
| Pull delivery counts and financial data | [Metrics Summary](/docs/courier-s2s/metrics-summary) |
| Understand how admin manages credentials | [Admin Operations](/docs/courier-s2s/admin-operations) |
