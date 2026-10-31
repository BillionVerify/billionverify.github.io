# Node.js SDK

Canonical page: https://billionverify.github.io/docs/sdks/node/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-node](https://github.com/BillionVerify/billionverify-node)

The Node.js SDK is a small TypeScript client for the BillionVerify API. It requires Node.js 18 or later (it uses the built-in `fetch`) and ships both ESM and CommonJS builds with type definitions.

## Installation

```bash
npm install billionverify-sdk
# or
yarn add billionverify-sdk
# or
pnpm add billionverify-sdk
```

## Quick Start

```javascript
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

// Verify a single email
const result = await client.verify('user@example.com');
console.log(result.status); // 'valid', 'invalid', 'unknown', 'risky', 'disposable', 'catchall', or 'role'
console.log(result.is_deliverable); // true or false
```

## Configuration

### Client Options

```javascript
const client = new BillionVerify({
  apiKey: 'your-api-key', // Required

  // Optional configurations
  timeout: 30000,         // Request timeout in ms (default: 30000)
  retries: 3,             // Maximum attempts for retryable errors (default: 3)
  baseUrl: 'https://api.billionverify.com/v1', // Default; includes /v1
});
```

The constructor throws `AuthenticationError` if `apiKey` is empty. The SDK sends the key in the `EV-API-KEY` header, which the API still accepts alongside `BV-API-KEY`.

### Environment Variables

The SDK does not read environment variables by itself. Keep the key in one and pass it in:

```bash
# .env
BV_API_KEY=your_api_key_here
```

```javascript
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});
```

## Single Email Verification

### Basic Verification

```javascript
const result = await client.verify('user@example.com');

console.log({
  email: result.email,
  status: result.status,
  score: result.score,
  deliverable: result.is_deliverable,
  disposable: result.is_disposable,
  role: result.is_role,
});
```

### Verification with Options

```javascript
const result = await client.verify('user@example.com', {
  checkSmtp: true, // Perform SMTP verification (default: true)
});
```

### Response Structure

```typescript
interface VerifyResponse {
  email: string;
  status: 'valid' | 'invalid' | 'unknown' | 'risky' | 'disposable' | 'catchall' | 'role';
  score: number;
  is_deliverable: boolean;
  is_disposable: boolean;
  is_catchall: boolean;
  is_role: boolean;
  is_free: boolean;
  domain: string;
  domain_age?: number;
  mx_records?: string[];
  domain_reputation?: {
    mx_ip?: string;
    is_listed?: boolean;
    blacklists?: string[];
    checked?: boolean;
  };
  smtp_check: boolean;
  reason?: string;
  suggestion?: string;
  response_time: number;
  credits_used: number;
}
```

## Bulk Verification

### Verify a Batch Synchronously

`verifyBatch()` verifies up to 50 emails in one request and returns the results directly. With more than 50 emails it throws `ValidationError` before calling the API.

```javascript
const emails = [
  'user1@example.com',
  'user2@example.com',
  'user3@example.com',
];

const batch = await client.verifyBatch(emails, { checkSmtp: true });

console.log(`Total: ${batch.total_emails}, valid: ${batch.valid_emails}, invalid: ${batch.invalid_emails}`);
console.log(`Credits used: ${batch.credits_used}`);

for (const item of batch.results) {
  console.log(`${item.email}: ${item.status}`);
}
```

### Upload a File for Larger Lists

For larger lists, upload a CSV file. The job runs asynchronously. `uploadFile()` takes a `Buffer` or a readable stream, plus the file name.

```javascript
import { createReadStream } from 'node:fs';

const upload = await client.uploadFile(
  createReadStream('emails.csv'),
  'emails.csv',
  {
    emailColumn: 'email',   // Column that contains the emails
    checkSmtp: true,
    preserveOriginal: true, // Keep the original columns in the results
  },
);

console.log(`Job ID: ${upload.task_id}`);
console.log(`Status: ${upload.status}`);
```

### Check Job Status

```javascript
const status = await client.getFileJobStatus(upload.task_id);

console.log({
  id: status.job_id,
  status: status.status, // 'pending', 'processing', 'completed', 'failed'
  total: status.total_emails,
  processed: status.processed_emails,
  progress: status.progress_percent,
  valid: status.valid_emails,
  invalid: status.invalid_emails,
});
```

`getFileJobStatus()` also accepts `{ timeout }` (0 to 300 seconds) to long-poll instead of returning immediately.

### Get Results

```javascript
// Poll until the job is completed or failed (every 5 s, up to 10 min by default)
const done = await client.waitForFileJobCompletion(upload.task_id);

if (done.status === 'completed') {
  // Returns a fetch Response whose body is the result CSV.
  // Filters are optional: valid, invalid, catchall, role, unknown, disposable, risky.
  const response = await client.getFileJobResults(upload.task_id, { valid: true });
  const csv = await response.text();
  console.log(csv);
}
```

`waitForFileJobCompletion(jobId, pollInterval, maxWait)` takes milliseconds and throws `TimeoutError` if the job is still running after `maxWait`.

### Webhook Notifications

Instead of polling, register a webhook. It is called when a file job finishes (`file.completed`) or fails (`file.failed`):

```javascript
const webhook = await client.createWebhook({
  url: 'https://your-domain.com/webhooks/billionverify',
  events: ['file.completed', 'file.failed'],
});

console.log(webhook.id);
console.log(webhook.secret); // Returned only when the webhook is created. Store it.

const { webhooks, total } = await client.listWebhooks();
await client.deleteWebhook(webhook.id);
```

