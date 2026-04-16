# Rate limiting

Yonne does not currently publish fixed rate-limit numbers in the API contract.

You should still treat the API as a shared production service and send requests with predictable, controlled traffic patterns.

## Recommended client behavior

- Avoid tight retry loops.
- Use exponential backoff for transient failures.
- Reuse the same `Idempotency-Key` when retrying `POST /api/v1/external/create-order`.
- Cache stable merchant context from `GET /api/v1/external/validate` when possible.
- Do not poll tracking endpoints aggressively when webhook delivery can give you the same state changes.

## Order creation retries

Order creation is the most important endpoint to protect against duplicate submission.

If a create-order request times out on your side, retry with the same `Idempotency-Key` instead of generating a new one.

## Capacity and balance failures

Do not treat business failures as retryable rate-limit events.

Examples include:

- `402 Insufficient Funds`
- `422 ERR_NO_CAPACITY_AVAILABLE`
- validation failures such as missing coordinates

Resolve the underlying issue before sending another request.

## Operational guidance

Throttle bursts in your own backend, especially during checkout spikes, imports, or replay jobs.

If Yonne introduces explicit `429` responses or response headers later, update your client policy to honor them exactly.
