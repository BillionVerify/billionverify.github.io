# Verify in bulk

Canonical page: https://billionverify.github.io/docs/api/verify-bulk/

Verify multiple email addresses synchronously. Use this for small batches of up to `50` emails.

```http
POST /verify/bulk
```

## Request

```json
{
  "emails": ["user1@example.com", "user2@example.com"],
  "check_smtp": false
}
```

### Parameters

| Parameter    | Type    | Required | Default | Description                            |
| ------------ | ------- | -------- | ------- | -------------------------------------- |
| `emails`     | array   | Yes      | -       | Array of email addresses (max: `50`)   |
| `check_smtp` | boolean | No       | server default | `true` runs a live SMTP check, `false` skips it. When omitted, the server default applies (SMTP check on in production) |
| `force_refresh` | boolean | No    | `false` | Bypass cached results for every address in the batch |
| `include_domain_reputation` | boolean | No | `false` | Also fetch MX IP blacklist status for cached results |

Sending more than 50 emails returns HTTP `400` with code `3103`. Use [file upload](/docs/api/upload-file/) for larger lists.

Billing is the same as for single verification: `unknown` results, addresses with invalid syntax and known disposable domains are not charged. Disposable domains are charged only when you explicitly send `check_smtp: true` and the SMTP check returns a definite result. Every other status costs 1 credit.

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "results": [
      {
        "email": "user1@example.com",
        "status": "valid",
        "score": 0.95,
        "is_deliverable": true,
        "credits_used": 1
      },
      {
        "email": "user2@example.com",
        "status": "invalid",
        "score": 0,
        "is_deliverable": false,
        "credits_used": 1
      }
    ],
    "total_emails": 2,
    "valid_emails": 1,
    "invalid_emails": 1,
    "credits_used": 2,
    "process_time": 1500,
    "deliverability_buckets": {
      "strict_deliverable": 1,
      "usable_with_catchall": 1,
      "not_deliverable": 1,
      "unknown": 0,
      "anti_verified": 0,
      "transient_failure": 0
    }
  }
}
```

Each item in `results` has the same fields as a [single verification](/docs/api/verify-single/) result (shortened in this example). `process_time` is in milliseconds.

`deliverability_buckets` groups the results: `strict_deliverable` is valid plus role, `usable_with_catchall` adds catch-all, `not_deliverable` is invalid plus disposable, and `unknown` counts the unknown results. `anti_verified` and `transient_failure` are subsets of `unknown`.

The response can also include `retryable_emails` (addresses worth retrying with `force_refresh`), `deferred_emails` and `retry_after_seconds`. They are omitted when empty.