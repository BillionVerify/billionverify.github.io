# Codex

Canonical page: https://billionverify.github.io/docs/ai/cli/codex/

OpenAI Codex CLI provides AI-powered code generation and terminal assistance. Learn how to integrate BillionVerify for email verification tasks.

## What is Codex CLI?

Codex CLI is OpenAI's command-line tool that leverages the Codex model for:
- Natural language to code translation
- Shell command generation
- Code completion and debugging
- Task automation

## Setup

### Install BillionVerify CLI

```bash
npm install -g billionverify
billionverify config set api_key YOUR_API_KEY
```

### Configure Environment

The CLI reads `BILLIONVERIFY_API_KEY`, and the code below uses the same variable:

```bash
export BILLIONVERIFY_API_KEY=your-api-key
export OPENAI_API_KEY=your-openai-key
```

## Usage Examples

### Generate Verification Script

**Prompt:** Write a Python script that verifies a list of emails using the BillionVerify API

**Codex Output:**
```python
import os
import requests
from typing import List, Dict

def verify_emails(emails: List[str]) -> List[Dict]:
    """Verify a list of email addresses using BillionVerify API."""
    api_key = os.environ.get('BILLIONVERIFY_API_KEY')
    results = []

    for email in emails:
        response = requests.post(
            'https://api.billionverify.com/v1/verify/single',
            headers={
                'BV-API-KEY': api_key,
                'Content-Type': 'application/json',
            },
            json={'email': email}
        )
        results.append(response.json()['data'])  # the API wraps results in {success, code, message, data}

    return results

def main():
    emails = [
        'test@example.com',
        'user@gmail.com',
        'fake@mailinator.com',
    ]

    results = verify_emails(emails)

    for result in results:
        status = '✅' if result['status'] == 'valid' else '❌'
        print(f"{status} {result['email']}: {result['status']}")

if __name__ == '__main__':
    main()
```

### Generate Shell Command

**Prompt:** One-liner to verify all emails in a CSV file's second column

**Codex Output:**
```bash
cut -d',' -f2 users.csv | tail -n +2 | while read email; do [ "$(billionverify verify "$email" --json | jq -r '.status')" = "valid" ] && echo "valid   $email" || echo "other   $email"; done
```

### Generate Test Cases

**Prompt:** Generate Jest tests for an email verification function

**Codex Output:**
```typescript
import { verifyEmail } from './email-service';

describe('Email Verification', () => {
  beforeAll(() => {
    process.env.BILLIONVERIFY_API_KEY = 'test-api-key';
  });

  test('validates correct email format', async () => {
    const result = await verifyEmail('user@example.com');
    expect(result).toHaveProperty('status');
    expect(result).toHaveProperty('score');
  });

  test('detects disposable emails', async () => {
    const result = await verifyEmail('test@mailinator.com');
    expect(result.is_disposable).toBe(true);
  });

  test('handles API errors gracefully', async () => {
    // Simulate network error
    const result = await verifyEmail('test@example.com');
    expect(result).not.toBeNull();
  });
});
```

### Generate API Integration

**Prompt:** Create a Node.js module for BillionVerify integration with caching

**Codex Output:**
```typescript
import NodeCache from 'node-cache';

interface VerificationResult {
  email: string;
  status: 'valid' | 'invalid' | 'unknown' | 'risky' | 'disposable' | 'catchall' | 'role';
  score: number;
  is_deliverable: boolean;
  is_disposable: boolean;
  is_role: boolean;
}

class BillionVerifyClient {
  private apiKey: string;
  private cache: NodeCache;
  private baseUrl = 'https://api.billionverify.com/v1';

  constructor(apiKey: string, cacheTTL: number = 3600) {
    this.apiKey = apiKey;
    this.cache = new NodeCache({ stdTTL: cacheTTL });
  }

  async verify(email: string): Promise<VerificationResult> {
    // Check cache first
    const cached = this.cache.get<VerificationResult>(email);
    if (cached) return cached;

    const response = await fetch(`${this.baseUrl}/verify/single`, {
      method: 'POST',
      headers: {
        'BV-API-KEY': this.apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });

    const { data } = await response.json();
    const result: VerificationResult = data;

    // Cache the result
    this.cache.set(email, result);

    return result;
  }

  async verifyBulk(emails: string[]): Promise<VerificationResult[]> {
    const response = await fetch(`${this.baseUrl}/verify/bulk`, {
      method: 'POST',
      headers: {
        'BV-API-KEY': this.apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ emails }),
    });

    const { data } = await response.json();
    return data.results; // up to 50 emails per request
  }
}

export default BillionVerifyClient;
```

## Code Generation Patterns

### Pattern 1: Validation Middleware

**Prompt:** Express middleware to validate email in request body

```typescript
import { Request, Response, NextFunction } from 'express';

export const validateEmail = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const { email } = req.body;

  if (!email) {
    return res.status(400).json({ error: 'Email is required' });
  }

  const response = await fetch('https://api.billionverify.com/v1/verify/single', {
    method: 'POST',
    headers: {
      'BV-API-KEY': process.env.BILLIONVERIFY_API_KEY,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ email }),
  });

  const { data: result } = await response.json();

  if (result.status === 'invalid') {
    return res.status(400).json({ error: 'Invalid email address' });
  }

  if (result.is_disposable) {
    return res.status(400).json({ error: 'Disposable emails not allowed' });
  }

  req.verificationResult = result;
  next();
};
```

### Pattern 2: React Hook

**Prompt:** React hook for email verification with debounce

```typescript
import { useState, useEffect } from 'react';
import { useDebouncedCallback } from 'use-debounce';

interface VerificationState {
  isLoading: boolean;
  isValid: boolean | null;
  result: any;
  error: string | null;
}

export function useEmailVerification(debounceMs: number = 500) {
  const [state, setState] = useState<VerificationState>({
    isLoading: false,
    isValid: null,
    result: null,
    error: null,
  });

  const verify = useDebouncedCallback(async (email: string) => {
    if (!email || !email.includes('@')) {
      setState({ isLoading: false, isValid: null, result: null, error: null });
      return;
    }

    setState(prev => ({ ...prev, isLoading: true, error: null }));

    try {
      const response = await fetch('/api/verify-email', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email }),
      });

      const result = await response.json(); // your /api/verify-email route returns the verification data

      setState({
        isLoading: false,
        isValid: result.status === 'valid',
        result,
        error: null,
      });
    } catch (error) {
      setState({
        isLoading: false,
        isValid: null,
        result: null,
        error: 'Verification failed',
      });
    }
  }, debounceMs);

  return { ...state, verify };
}
```

## Next Steps

  - [OpenCode](/docs/ai/cli/opencode/): Open-source alternative
  - [Droid](/docs/ai/cli/droid/): Terminal AI assistant
  - [API Reference](/docs/api/): Full API documentation