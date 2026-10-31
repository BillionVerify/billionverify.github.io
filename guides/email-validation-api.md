# Email Validation API: How It Works and How to Use It

Canonical page: https://billionverify.github.io/guides/email-validation-api/

An [email validation API](https://billionverify.com/email-validation-api) is a web service your app calls to check an address before it enters your database. It returns a status and the signals behind it, so your code can accept, flag or reject the address. The aim is to stop bad records at the point of entry, instead of cleaning them after a campaign bounces.

**Key facts (checked October 2026):**

- BillionVerify returns one status per address: valid, invalid, disposable, catchall, role or unknown.
- Unknown results and syntax failures cost nothing.
- The bulk endpoint takes up to 50 addresses per request.
- The disposable check endpoint does not use credits.

## What does an email validation API check?

A good API stacks checks from cheap to expensive.

### Syntax, domain and SMTP

Syntax comes first. It rejects malformed input but proves nothing about the mailbox. The domain check follows, and a domain with no MX records cannot receive mail. The deepest check asks the recipient's mail server over [SMTP](https://www.rfc-editor.org/rfc/rfc5321) whether the mailbox would accept a message, without sending one. It is the slowest step because it depends on someone else's server.

### Risk signals

Risk signals sit on top. These flag [disposable addresses](https://billionverify.com/disposable-email-detection), role accounts such as info@ and [catch-all domains](https://billionverify.com/catch-all-verifier) that accept any address. [AWS SES](https://docs.aws.amazon.com/ses/latest/dg/email-validation.html), for comparison, returns HIGH, MEDIUM or LOW verdicts with flags for role addresses and disposable domains, as described in BillionVerify's guide.

## What does each status mean?

BillionVerify's [real-time guide](https://billionverify.com/blog/real-time-email-validation-api) suggests these defaults for a signup form.

| Status | Meaning | Default action |
| --- | --- | --- |
| valid | Mailbox exists | Accept |
| invalid | Address cannot receive mail | Block and ask again |
| disposable | Temporary inbox | Block or limit |
| catchall | Domain accepts every address | Accept and monitor |
| role | Shared inbox | Accept, maybe flag |
| unknown | Could not confirm | Accept, re-check later |

### Handling unknown results

Do not collapse unknown into invalid. Corporate servers often greylist or rate-limit SMTP checks, and real people sit behind many unknown results. Fail closed on clear errors and fail open on uncertainty. Set a timeout on the call, and when it fires, treat the result as unknown.

## Should you use real-time or bulk validation?

Real-time validation runs on one address inside a user request, so it has a strict time budget. Call it from your backend, validate on submit and not on every keystroke, and keep the API key off the browser. Bulk validation runs on a whole list in the background, for cleaning a CRM or a list before a campaign. Most teams use both.

BillionVerify documents [POST /verify/bulk](/docs/api/verify-bulk/) for up to 50 addresses per request, and [POST /verify/file](/docs/api/upload-file/) for CSV, TXT or XLSX files. Webhooks announce when a file job completes. A separate [disposable-only endpoint](/docs/api/disposable-check/) answers one question and does not use credits.

On cost, a single check normally uses 1 credit. Unknown results and syntax failures are free, and a re-check within 24 hours is free. A [BillionVerify account](https://billionverify.com/auth/sign-up) gets 20 free credits on each day you use your account, up to 600 a month. Read endpoint and authentication details from the current [API documentation](/docs/api/), since older tutorials differ.

## Read the full guides on BillionVerify

- [Email Validation API: Secure Signups, Lower Bounces](https://billionverify.com/blog/email-validation-api)
- [Real-Time Email Validation API: A Developer's Guide](https://billionverify.com/blog/real-time-email-validation-api)
- [Verify Email Address API: A Complete Developer Guide 2026](https://billionverify.com/blog/verify-email-address-api)
- [Email Verification API: Developer Guide](https://billionverify.com/blog/email-verification-api)
