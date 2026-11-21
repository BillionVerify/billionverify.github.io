# Disposable email check

Canonical page: https://billionverify.github.io/docs/api/disposable-check/

Check whether an email address belongs to a known disposable (temporary) email provider. This endpoint is **free**. It looks up the domain in a built-in list and does not use credits.

```http
POST /verify/disposable
GET  /verify/disposable?email=user@example.com
```

## Request (POST)

```json
{
  "email": "user@example.com"
}
```

### Parameters

| Parameter | Type   | Required | Description            |
| --------- | ------ | -------- | ---------------------- |
| `email`   | string | Yes      | Email address to check (max 254 characters) |

### Notes

- This endpoint does **not** charge credits.
- `check_smtp` is not supported. Sending `check_smtp: true` returns `400`. For SMTP verification use `/verify/single`, `/verify/bulk` or `/verify/file`.
- Accepts both `POST` with a JSON body and `GET` with an `?email=` query parameter.
- A malformed address returns a `400` error, not `is_disposable: false`.
- It only answers whether the domain is on the disposable list. It does not check that the mailbox exists. Use `/verify/single` for that.

This endpoint only reads the disposable list. To [check an address for catch-all risk](https://billionverify.com/email-checker) or to see the role flag next to the disposable flag, use the free checker on the website for one address.

## Response

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "email": "user@mailnull.com",
    "domain": "mailnull.com",
    "is_disposable": true,
    "checked_at": "2026-05-19T10:00:00Z"
  }
}
```

### Response fields

| Field           | Type    | Description                                             |
| --------------- | ------- | ------------------------------------------------------- |
| `email`         | string  | Normalised email address (lowercase)                    |
| `domain`        | string  | Domain portion of the email (ASCII / Punycode)          |
| `is_disposable` | boolean | `true` if the domain is a known disposable provider     |
| `checked_at`    | string  | UTC timestamp of the check (RFC 3339)                   |

For a quick look at one address with a live SMTP check, [BillionVerify's verification tool](https://billionverify.com/email-verifier) is free to use without signing up.