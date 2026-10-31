# File results

Canonical page: https://billionverify.github.io/docs/api/file-results/

Download verification results for a completed file job.

```http
GET /verify/file/{job_id}/results
```

## Request

### Behavior

- Without filters, the endpoint responds with `307 Temporary Redirect` to the full result file. Follow the redirect (`curl -L`), or you download an empty file.
- With filters, the endpoint returns a generated CSV directly.
- Filters are combined with OR semantics.

### Query parameters

| Parameter    | Type    | Description               |
| ------------ | ------- | ------------------------- |
| `valid`      | boolean | Include valid emails      |
| `invalid`    | boolean | Include invalid emails    |
| `risky`      | boolean | Include risky emails (only results created before 2026-09-11 have this status) |
| `unknown`    | boolean | Include unknown emails    |
| `catchall`   | boolean | Include catch-all emails  |
| `role`       | boolean | Include role emails       |
| `disposable` | boolean | Include disposable emails |

## Example

```bash
curl -L -o filtered-results.csv \
  "https://api.billionverify.com/v1/verify/file/{job_id}/results?valid=true&catchall=true&role=true" \
  -H "BV-API-KEY: sk_xxx"
```