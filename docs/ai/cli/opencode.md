# OpenCode

Canonical page: https://billionverify.github.io/docs/ai/cli/opencode/

OpenCode is an open-source AI coding assistant for the terminal. Learn how to use BillionVerify for email verification in your OpenCode workflows.

## What is OpenCode?

OpenCode is a community-driven, open-source AI coding assistant that provides:
- Natural language code generation
- Shell command execution
- Code review and debugging
- Integration with local and cloud LLMs

## Setup

### Install BillionVerify CLI

```bash
npm install -g billionverify
billionverify config set api_key YOUR_API_KEY
```

### Configure Environment

The CLI reads `BILLIONVERIFY_API_KEY`, and the code below uses the same variable. Add it to your shell configuration:

```bash
export BILLIONVERIFY_API_KEY=your-api-key
```

## Usage with OpenCode

### Basic Verification

**You:** Verify if user@company.com is a valid email using the billionverify command

**OpenCode:**
```
I'll run the BillionVerify CLI to check that email.

$ billionverify verify user@company.com

  ✅ user@company.com
  Status:       valid
  Score:        0.92
  Deliverable:  yes
  Domain:       company.com
  Credits:      1

The email is valid and safe to use.
```

### Generate Verification Code

**You:** Create a TypeScript function to verify emails with BillionVerify

**OpenCode:**
```typescript
async function verifyEmail(email: string): Promise<VerificationResult> {
  const response = await fetch('https://api.billionverify.com/v1/verify/single', {
    method: 'POST',
    headers: {
      'BV-API-KEY': process.env.BILLIONVERIFY_API_KEY!,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ email }),
  });

  if (!response.ok) {
    throw new Error(`Verification failed: ${response.statusText}`);
  }

  const { data } = await response.json(); // the API wraps results in { success, code, message, data }
  return data;
}

interface VerificationResult {
  email: string;
  status: 'valid' | 'invalid' | 'unknown' | 'risky' | 'disposable' | 'catchall' | 'role';
  score: number;
  is_deliverable: boolean;
  is_disposable: boolean;
  is_role: boolean;
}
```

### Data Processing

**You:** Write a script to clean up my user database by verifying all emails

**OpenCode:**
```python
#!/usr/bin/env python3
"""
Clean user database by verifying email addresses.
Moves invalid emails to a separate table for review.
"""
import os
import requests
import sqlite3
from typing import List, Tuple

API_KEY = os.environ['BILLIONVERIFY_API_KEY']
API_URL = 'https://api.billionverify.com/v1/verify/single'

def verify_email(email: str) -> dict:
    """Verify a single email address."""
    response = requests.post(
        API_URL,
        headers={
            'BV-API-KEY': API_KEY,
            'Content-Type': 'application/json',
        },
        json={'email': email}
    )
    return response.json()['data']

def process_users(db_path: str) -> Tuple[int, int, int]:
    """Process all users and categorize by email validity."""
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()

    # Create table for invalid emails if not exists
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS invalid_emails (
            user_id INTEGER,
            email TEXT,
            reason TEXT,
            verified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')

    # Get all users
    cursor.execute('SELECT id, email FROM users')
    users = cursor.fetchall()

    valid = 0
    invalid = 0
    unknown = 0

    for user_id, email in users:
        result = verify_email(email)

        if result['status'] == 'valid':
            valid += 1
        elif result['status'] == 'invalid':
            invalid += 1
            cursor.execute(
                'INSERT INTO invalid_emails (user_id, email, reason) VALUES (?, ?, ?)',
                (user_id, email, 'invalid')
            )
        else:
            unknown += 1

        # Rate limiting
        import time
        time.sleep(0.1)

    conn.commit()
    conn.close()

    return valid, invalid, unknown

if __name__ == '__main__':
    valid, invalid, unknown = process_users('users.db')
    print(f'Results: {valid} valid, {invalid} invalid, {unknown} unknown')
```

### CI/CD Integration

**You:** Add email verification to my GitHub Actions workflow

**OpenCode:**
```yaml
# .github/workflows/verify-test-emails.yml
name: Verify Test Emails

on:
  pull_request:
    paths:
      - 'test/**'
      - 'fixtures/**'

jobs:
  verify-emails:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install BillionVerify CLI
        run: npm install -g billionverify

      - name: Extract and verify emails
        env:
          BILLIONVERIFY_API_KEY: ${{ secrets.BILLIONVERIFY_API_KEY }}
        run: |
          # Find all email addresses in test files
          emails=$(grep -rhoE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' test/ fixtures/ | sort -u)

          echo "Found emails to verify:"
          echo "$emails"

          # Verify each email
          failed=0
          for email in $emails; do
            # Skip example.com and test domains
            if [[ "$email" =~ @(example\.(com|org|net)|test\.com)$ ]]; then
              echo "⏭️ Skipping test domain: $email"
              continue
            fi

            result=$(billionverify verify "$email" --json 2>/dev/null)
            status=$(echo "$result" | jq -r '.status')

            if [ "$status" = "valid" ]; then
              echo "⚠️ Real email found in tests: $email"
              failed=1
            fi
          done

          if [ $failed -eq 1 ]; then
            echo "::error::Real email addresses found in test files!"
            exit 1
          fi

          echo "✅ No real emails in test files"
```

## Automation Recipes

### Batch Processing Script

```bash
#!/bin/bash
# verify-list.sh - Verify a list of emails and output results as CSV

input_file="${1:-emails.txt}"
output_file="${2:-results.csv}"

echo "email,status,score,disposable,role" > "$output_file"

while IFS= read -r email; do
    [ -z "$email" ] && continue

    result=$(billionverify verify "$email" --json 2>/dev/null)

    status=$(echo "$result" | jq -r '.status')
    score=$(echo "$result" | jq -r '.score')
    disposable=$(echo "$result" | jq -r '.is_disposable')
    role=$(echo "$result" | jq -r '.is_role')

    echo "$email,$status,$score,$disposable,$role" >> "$output_file"

    # Progress indicator
    echo -n "."
done < "$input_file"

echo ""
echo "Results saved to $output_file"
```

### Webhook Handler

```javascript
// webhook-handler.js - Verify emails from webhook events

const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook/new-signup', async (req, res) => {
  const { email, userId } = req.body;

  // Verify the email
  const verifyResponse = await fetch('https://api.billionverify.com/v1/verify/single', {
    method: 'POST',
    headers: {
      'BV-API-KEY': process.env.BILLIONVERIFY_API_KEY,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ email }),
  });

  const { data: result } = await verifyResponse.json();

  if (result.status === 'invalid' || result.is_disposable) {
    // Flag the user for review
    console.log(`Flagged user ${userId}: ${email} - ${result.status}`);
    // await flagUser(userId, 'suspicious_email');
  }

  res.json({ verified: true, result });
});

app.listen(3000);
```

## Next Steps

  - [Droid](/docs/ai/cli/droid/): Droid terminal assistant
  - [Claude Code](/docs/ai/cli/claude-code/): Claude Code integration
  - [API Reference](/docs/api/): Full API documentation