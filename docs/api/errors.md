# Status and Error Codes

Canonical page: https://billionverify.github.io/docs/api/errors/

## Verification Status

| Status       | Meaning                                          | Recommended Action                   |
| ------------ | ------------------------------------------------ | ------------------------------------ |
| `valid`      | Email exists and can receive messages            | Safe to send                         |
| `invalid`    | Email does not exist or cannot receive           | Remove from list                     |
| `unknown`    | Deliverability could not be confirmed            | Retry later or use caution           |
| `risky`      | Only on results created before 2026-09-11. Deliverable, but the domain was on an external abuse blocklist | Remove from list |
| `disposable` | Temporary or disposable mailbox                  | Usually remove                       |
| `catchall`   | Domain accepts all addresses                     | Monitor bounce behavior              |
| `role`       | Role-based mailbox such as `info@` or `support@` | Usually deliverable, may engage less |

New verifications no longer return `risky`. The blocklist check behind it was removed on 2026-09-11, and `risky` never means "retry later". A check that could not finish is returned as `unknown`, which is not charged.

## Error Codes

Every error response has `success: false`, a numeric string in `code`, and the same code in `error.code`. Branch on `code`, not on the message text.

| Code   | HTTP Status | Description                                        |
| ------ | ----------- | -------------------------------------------------- |
| `0`    | 200         | Success                                            |
| `1000` | 500         | Internal server error                              |
| `1001` | 400         | Invalid request format                             |
| `1002` | 400         | Invalid parameters                                 |
| `1003` | 429         | Rate limit exceeded                                |
| `1004` | 503         | Service temporarily unavailable                    |
| `1005` | 504         | Request timeout                                    |
| `1006` | 413         | Payload too large                                  |
| `1007` | 503         | Service temporarily overloaded, retry              |
| `2000` | 401         | Unauthorized                                       |
| `2001` | 401         | Invalid API key                                    |
| `2002` | 401         | API key expired                                    |
| `2003` | 401         | API key disabled                                   |
| `2004` | 403         | Insufficient permissions                           |
| `3000` | 402         | Insufficient credits                               |
| `3001` | 500         | Credit system error                                |
| `3002` | 402         | Team credit limit reached                          |
| `3100` | 400         | Invalid email format                               |
| `3101` | 404         | Email not found                                    |
| `3102` | 422         | Email verification failed                          |
| `3103` | 400         | Batch size limit exceeded (more than 50 emails)    |
| `3200` | 400         | Unsupported file format                            |
| `3201` | 400         | File exceeds the 20 GB limit                       |
| `3202` | 500         | File processing failed                             |
| `3300` | 404         | Resource not found                                 |
| `3301` | 410         | Resource no longer available                       |
| `4000` | 500         | Database operation failed                          |
| `4001` | 500         | Cache operation failed                             |
| `4002` | 502         | External service call failed                       |
| `4003` | 503         | Upstream provider unavailable                      |

Retry `429`, `503` and `504` responses with backoff. A `429` or `503` can carry a `Retry-After` header.

## Best Practices

### 1. Keep API Keys Out of Source Code

```javascript
const apiKey = process.env.BV_API_KEY;
```

### 2. Use Batch vs File Endpoints Correctly

| Use Case                      | Endpoint         | Limit               |
| ----------------------------- | ---------------- | ------------------- |
| Single real-time verification | `/verify/single` | 1 email             |
| Small synchronous batch       | `/verify/bulk`   | 50 emails           |
| Large asynchronous processing | `/verify/file`   | 20 GB per file, no row limit |

### 3. Prefer Webhooks for File Jobs

Polling is supported, but webhooks are a better fit for long-running file verification workflows.