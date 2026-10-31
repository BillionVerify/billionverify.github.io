# TypeScript SDK

Canonical page: https://billionverify.github.io/docs/sdks/typescript/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-node](https://github.com/BillionVerify/billionverify-node)

The TypeScript SDK is the Node.js SDK. It is written in TypeScript and ships its own type definitions, so there is nothing extra to install.

> The TypeScript SDK shares the same package as the Node.js SDK. It requires Node.js 18 or later.

## Installation

```bash
npm install billionverify-sdk
# or
yarn add billionverify-sdk
# or
pnpm add billionverify-sdk
```

## Quick Start

```typescript
import { BillionVerify } from 'billionverify-sdk';
import type { VerifyResponse } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY!,
});

const result: VerifyResponse = await client.verify('user@example.com');
console.log(result.status); // Type: 'valid' | 'invalid' | 'unknown' | 'risky' | 'disposable' | 'catchall' | 'role'
```

## Type Definitions

The package exports these types: `BillionVerifyConfig`, `VerifyOptions`, `VerifyResponse`, `VerificationStatus`, `DomainReputation`, `BatchVerifyOptions`, `BatchVerifyResponse`, `BatchVerifyResultItem`, `FileUploadOptions`, `FileUploadResponse`, `FileJobStatusOptions`, `FileJobStatusResponse`, `FileJobResultsOptions`, `CreditsResponse`, `WebhookConfig`, `WebhookEvent`, `Webhook`, `WebhookPayload`, `HealthCheckResponse`, `ApiError` and `ApiResponse`.

### Client Options

```typescript
interface BillionVerifyConfig {
  apiKey: string;
  baseUrl?: string;   // Default: 'https://api.billionverify.com/v1'
  timeout?: number;   // Default: 30000 (ms)
  retries?: number;   // Default: 3 (maximum attempts for retryable errors)
}
```

### Verification Result

```typescript
type VerificationStatus =
  | 'valid'
  | 'invalid'
  | 'unknown'
  | 'risky'
  | 'disposable'
  | 'catchall'
  | 'role';

interface VerifyResponse {
  email: string;
  status: VerificationStatus;
  score: number;
  is_deliverable: boolean;
  is_disposable: boolean;
  is_catchall: boolean;
  is_role: boolean;
  is_free: boolean;
  domain: string;
  domain_age?: number;
  mx_records?: string[];
  domain_reputation?: DomainReputation;
  smtp_check: boolean;
  reason?: string;
  suggestion?: string;
  response_time: number;
  credits_used: number;
}

interface DomainReputation {
  mx_ip?: string;
  is_listed?: boolean;
  blacklists?: string[];
  checked?: boolean;
}
```

### Bulk Job Types

```typescript
interface BatchVerifyResponse {
  results: BatchVerifyResultItem[];
  total_emails: number;
  valid_emails: number;
  invalid_emails: number;
  credits_used: number;
  process_time: number;
}

interface BatchVerifyResultItem {
  email: string;
  status: VerificationStatus;
  score: number;
  is_deliverable: boolean;
  is_disposable?: boolean;
  is_catchall?: boolean;
  is_role?: boolean;
  is_free?: boolean;
  credits_used: number;
}

interface FileJobStatusResponse {
  job_id: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  file_name: string;
  total_emails: number;
  processed_emails: number;
  progress_percent: number;
  valid_emails: number;
  invalid_emails: number;
  unknown_emails: number;
  role_emails: number;
  catchall_emails: number;
  disposable_emails: number;
  credits_used: number;
  process_time_seconds?: number;
  result_file_path?: string;
  download_url?: string;
  created_at: string;
  completed_at?: string;
  error_message?: string;
}
```

### Error Types

```typescript
class BillionVerifyError extends Error {
  readonly code: string;
  readonly statusCode: number;
  readonly details?: string;
}

class AuthenticationError extends BillionVerifyError {}        // 401
class InsufficientCreditsError extends BillionVerifyError {}   // 402
class NotFoundError extends BillionVerifyError {}              // 404
class ValidationError extends BillionVerifyError {}            // 400
class TimeoutError extends BillionVerifyError {}               // 504

class RateLimitError extends BillionVerifyError {              // 429
  readonly retryAfter: number; // seconds
}
```