Each delivery is a `POST` with a JSON body of the form `{ event, timestamp, data }`. The API signs it with HMAC-SHA256 over `"<timestamp>.<raw body>"`, using the webhook secret, and sends these headers:

- `X-Webhook-Signature`: `sha256=<hex digest>`
- `X-Webhook-Timestamp`: Unix time in seconds, the value used in the signature
- `X-Webhook-Event`: the event name

Verify against the raw body before parsing it:

```javascript
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValidWebhook(rawBody, headers, secret) {
  const timestamp = headers['x-webhook-timestamp'];
  const received = headers['x-webhook-signature'] ?? '';
  const expected =
    'sha256=' + createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');

  const a = Buffer.from(received);
  const b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

> The SDK's `client.verifyWebhookSignature(payload, signature, secret)` signs the body only, without the timestamp prefix, so it does not match what the API sends. Use the function above for real deliveries.

## Credit Management

### Check Available Credits

```javascript
const credits = await client.getCredits();

console.log({
  balance: credits.credits_balance,
  consumed: credits.credits_consumed,
  added: credits.credits_added,
});
```

## Error Handling

### Error Types

All errors extend `BillionVerifyError`, which carries `code`, `statusCode` and an optional `details` string.

| Class | HTTP status | Notes |
| --- | --- | --- |
| `AuthenticationError` | 401 | Missing or invalid API key |
| `InsufficientCreditsError` | 402 | Not enough credits |
| `NotFoundError` | 404 | For example an unknown job ID |
| `ValidationError` | 400 | Also thrown locally for more than 50 emails in `verifyBatch()` |
| `RateLimitError` | 429 | Has `retryAfter` in seconds |
| `TimeoutError` | 504 | Request timed out, or `waitForFileJobCompletion()` ran out of time |

```javascript
import {
  BillionVerify,
  BillionVerifyError,
  AuthenticationError,
  InsufficientCreditsError,
  RateLimitError,
  ValidationError,
} from 'billionverify-sdk';

try {
  const result = await client.verify('user@example.com');
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error('Invalid API key');
  } else if (error instanceof InsufficientCreditsError) {
    console.error('Not enough credits');
  } else if (error instanceof RateLimitError) {
    console.error(`Rate limited. Retry after ${error.retryAfter} seconds`);
  } else if (error instanceof ValidationError) {
    console.error(`Invalid input: ${error.message}`);
  } else if (error instanceof BillionVerifyError) {
    console.error(`API error: ${error.message}`);
  }
}
```

### Retry Logic

The SDK retries 429, 500, 502 and 503 responses with exponential backoff (`2^attempt` seconds, or the `Retry-After` header on 429). `retries` is the maximum number of attempts, including the first one. There are no other retry options.

```javascript
const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
  retries: 5, // Up to 5 attempts for retryable errors
});
```

## Framework Integration

### Express.js

```javascript
import express from 'express';
import { BillionVerify } from 'billionverify-sdk';

const app = express();
app.use(express.json());

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

app.post('/api/verify-email', async (req, res) => {
  const { email } = req.body;

  try {
    const result = await client.verify(email);

    if (result.status === 'invalid') {
      return res.status(400).json({
        valid: false,
        message: 'Please enter a valid email address',
      });
    }

    if (result.is_disposable) {
      return res.status(400).json({
        valid: false,
        message: 'Disposable emails are not allowed',
      });
    }

    res.json({ valid: true, result });
  } catch (error) {
    res.status(500).json({ error: 'Verification failed' });
  }
});
```

### Fastify

```javascript
import Fastify from 'fastify';
import { BillionVerify } from 'billionverify-sdk';

const fastify = Fastify();
const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

fastify.post('/verify', async (request, reply) => {
  const { email } = request.body;
  const result = await client.verify(email);
  return result;
});
```

### Next.js API Route

```javascript
// pages/api/verify.js
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { email } = req.body;

  try {
    const result = await client.verify(email);
    res.status(200).json(result);
  } catch (error) {
    res.status(500).json({ error: 'Verification failed' });
  }
}
```

## Complete Example

### User Registration with Email Verification

```javascript
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

async function validateRegistrationEmail(email) {
  const result = await client.verify(email);

  // Reject invalid emails
  if (result.status === 'invalid') {
    return {
      valid: false,
      reason: 'invalid_email',
      message: 'This email address is invalid',
    };
  }

  // Reject disposable emails
  if (result.is_disposable) {
    return {
      valid: false,
      reason: 'disposable',
      message: 'Please use a permanent email address',
    };
  }

  // Warn about role-based emails
  if (result.is_role) {
    return {
      valid: true,
      warning: 'role_based',
      message: 'Using a personal email is recommended',
    };
  }

  // Accept everything else
  return {
    valid: true,
    score: result.score,
  };
}

// Usage
const validation = await validateRegistrationEmail('user@example.com');
if (!validation.valid) {
  console.error(validation.message);
}
```

## TypeScript Support

The SDK includes TypeScript definitions out of the box:

```typescript
import { BillionVerify } from 'billionverify-sdk';
import type {
  BillionVerifyConfig,
  VerifyResponse,
  BatchVerifyResponse,
  FileJobStatusResponse,
} from 'billionverify-sdk';

const config: BillionVerifyConfig = {
  apiKey: process.env.BV_API_KEY!,
  timeout: 30000,
};

const client = new BillionVerify(config);

const result: VerifyResponse = await client.verify('user@example.com');
```

## Next Steps

  - [API Reference](/docs/api/): Detailed API endpoint documentation
  - [Integration Guides](https://billionverify.com/docs/integration-guides): Platform integrations