# Guide: Authentication

Authenticate requests with an API key in the `X-API-Key` header.

Production keys start with `yonne_live_`. Sandbox keys start with `yonne_test_`.

## Recommended header

```http
X-API-Key: yonne_live_xxxxxxxxx
```

## Alternate header support

The API also accepts:

```http
Authorization: Bearer yonne_live_xxxxxxxxx
```

Use one approach consistently across your integration. `X-API-Key` is the documented default.

## Validate the key early

Call `GET /api/v1/external/validate` during onboarding and when debugging merchant configuration issues.

That response confirms whether the key is active and returns useful merchant context such as wallet balance and pickup configuration.

## Common authentication failures

### Missing key

If you do not send a key, the API returns `401 Unauthorized`.

### Malformed or inactive key

If the key is invalid or inactive, the API also returns `401 Unauthorized`.

### Wrong environment expectation

The API host is the same in sandbox and production. The key prefix determines the environment behavior.