## Configuration

```typescript
const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY!,
  timeout: 30000,
  retries: 3,
});
```

## Single Verification

### Basic Usage

```typescript
const result = await client.verify('user@example.com');

// TypeScript knows all properties and their types
if (result.status === 'valid') {
  console.log(`Score: ${result.score}`);
  console.log(`Deliverable: ${result.is_deliverable}`);
}
```

### With Options

```typescript
interface VerifyOptions {
  checkSmtp?: boolean; // Default: true
}

const result = await client.verify('user@example.com', {
  checkSmtp: true,
});
```

### Type Guards

```typescript
function isSafeForMarketing(result: VerifyResponse): boolean {
  return (
    result.status === 'valid' &&
    !result.is_disposable &&
    !result.is_role
  );
}
```

## Bulk Verification

### Submit Job

`verifyBatch()` verifies up to 50 emails per call and returns the results directly. For larger lists, upload a CSV with `uploadFile()`.

```typescript
const emails: string[] = [
  'user1@example.com',
  'user2@example.com',
];

const batch: BatchVerifyResponse = await client.verifyBatch(emails);
```

```typescript
import { createReadStream } from 'node:fs';

const upload: FileUploadResponse = await client.uploadFile(
  createReadStream('emails.csv'),
  'emails.csv',
  { emailColumn: 'email' },
);
```

### Get Status

```typescript
const status: FileJobStatusResponse = await client.getFileJobStatus(upload.task_id);

// Type-safe status checking
switch (status.status) {
  case 'pending':
    console.log('Job is queued');
    break;
  case 'processing':
    console.log(`Progress: ${status.processed_emails}/${status.total_emails}`);
    break;
  case 'completed':
    console.log(`Done! Valid: ${status.valid_emails}, Invalid: ${status.invalid_emails}`);
    break;
  case 'failed':
    console.log(`Job failed: ${status.error_message}`);
    break;
}
```

### Get Results

`getFileJobResults()` returns a `fetch` `Response` whose body is the result CSV. Filter flags are optional.

```typescript
const done = await client.waitForFileJobCompletion(upload.task_id);

if (done.status === 'completed') {
  const response = await client.getFileJobResults(upload.task_id, { valid: true });
  const csv: string = await response.text();
}
```

## Error Handling

```typescript
import {
  BillionVerify,
  BillionVerifyError,
  AuthenticationError,
  RateLimitError,
  ValidationError,
} from 'billionverify-sdk';
import type { VerifyResponse } from 'billionverify-sdk';

async function verifyEmail(email: string): Promise<VerifyResponse | null> {
  try {
    return await client.verify(email);
  } catch (error) {
    if (error instanceof AuthenticationError) {
      console.error('Invalid API key');
    } else if (error instanceof RateLimitError) {
      console.error(`Rate limited. Retry after ${error.retryAfter}s`);
    } else if (error instanceof ValidationError) {
      console.error(`Invalid input: ${error.message}`);
    } else if (error instanceof BillionVerifyError) {
      console.error(`API error: ${error.message}`);
    }
    return null;
  }
}
```

## Generic Utilities

### Type-Safe Validation Function

```typescript
type ValidationResult =
  | { valid: true; score: number }
  | { valid: false; reason: string; message: string };

async function validateEmail(email: string): Promise<ValidationResult> {
  const result = await client.verify(email);

  if (result.status === 'invalid') {
    return {
      valid: false,
      reason: 'invalid_email',
      message: 'This email address is invalid',
    };
  }

  if (result.is_disposable) {
    return {
      valid: false,
      reason: 'disposable',
      message: 'Disposable emails are not allowed',
    };
  }

  return {
    valid: true,
    score: result.score,
  };
}
```

### Custom Result Processing

