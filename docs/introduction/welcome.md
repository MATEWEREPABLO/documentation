# Welcome

Yonne Deliveries helps you quote deliveries, create orders, track fulfillment, and handle operational failures through one external API.

Use this documentation to set up your first integration, understand key concepts, and move into the full API reference when you are ready.

## Base URL and environments

Yonne uses the same API host for both environments:

```text
https://api.yonne.app
```

Your API key determines whether the request runs in sandbox or production.

<CodeGroup>
```bash Sandbox
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_test_xxxxxxxxx"
```

```bash Production
curl --request GET "https://api.yonne.app/api/v1/external/validate" \
  --header "X-API-Key: yonne_live_xxxxxxxxx"
```
</CodeGroup>

## Recommended integration flow

1. Validate your API key with `GET /api/v1/external/validate`.
2. Confirm the merchant pickup location is configured.
3. Request a quote with `POST /api/v1/external/quote`.
4. Create the order with `POST /api/v1/external/create-order`.
5. Track progress with the order and tracking endpoints.
6. Handle webhooks and business failures in your backend.

## What to read next

- Read `docs/introduction/terminology` to align your team on core platform terms.
- Read `docs/guides/authentication` to understand API key usage and request headers.
- Read `docs/guides/external-api-first-order` for the full first-order workflow.
- Read `docs/guides/external-api-errors-and-retries` before going live.
