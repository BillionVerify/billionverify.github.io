# Email Verification in Node.js and Python

Canonical page: https://billionverify.github.io/guides/email-verification-node-python/

Adding email verification to a [Node.js](https://billionverify.com/blog/email-verification-nodejs) or [Python](https://billionverify.com/blog/email-verification-python) app follows the same shape in both. Your backend sends the address to an [email verification API](https://billionverify.com/email-verification-api) over HTTPS and reads a structured result. Your code then decides whether to accept, flag or reject the address. The browser never sees the API key.

**Key facts (checked October 2026):**

- BillionVerify's bulk endpoint takes up to 50 addresses per request.
- The file endpoint accepts CSV, TXT and XLSX files and runs in the background.
- Unknown results and syntax failures cost nothing.
- A re-check of the same address within 24 hours is free.

## How do you set up the client?

In Node.js, version 18 or higher gives you [native fetch](https://nodejs.org/api/globals.html), so a basic call needs no extra package. Older versions can use node-fetch or axios. In Python, the guide assumes version 3.8 or higher and the [requests library](https://requests.readthedocs.io/en/latest/), with aiohttp for async work. In both cases the key lives in an environment variable loaded from a .env file, never in source control or client-side code.

The response gives you more than a yes or no. In the newest [BillionVerify guide](https://billionverify.com/blog/real-time-email-validation-api), the result carries a status such as valid, invalid, disposable, catchall, role or unknown, plus a reason and flags such as is_disposable and is_role. Older tutorials in the series show different field names, so copy names from the current [API reference](/docs/api/).

## What patterns hold up in production?

### Retries

Both tutorials build a reusable client class instead of scattering raw calls. The class handles authentication, parses the response and retries when the API answers with a rate limit error. Retries use exponential backoff, so each attempt waits longer than the last.

### Caching

Caching comes next. The Node.js guide suggests a 24-hour cache for repeat addresses, with Redis or Memcached once you run several instances. BillionVerify also says re-checking an address your account verified in the last 24 hours is free.

### Framework layers

Each language adds its own layer:

- **Node.js:** Express middleware validates the address before it reaches your route handler.
- **Python:** a Flask decorator or a Django form validator, plus pandas for CSV files.
- **Python async:** asyncio with aiohttp when you must check many addresses quickly.

For Python lists, the guide picks a strategy by volume. Sequential calls suit small lists, threads suit medium ones, and the batch API suits large ones.

## How should you handle batches and unknown results?

For many addresses, BillionVerify documents [POST /verify/bulk](/docs/api/verify-bulk/) for up to 50 addresses per request. For large files, [POST /verify/file](/docs/api/upload-file/) accepts a CSV, TXT or XLSX file and runs in the background. A [webhook](/docs/api/webhooks/) can tell you when a file job completes, so you do not have to poll.

Set a timeout on every call. When it fires, or when the result is unknown or catchall, let the user through, store a flag and re-check later. Block only clear failures, such as invalid syntax or a domain with no mail records. Unknown results cost nothing in BillionVerify, and neither do syntax failures.

Test with mocked HTTP responses so unit tests never spend credits. Log outcomes, response times and error rates, and alert on a spike in authentication failures. A [BillionVerify account](https://billionverify.com/auth/sign-up) gets 20 free credits on each day you use your account, up to 600 a month.

## Read the full guides on BillionVerify

- [Email Verification API with Node.js Tutorial](https://billionverify.com/blog/email-verification-nodejs)
- [Email Verification API with Python Guide](https://billionverify.com/blog/email-verification-python)
- [Real-Time Email Validation API: A Developer's Guide](https://billionverify.com/blog/real-time-email-validation-api)
- [Email Verification API: Developer Guide](https://billionverify.com/blog/email-verification-api)
