# Guide: Errors, Retries, and Idempotency

This guide explains how to handle the most important failure modes in the Yonne External API.

## Always use idempotency on order creation

When calling:

```text
POST /api/v1/external/create-order
```

send:

```http
Idempotency-Key: <unique-key>
```

This prevents accidental duplicate orders when:

- the merchant retries after a timeout
- the network drops after the request is sent
- the client does not know whether the first attempt succeeded

If the same key is reused within 24 hours, the API returns the cached success response and may include:

```json
{
  "idempotent_replay": true
}
```

## Important error types

### 401 Unauthorized

Cause:

- API key missing
- API key malformed
- API key inactive

Action:

- confirm the `X-API-Key` header is present
- validate the key with `GET /api/v1/external/validate`

### 400 Missing coordinates / invalid coordinates

Cause:

- destination coordinates missing
- destination or pickup coordinates invalid
- merchant pickup not configured

Action:

- ensure `delivery_lat` and `delivery_lng` are sent
- validate coordinate formats before sending
- confirm the merchant pickup location exists

### 402 Insufficient Funds

Cause:

- live wallet balance is lower than the requested delivery fee

Action:

- call `GET /api/v1/external/wallet`
- top up before retrying

### 422 ERR_NO_CAPACITY_AVAILABLE

Cause:

- no online rider can carry the shipment at the required weight

Action:

- do not silently downgrade to a smaller vehicle
- surface a meaningful message
- retry later or reduce shipment size

## Retry policy

### Safe to retry

- `GET` requests
- `POST /create-order` **only with the same `Idempotency-Key`**

### Do not blindly retry

- `402 Insufficient Funds`
- `422 ERR_NO_CAPACITY_AVAILABLE`
- validation failures such as missing coordinates

## Suggested client logic

### For temporary uncertainty

If create-order times out on the client side:

1. retry with the same `Idempotency-Key`
2. if needed, fetch the merchant order list or track by your saved references

### For business failures

If you receive:

- `ERR_NO_CAPACITY_AVAILABLE`
- `Insufficient Funds`

then resolve the business issue first instead of retrying immediately.

## Good operational logging

Log these on your side for support and reconciliation:

- merchant reference ID
- idempotency key
- order ID
- tracking ID
- response status code
- response error payload
