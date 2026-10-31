# Webhooks

Canonical page: https://billionverify.github.io/docs/api/webhooks/

Use webhooks to receive asynchronous file-job notifications.

## Endpoints

- `POST /webhooks`
- `GET /webhooks`
- `DELETE /webhooks/{webhook_id}`

## Supported Events

- `file.completed`
- `file.failed`

`verification.completed` is not part of the public webhook contract.

## Create Webhook

```http
POST /webhooks
```

### Request

```json
{
  "url": "https://your-app.com/webhooks/billionverify",
  "events": ["file.completed", "file.failed"]
}
```

### Rules

- `url` must be HTTPS and at most 2048 characters
- `events` must contain 1 to 10 supported event names

### Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "id": "673f3bff-81ea-4730-9cef-af1eb7f134bf",
    "url": "https://your-app.com/webhooks/billionverify",
    "events": ["file.completed", "file.failed"],
    "secret": "5c609e6893ab3b65ce8940549a030bc165762fe171e0b38f880bd570099619bf",
    "is_active": true,
    "created_at": "2026-02-04T10:30:00Z",
    "updated_at": "2026-02-04T10:30:00Z"
  }
}
```

> The webhook `secret` is returned only on creation. Store it securely. You need
> it to verify webhook signatures.

## List Webhooks

```http
GET /webhooks
```

### Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "webhooks": [
      {
        "id": "673f3bff-81ea-4730-9cef-af1eb7f134bf",
        "url": "https://your-app.com/webhooks/billionverify",
        "events": ["file.completed", "file.failed"],
        "is_active": true,
        "last_delivery_status": "success",
        "last_delivery_at": "2026-04-01T12:00:00Z",
        "last_error": null,
        "created_at": "2026-02-04T10:30:00Z",
        "updated_at": "2026-02-04T10:30:00Z"
      }
    ],
    "total": 1
  }
}
```

- `last_delivery_status` is `success` or `failed` when a delivery has been attempted.
- `last_delivery_at` is the timestamp of the most recent final delivery outcome.
- `last_error` contains the latest final error message, or `null` when the latest delivery succeeded.
- For webhooks that have never been delivered, all three fields are `null`.

## Delete Webhook

```http
DELETE /webhooks/{webhook_id}
```

### Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "message": "Webhook deleted successfully",
    "webhook_id": "673f3bff-81ea-4730-9cef-af1eb7f134bf"
  }
}
```

## Webhook Delivery Headers

| Header                | Description                              |
| --------------------- | ---------------------------------------- |
| `Content-Type`        | `application/json`                       |
| `X-Webhook-Event`     | Event type                               |
| `X-Webhook-Timestamp` | Unix timestamp in seconds, used in signature verification |
| `X-Webhook-Signature` | `sha256=<hex_digest>`                    |
| `X-Webhook-Delivery-ID` | Stable ID of the delivery. Present on queued deliveries. Use it to ignore duplicates |

Delivery is at-least-once, so the same event can arrive more than once. Respond with any `2xx` status to confirm it. Other responses and timeouts (30 seconds) count as failures, and the delivery is retried with a growing delay for about 24 hours.

## Webhook Payload

```json
{
  "event": "file.completed",
  "timestamp": "2026-02-04T10:35:00Z",
  "data": {
    "job_id": "7143874e-21c5-43c1-96f3-2b52ea41ae6a",
    "status": "completed",
    "file_name": "emails.csv",
    "total_emails": 1000,
    "valid_emails": 850,
    "invalid_emails": 100,
    "role_emails": 30,
    "catchall_emails": 10,
    "unknown_emails": 10,
    "disposable_emails": 0,
    "disposable_charged_emails": 0,
    "credits_used": 990,
    "process_time_seconds": 125.5,
    "download_url": "https://api.billionverify.com/v1/verify/file/7143874e-21c5-43c1-96f3-2b52ea41ae6a/results",
    "direct_download_url": "https://download.example.com/results_7143874e.csv?signature=...",
    "direct_download_expires_at": "2026-02-04T11:35:00Z"
  }
}
```

A `file.failed` event has a smaller `data` object:

```json
{
  "event": "file.failed",
  "timestamp": "2026-02-04T10:35:00Z",
  "data": {
    "job_id": "7143874e-21c5-43c1-96f3-2b52ea41ae6a",
    "file_name": "emails.csv",
    "error_message": "..."
  }
}
```

## Verifying Webhook Signatures

Webhook signatures use HMAC-SHA256 over this exact string:

```text
{timestamp}.{raw_body}
```

The signature header format is:

```text
X-Webhook-Signature: sha256=<hex_digest>
```

Recommended verification flow:

1. Read the raw request body exactly as received.
2. Read `X-Webhook-Timestamp`.
3. Build `{timestamp}.{raw_body}`.
4. Compute HMAC-SHA256 with your webhook secret.
5. Compare against `X-Webhook-Signature`.
6. Reject stale timestamps to reduce replay risk.

Compute the signature from the raw body, not from re-serialized JSON.

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(rawBody, timestamp, signature, secret) {
  const signedPayload = `${timestamp}.${rawBody}`;
  const expectedSignature =
    'sha256=' +
    crypto.createHmac('sha256', secret).update(signedPayload).digest('hex');

  const a = Buffer.from(signature);
  const b = Buffer.from(expectedSignature);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

```python
import hashlib
import hmac

def verify_webhook_signature(raw_body: bytes, timestamp: str, signature: str, secret: str) -> bool:
    signed_payload = timestamp.encode() + b"." + raw_body
    expected = "sha256=" + hmac.new(
        secret.encode(),
        signed_payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)
```