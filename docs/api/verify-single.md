# Verify a single email

Canonical page: https://billionverify.github.io/docs/api/verify-single/

Verify a single email synchronously.

```http
POST /verify/single
```

## Request

```json
{
  "email": "user@example.com",
  "check_smtp": false
}
```

### Parameters

| Parameter       | Type    | Required | Default | Description                                                        |
| --------------- | ------- | -------- | ------- | ------------------------------------------------------------------ |
| `email`         | string  | Yes      | -       | Email address to verify                                            |
| `check_smtp`    | boolean | No       | server default | `true` runs a live SMTP mailbox check, `false` skips it. When omitted, the server default applies (SMTP check on in production) |
| `force_refresh` | boolean | No       | `false` | Bypass cached results and force a fresh live verification          |
| `include_domain_reputation` | boolean | No | `false` | When a cached result is returned, also fetch the MX IP blacklist status so `domain_reputation` is filled |

### Notes

- Omit `check_smtp` to use the server default. Send `check_smtp: false` to skip live SMTP network checks for this request, for example when you only need syntax, DNS and MX results.
- A known disposable domain returns `disposable` without an SMTP check and is not charged. If you explicitly send `check_smtp: true`, the full check runs and a definite result is charged.
- `force_refresh` defaults to `false`. Set it to `true` only when you need a fresh result. A billable result is charged even if a cached one exists.
- Billing: `unknown` results, addresses with invalid syntax and known disposable domains (without explicit `check_smtp: true`) are not charged. Every other status costs 1 credit. See [Verification Reasons](/docs/verification-reasons/) for `reason` values.

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "email": "user@example.com",
    "status": "valid",
    "score": 0.95,
    "is_deliverable": true,
    "is_disposable": false,
    "is_catchall": false,
    "is_role": false,
    "is_free": false,
    "domain": "example.com",
    "domain_age": 10,
    "mx_records": ["mail.example.com"],
    "check_smtp": true,
    "smtp_result": "deliverable",
    "catch_all_evidence": "direct_no",
    "reason": "smtp_deliverable",
    "response_time": 250,
    "credits_used": 1,
    "verification_mode": "smtp",
    "retryable": false,
    "smtputf8": false
  }
}
```

### Response fields

| Field | Type | Description |
| ----- | ---- | ----------- |
| `email` | string | The verified address |
| `status` | string | `valid`, `invalid`, `unknown`, `disposable`, `catchall`, `role`. Results created before 2026-09-11 can also be `risky` |
| `score` | number | Quality score from 0 to 1. A heuristic, not a probability |
| `is_deliverable` | boolean | The check confirmed the address can receive mail. Can be `true` together with `is_catchall` or `is_role` |
| `is_disposable`, `is_catchall`, `is_role`, `is_free` | boolean | Address type flags. `is_free` marks free mailbox providers |
| `domain` | string | Domain part of the address |
| `domain_age` | integer | Domain age in years. Omitted when unknown |
| `mx_records` | array | MX hosts. Omitted when none were found |
| `mx_ip`, `mx_asn`, `mx_as_org` | string / integer | Primary MX details. Omitted when unknown |
| `domain_reputation` | object | MX IP blacklist info (`mx_ip`, `is_listed`, `blacklists`, `checked`). Present only when reputation was checked |
| `check_smtp` | boolean | Whether the result includes SMTP data |
| `smtp_result` | string | `not_checked`, `deliverable`, `undeliverable` or `inconclusive` |
| `catch_all_evidence` | string | How the catch-all probe behaved: `direct_no`, `cached_no`, `direct_yes`, `cached_yes`, `blocked`, `tempfail`, `unknown`. Omitted when no SMTP probe ran |
| `reason` | string | Why the address got this status. See [Verification Reasons](/docs/verification-reasons/) |
| `domain_suggestion` | string | Suggested domain when a typo is detected. Omitted otherwise |
| `alias_of` | string | Canonical address for Gmail dot / plus aliases and Microsoft plus-addressing. Omitted when the address is already canonical |
| `response_time` | integer | Processing time in milliseconds |
| `credits_used` | integer | Credits charged for this request |
| `verification_mode` | string | Path that produced the result, such as `smtp`, `yahoo_api`, `syntax_only` or a `cache_hit_*` value |
| `retryable` | boolean | `true` when a retry with `force_refresh` may give a different result. Only `unknown` results can be retryable |
| `retry_after_seconds` | integer | Delay suggested by the remote server for a temporary failure. Omitted when none was given |
| `smtputf8` | boolean | `true` when the local part has non-ASCII characters, so your mail server must support SMTPUTF8 |
| `smtp_response` | string | Redacted SMTP response. Returned only to allowlisted accounts |