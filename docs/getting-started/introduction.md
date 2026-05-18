---
title: "What is Yonne Deliveries?"
description: "Yonne gives your application a single API to quote delivery fees, dispatch riders, and track shipments in real time across Malawi."
---

## Overview

Yonne is a last-mile delivery platform. Integrate once and your customers get live delivery fees at checkout, automatic rider dispatch when they pay, and a real-time tracking link the moment the order is confirmed.

<Note>
  Don't have a merchant account yet? Sign up or log in at [merchant.yonne.app](https://merchant.yonne.app).
</Note>

---

## The 6-step integration flow

Every integration follows the same sequence. Understanding this flow end-to-end is the fastest way to build correctly.

```
┌─────────────────────────────────────────────────────────────────┐
│  Step 1 › Validate          GET /api/v1/external/validate       │
│           Confirm your API key is active and your pickup        │
│           address is configured before doing anything else.     │
├─────────────────────────────────────────────────────────────────┤
│  Step 2 › Quote             POST /api/v1/external/quote         │
│           Request the customer's location via the browser       │
│           Geolocation API, then send the coordinates to get a   │
│           real-time delivery_fee and suggested_vehicle_class.   │
├─────────────────────────────────────────────────────────────────┤
│  Step 3 › Collect Payment   (your checkout logic)               │
│           The customer pays on your platform. Hold the          │
│           quoted delivery_fee — you'll pass it to Step 4.       │
├─────────────────────────────────────────────────────────────────┤
│  Step 4 › Create Order      POST /api/v1/external/create-order  │
│           Dispatch the rider. Always include an Idempotency-Key │
│           to prevent duplicate orders on retries.               │
├─────────────────────────────────────────────────────────────────┤
│  Step 5 › Track             GET /api/v1/external/track/:id      │
│           Poll tracking status or surface the tracking_link     │
│           to your customer.                                     │
├─────────────────────────────────────────────────────────────────┤
│  Step 6 › Handle Webhooks   POST <your-endpoint>                │
│           Yonne pushes order.status_updated events to your      │
│           server in real time — no polling required.            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Base URL

All requests go to the same host regardless of environment:

```text
https://api.yonne.app
```

Your API key prefix (`yonne_live_` vs `yonne_test_`) is what determines whether you're in production or sandbox. See [Environments](/docs/getting-started/environments) for the full breakdown.

---

## What to read next

| You want to… | Go here |
|---|---|
| Make your first API call in 5 minutes | [Quickstart](/docs/getting-started/quickstart) |
| Understand API keys and headers | [Authentication](/docs/getting-started/authentication) |
| See test vs. live mode behavior | [Environments](/docs/getting-started/environments) |
| Build an e-commerce checkout integration | [E-commerce Checkout Workflow](/docs/integration-workflows/ecommerce-checkout) |