```typescript
interface ProcessedResult {
  email: string;
  isValid: boolean;
  flags: string[];
}

function processResult(result: VerifyResponse): ProcessedResult {
  const flags: string[] = [];

  if (result.is_disposable) flags.push('disposable');
  if (result.is_role) flags.push('role');
  if (result.is_catchall) flags.push('catchall');
  if (result.is_free) flags.push('free');

  return {
    email: result.email,
    isValid: result.status === 'valid',
    flags,
  };
}
```

## Framework Integration

### Next.js App Router

```typescript
// app/api/verify/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { BillionVerify } from 'billionverify-sdk';
import type { VerifyResponse } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY!,
});

interface RequestBody {
  email: string;
}

interface ResponseBody {
  valid: boolean;
  score?: number;
  message?: string;
}

export async function POST(request: NextRequest): Promise<NextResponse<ResponseBody>> {
  const body: RequestBody = await request.json();

  try {
    const result: VerifyResponse = await client.verify(body.email);

    if (result.status === 'invalid') {
      return NextResponse.json(
        { valid: false, message: 'Invalid email address' },
        { status: 400 }
      );
    }

    return NextResponse.json({
      valid: true,
      score: result.score,
    });
  } catch (error) {
    return NextResponse.json(
      { valid: false, message: 'Verification failed' },
      { status: 500 }
    );
  }
}
```

### NestJS Service

```typescript
// email-verification.service.ts
import { Injectable } from '@nestjs/common';
import { BillionVerify } from 'billionverify-sdk';
import type { VerifyResponse } from 'billionverify-sdk';

@Injectable()
export class EmailVerificationService {
  private client: BillionVerify;

  constructor() {
    this.client = new BillionVerify({
      apiKey: process.env.BV_API_KEY!,
    });
  }

  async verify(email: string): Promise<VerifyResponse> {
    return this.client.verify(email);
  }

  async isValidForRegistration(email: string): Promise<boolean> {
    const result = await this.verify(email);
    return result.status === 'valid' && !result.is_disposable;
  }
}
```

### Express with TypeScript

```typescript
import express, { Request, Response } from 'express';
import { BillionVerify } from 'billionverify-sdk';
import type { VerifyResponse } from 'billionverify-sdk';

const app = express();
app.use(express.json());

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY!,
});

interface VerifyRequest {
  email: string;
}

interface VerifyReply {
  success: boolean;
  data?: VerifyResponse;
  error?: string;
}

app.post('/verify', async (
  req: Request<{}, VerifyReply, VerifyRequest>,
  res: Response<VerifyReply>
) => {
  try {
    const result = await client.verify(req.body.email);
    res.json({ success: true, data: result });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error',
    });
  }
});
```

## Zod Schema Integration

```typescript
import { z } from 'zod';
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY!,
});

const emailSchema = z.string().email().refine(
  async (email) => {
    try {
      const result = await client.verify(email);
      return result.status === 'valid';
    } catch {
      return false;
    }
  },
  { message: 'Email address is not valid or deliverable' }
);

// Usage
async function validateEmail(email: string) {
  return emailSchema.parseAsync(email);
}
```

## Best Practices

### 1. Use Strict Null Checks

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "strictNullChecks": true
  }
}

// Optional fields are typed as possibly undefined
const result = await client.verify(email);
if (result.reason !== undefined) {
  console.log(`Reason: ${result.reason}`);
}
```

### 2. Create Type-Safe Wrappers

```typescript
// verification.ts
export async function verifyAndCategorize(email: string) {
  const result = await client.verify(email);

  return {
    email: result.email,
    category: categorize(result),
    score: result.score,
  } as const;
}

function categorize(result: VerifyResponse) {
  if (result.status !== 'valid') return result.status;
  if (result.is_disposable) return 'disposable';
  if (result.is_role) return 'role';
  return 'valid';
}
```

## Next Steps

  - [Node.js SDK](/docs/sdks/node/): Full Node.js SDK documentation
  - [API Reference](/docs/api/): Detailed API documentation
  - [Integration Guides](https://billionverify.com/docs/integration-guides): Platform integrations