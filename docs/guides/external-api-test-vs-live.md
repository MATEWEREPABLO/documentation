# Yonne External API — Test Mode vs Live Mode

This guide explains the practical difference between **test mode** and **live mode** when using the Yonne External API.

## How the environment is selected

The environment is determined by the merchant API key prefix.

- `yonne_live_...` -> **live**
- `yonne_test_...` -> **test**

The same API host is used for both:

```text
https://api.yonne.app
```

## Why this matters

The environment affects:

- wallet behavior
- order creation side effects
- tracking identifiers
- webhook behavior
- operational expectations

## Live mode

Live mode is the production behavior used for real deliveries.

### In live mode

- merchant wallet balance is checked before order creation
- live wallet balance is debited
- real dispatch logic runs
- capacity-aware rider selection runs
- `ERR_NO_CAPACITY_AVAILABLE` can block order creation
- notifications and downstream side effects run
- order webhooks are dispatched

### Live create-order example

```http
POST /api/v1/external/create-order
X-API-Key: yonne_live_xxxxxxxxx
Idempotency-Key: order-100245-attempt-1
```

Possible live failure:

```json
{
  "success": false,
  "error": "Insufficient Funds",
  "message": "Wallet balance is insufficient for this delivery. Please top up.",
  "balance": 1500,
  "required": 7350
}
```

Another possible live failure:

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

## Test mode

Test mode is for integration validation and QA.

### In test mode

- environment is returned as `test`
- test tracking IDs are generated
- real live merchant billing should not be assumed
- the flow is safer for sandbox experimentation
- webhook simulation tools are available

### Important note

Even though the endpoint behavior resembles production, your integration should treat test mode as **non-production** and avoid assuming real financial or operational fulfillment.

## Side-by-side comparison

| Area | Live | Test |
|------|------|------|
| Key prefix | `yonne_live_...` | `yonne_test_...` |
| Environment field | `live` | `test` |
| Wallet validation | Yes | sandbox/test context |
| Real dispatch expectation | Yes | no production assumption |
| Production merchant operations | Yes | No |
| Webhook simulation endpoint | Available | especially useful here |

## Recommended test workflow

### 1. Validate the test key

```http
GET /api/v1/external/validate
X-API-Key: yonne_test_xxxxxxxxx
```

### 2. Create a test quote

```http
POST /api/v1/external/quote
X-API-Key: yonne_test_xxxxxxxxx
```

### 3. Create a test order

```http
POST /api/v1/external/create-order
X-API-Key: yonne_test_xxxxxxxxx
Idempotency-Key: test-order-1
```

### 4. Simulate status changes

```http
POST /api/v1/external/test/simulate-status
X-API-Key: yonne_test_xxxxxxxxx
```

Example body:

```json
{
  "order_id": "ORD-123456",
  "status": "In Transit"
}
```

## Recommended live workflow

### Before going live

- confirm merchant pickup location is configured
- confirm wallet has sufficient balance
- confirm quote and create-order payloads use canonical snake_case fields
- confirm your webhook receiver is ready
- confirm heavy-package handling is tested
- confirm idempotency keys are implemented

### During production operation

- always send `Idempotency-Key` on create-order
- log `order_id`, `tracking_id`, `merchant_reference_id`
- handle `402` and `422` separately from transport errors

## Common mistakes

### Using the wrong key in the wrong environment

Symptom:

- order not found
- unexpected wallet values
- confusion about missing orders

Fix:

- make sure your integration keeps live and test keys separate

### Treating test mode like production dispatch

Symptom:

- assuming every test order maps to a real rider workflow

Fix:

- use test mode to validate integration logic, not real courier operations

### Not using idempotency in either mode

Symptom:

- duplicate orders after retry

Fix:

- always send `Idempotency-Key` in both live and test mode

## Best practice summary

- use **test mode** for integration QA, webhook receiver development, and payload validation
- use **live mode** only after merchant setup is complete
- keep keys, logging, and dashboards clearly separated by environment
