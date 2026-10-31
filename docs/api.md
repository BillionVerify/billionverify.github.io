# API Overview

Canonical page: https://billionverify.github.io/docs/api/

> The REST API is the source of truth for BillionVerify. SDKs in every language are reference implementations. If an SDK differs from this reference, follow the API.

The BillionVerify API is a REST API for single email verification, synchronous batch checks, asynchronous file jobs, credit balance and webhooks. Use HTTPS for all requests.

## Base URL

```text
https://api.billionverify.com/v1
```

## OpenAPI Specification

The canonical machine-readable contract is available here:

- **OpenAPI YAML**: [https://api.billionverify.com/openapi.yaml](https://api.billionverify.com/openapi.yaml)
- **Interactive API Docs**: [https://api.billionverify.com/docs](https://api.billionverify.com/docs)

## Authentication

Send your API key in the `BV-API-KEY` header:

```http
BV-API-KEY: sk_...
```

These forms are accepted:

1. `BV-API-KEY` *(recommended)*
2. `BILLIONVERIFY-API-KEY`
3. `Authorization: Bearer <api_key>`

If a request carries more than one, `BILLIONVERIFY-API-KEY` is read first, then `BV-API-KEY`, then the `Authorization` header.

> Get your API key from the [BillionVerify dashboard](https://billionverify.com/auth/sign-in?next=/home).

### Example

```bash
curl -X POST https://api.billionverify.com/v1/verify/single \
  -H "BV-API-KEY: sk_xxx" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","check_smtp":false}'
```

## Rate Limits

Rate limits are applied per account and endpoint type. These are the default limits. An account can have custom limits.

| Endpoint                                        | Rate Limit            |
| ----------------------------------------------- | --------------------- |
| Single Verification (`/verify/single`)          | 6,000 requests/minute |
| Batch Verification (`/verify/bulk`)             | 1,500 requests/minute |
| File Upload (`/verify/file`)                    | 300 requests/minute   |
| Disposable Check (`/verify/disposable`)         | 6,000 requests/minute |
| Other endpoints (`/credits`, `/webhooks`, etc.) | 600 requests/minute   |

Rate-limited endpoints return `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` (Unix timestamp) headers. When you exceed a limit, the API returns `429 Too Many Requests` with a `Retry-After` header (seconds):

```json
{
  "success": false,
  "code": "1003",
  "message": "Rate limit exceeded, please try again later",
  "error": {
    "code": "1003",
    "message": "Rate limit exceeded, please try again later",
    "description": "Rate limit exceeded: 6000 requests per 1m0s",
    "details": {
      "client_id": "abc123",
      "retry_after": 60
    }
  }
}
```

The `description` text can differ between responses. Use `code` to detect the error.

## Response Format

Successful responses:

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {}
}
```

Error responses use the same envelope. `code` is a numeric string, and `error.code` repeats it:

```json
{
  "success": false,
  "code": "2001",
  "message": "Invalid API Key",
  "error": {
    "code": "2001",
    "message": "Invalid API Key",
    "description": "Invalid or inactive API Key"
  }
}
```

See [Status and Error Codes](/docs/api/errors/) for the full list.