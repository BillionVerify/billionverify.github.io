# Droid

Canonical page: https://billionverify.github.io/docs/ai/cli/droid/

Droid is an AI-powered terminal assistant for developers. Learn how to integrate BillionVerify for email verification in your Droid workflows.

## What is Droid?

Droid is a modern AI terminal assistant that provides:
- Intelligent command suggestions
- Code generation and explanation
- Shell automation
- Multi-model support (GPT-4, Claude, local models)

## Setup

### Install BillionVerify CLI

```bash
npm install -g billionverify
billionverify config set api_key YOUR_API_KEY
```

### Environment Setup

The CLI reads `BILLIONVERIFY_API_KEY`, and the code below uses the same variable:

```bash
export BILLIONVERIFY_API_KEY=your-api-key
```

## Usage with Droid

### Quick Verification

**You:** verify john@example.com with billionverify

**Droid:**
```
Running: billionverify verify john@example.com --json

{
  "email": "john@example.com",
  "status": "valid",
  "score": 0.88,
  "is_deliverable": true,
  "is_disposable": false,
  "is_role": false,
  ...
}

Email is valid (score 0.88)
```

### Batch Operations

**You:** verify all emails in contacts.csv and create a report

**Droid:**
```
I'll process the CSV and verify each email.

Reading contacts.csv... 250 entries found.

Progress: ████████████████████ 100%

Summary Report:
┌──────────────┬───────┬────────┐
│ Status       │ Count │ %      │
├──────────────┼───────┼────────┤
│ Valid        │ 198   │ 79.2%  │
│ Invalid      │ 32    │ 12.8%  │
│ Unknown      │ 12    │ 4.8%   │
│ Disposable   │ 8     │ 3.2%   │
└──────────────┴───────┴────────┘

Generated files:
- contacts_valid.csv (198 emails)
- contacts_invalid.csv (32 emails)
- contacts_review.csv (20 emails)

Would you like me to explain any of the invalid emails?
```

### Code Generation

**You:** create a Node.js API endpoint for email verification

**Droid:**
```typescript
// routes/verify.ts
import { Router } from 'express';
import rateLimit from 'express-rate-limit';

const router = Router();

// Rate limit: 100 requests per minute
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  message: { error: 'Too many requests' }
});

router.post('/verify', limiter, async (req, res) => {
  const { email } = req.body;

  if (!email) {
    return res.status(400).json({ error: 'Email is required' });
  }

  try {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BILLIONVERIFY_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });

    const { data: result } = await response.json(); // the API wraps results in { success, code, message, data }

    return res.json({
      email,
      isValid: result.status === 'valid',
      score: result.score,
      flags: {
        disposable: result.is_disposable,
        role: result.is_role,
      }
    });
  } catch (error) {
    return res.status(500).json({ error: 'Verification service unavailable' });
  }
});

export default router;
```

Saved to routes/verify.ts

### Shell Automation

**You:** create a cron job that verifies new signups daily

**Droid:**
```bash
# Create the verification script
cat > /usr/local/bin/verify-daily-signups.sh << 'EOF'
#!/bin/bash
set -e

LOG_FILE="/var/log/email-verification.log"
DB_HOST="${DB_HOST:-localhost}"
DB_NAME="${DB_NAME:-myapp}"

echo "$(date): Starting daily email verification" >> $LOG_FILE

# Get unverified emails from database
emails=$(psql -h $DB_HOST -d $DB_NAME -t -c \
  "SELECT email FROM users WHERE email_verified IS NULL AND created_at > NOW() - INTERVAL '1 day'")

count=0
invalid=0

for email in $emails; do
  result=$(billionverify verify "$email" --json 2>/dev/null)
  status=$(echo "$result" | jq -r '.status')

  if [ "$status" = "valid" ]; then
    psql -h $DB_HOST -d $DB_NAME -c \
      "UPDATE users SET email_verified = true WHERE email = '$email'"
  else
    psql -h $DB_HOST -d $DB_NAME -c \
      "UPDATE users SET email_verified = false, email_status = '$status' WHERE email = '$email'"
    ((invalid++))
  fi
  ((count++))

  sleep 0.1  # Rate limiting
done

echo "$(date): Verified $count emails, $invalid invalid" >> $LOG_FILE
EOF

chmod +x /usr/local/bin/verify-daily-signups.sh

# Add to crontab (runs at 2 AM daily)
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/verify-daily-signups.sh") | crontab -

echo "✅ Cron job created. Will run daily at 2 AM."
```

### Integration Testing

**You:** write integration tests for email verification

**Droid:**
```typescript
// __tests__/email-verification.test.ts
import { describe, it, expect, beforeAll } from 'vitest';

describe('Email Verification Integration', () => {
  const API_URL = 'https://api.billionverify.com/v1/verify/single';
  let apiKey: string;

  beforeAll(() => {
    apiKey = process.env.BILLIONVERIFY_API_KEY!;
    if (!apiKey) {
      throw new Error('BILLIONVERIFY_API_KEY not set');
    }
  });

  async function verifyEmail(email: string) {
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'BV-API-KEY': apiKey,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });
    const { data } = await response.json();
    return data;
  }

  it('returns valid for known good email', async () => {
    const result = await verifyEmail('test@gmail.com');
    expect(result.status).toBe('valid');
    expect(result.score).toBeGreaterThan(0.5);
  });

  it('detects disposable emails', async () => {
    const result = await verifyEmail('test@mailinator.com');
    expect(result.is_disposable).toBe(true);
  });

  it('detects role-based emails', async () => {
    const result = await verifyEmail('info@example.com');
    expect(result.is_role).toBe(true);
  });

  it('handles invalid domains', async () => {
    const result = await verifyEmail('user@thisisnotarealdomain12345.xyz');
    expect(result.status).toBe('invalid');
  });

  it('returns proper structure', async () => {
    const result = await verifyEmail('user@example.com');

    expect(result).toHaveProperty('email');
    expect(result).toHaveProperty('status');
    expect(result).toHaveProperty('score');
    expect(result).toHaveProperty('is_deliverable');
    expect(result).toHaveProperty('is_disposable');
    expect(result).toHaveProperty('is_role');
  });
});
```

## Shell Aliases

Add these to your shell configuration (`~/.zshrc` or `~/.bashrc`). Droid runs commands in your shell, so it can use them too:

```bash
# Verify an email address
vmail() {
  billionverify verify "$1"
}

# Verify an email and output JSON
vmail-json() {
  billionverify verify "$1" --json | jq .
}

# Upload a file, wait for the job and download the results
vmail-batch() {
  local task_id
  task_id=$(billionverify verify -f "$1" --json | jq -r '.task_id')
  billionverify status "$task_id" --wait
  billionverify download "$task_id" -o results.csv
}

# Quick check if an email is valid (returns 0 or 1)
vmail-check() {
  [ "$(billionverify verify "$1" --json | jq -r '.status')" = "valid" ]
}
```

Usage:
```
vmail user@example.com
vmail-batch contacts.csv
```

## Next Steps

  - [Claude Code](/docs/ai/cli/claude-code/): Anthropic's CLI assistant
  - [Agent Skills](/docs/ai/agent-skills/): Build AI agent capabilities
  - [API Reference](/docs/api/): Full API documentation