# Check an Email Address from the Command Line

Canonical page: https://billionverify.github.io/guides/check-email-from-command-line/

This guide checks one email address with `curl` and the BillionVerify API, then reads the result with `jq`. It takes about two minutes. You need an API key from your BillionVerify dashboard.

If you only want to check a single address by hand, you do not need a key. The [BillionVerify Email Checker](https://billionverify.com/email-checker) runs the same checks in the browser, 20 times every 24 hours, with no sign-up.

## 1. Store your key

Keep the key out of your shell history and your scripts.

```bash
export BV_API_KEY="sk_your_key_here"
```

## 2. Check one address

```bash
curl -s -X POST https://api.billionverify.com/v1/verify/single \
  -H "BV-API-KEY: $BV_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"jane.doe@example.com"}'
```

When you leave out `check_smtp`, the server default applies, and in production that runs the live mailbox check. Set it to `false` for a fast syntax and MX check only.

## 3. Read the fields that matter

The response has many fields. Most decisions need six of them.

```bash
curl -s -X POST https://api.billionverify.com/v1/verify/single \
  -H "BV-API-KEY: $BV_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"jane.doe@example.com"}' \
  | jq '.data | {status, reason, is_disposable, is_catchall, is_role, is_free}'
```

```json
{
  "status": "valid",
  "reason": "smtp_deliverable",
  "is_disposable": false,
  "is_catchall": false,
  "is_role": false,
  "is_free": false
}
```

## 4. Decide what to do

| `status` | Meaning | Action |
| --- | --- | --- |
| `valid` | The mailbox exists and accepts mail | Send |
| `invalid` | The mailbox does not exist or cannot receive mail | Remove |
| `unknown` | The server gave no clear answer | Retry later |
| `disposable` | A temporary inbox service | Usually remove |
| `catchall` | The domain accepts every address | Send with care and watch bounces |
| `role` | A shared inbox such as `info@` | Usually deliverable, may engage less |

The `is_*` flags are separate from `status`. An address can be `valid` and still have `is_role: true`.

If a result surprises you, open the Email Verifier on [BillionVerify](https://billionverify.com/email-verifier) and run the same address. It shows the mailbox check in the browser, with no key.

## 5. A small loop for a few addresses

```bash
while read -r email; do
  curl -s -X POST https://api.billionverify.com/v1/verify/single \
    -H "BV-API-KEY: $BV_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"$email\"}" \
    | jq -r '.data | [.email, .status, .reason] | @tsv'
done < emails.txt
```

This is fine for a handful of addresses. For a real list, use the batch or file upload endpoints instead. Both are described in the [API reference](/docs/api/), along with rate limits and error codes.
