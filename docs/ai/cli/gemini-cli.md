# Gemini CLI

Canonical page: https://billionverify.github.io/docs/ai/cli/gemini-cli/

Gemini CLI is Google's AI assistant for the terminal. Learn how to use BillionVerify for email verification in your Gemini CLI workflows.

## What is Gemini CLI?

Gemini CLI brings Google's Gemini AI models to your terminal for:
- Code generation and debugging
- Shell command execution
- File operations and analysis
- API interactions

## Setup

### Install BillionVerify CLI

First, install the BillionVerify CLI tool:

```bash
npm install -g billionverify
billionverify config set api_key YOUR_API_KEY
```

### Environment Variable

Alternatively, set the environment variable:

```bash
export BILLIONVERIFY_API_KEY=your-api-key
```

## Usage with Gemini CLI

### Verify Single Email

**You:** Use the billionverify command to verify john@google.com

**Gemini:**
```
I'll run the BillionVerify CLI to check that email.

$ billionverify verify john@google.com --json

{
  "email": "john@google.com",
  "status": "valid",
  "score": 0.95,
  "is_deliverable": true,
  "is_disposable": false,
  "is_role": false,
  ...
}

The email john@google.com came back as valid with a score of 0.95.
It's deliverable and not a disposable or role-based address.
```

### Batch Verification

**You:** Verify all emails in my newsletter-subscribers.txt file

**Gemini:**
```
I'll verify each email in your subscriber list.

$ cat newsletter-subscribers.txt | head -5
alice@gmail.com
bob@company.org
test@mailinator.com
jane@outlook.com
invalid@asdfgh.xyz

$ billionverify verify -f newsletter-subscribers.txt --json
{"task_id": "task_abc123", ...}

$ billionverify status task_abc123 --wait
Job task_abc123
Status:    completed
Total:     500
Valid:     423
Invalid:   52

$ billionverify download task_abc123 -o results.csv

Summary:
- Valid: 423
- Invalid: 52
- Other (unknown, catch-all, role, disposable): 25

Would you like me to:
1. Create a cleaned list with only valid emails?
2. Generate a report of invalid addresses?
3. Show the emails that need manual review?
```

### Scripting with Gemini

**You:** Write a bash script that verifies emails from stdin and outputs only valid ones

**Gemini:**
```bash
#!/bin/bash
# verify-emails.sh - Filter valid emails from stdin

while IFS= read -r email; do
    # Skip empty lines
    [ -z "$email" ] && continue

    # Verify the email
    status=$(billionverify verify "$email" --json 2>/dev/null | jq -r '.status')

    # Output only emails with status "valid"
    if [ "$status" = "valid" ]; then
        echo "$email"
    fi
done

# Usage:
# cat emails.txt | ./verify-emails.sh > valid-emails.txt
```

Let me save this script for you.

### Integration with Google Workspace

**You:** Help me verify the email addresses I'm about to add to my Google Contacts

**Gemini:**
```
I'll verify the emails before you import them.

Please provide the emails (paste them or give me a file path).

[User pastes emails]

Verifying 10 email addresses...

Results:
| Email | Status | Safe to Add? |
|-------|--------|--------------|
| john.doe@gmail.com | Valid | ✅ Yes |
| sales@tempmail.io | Disposable | ❌ No (disposable) |
| info@company.com | Valid | ⚠️ Role-based |
| test@invalid.xyz | Invalid | ❌ No (domain error) |
...

Recommendations:
- 7 emails are safe to add
- Remove 2 invalid emails
- Consider if you want role-based addresses (info@, support@)
```

## Shell Functions

Add these to your shell configuration:

```bash
# Quick verify with Gemini
gverify() {
    gemini "Use billionverify to verify this email and explain the result: $1"
}

# Batch verify with summary
gverify-batch() {
    gemini "Verify all emails in $1 using the billionverify CLI and provide a summary report"
}

# Check email before sending
email-check() {
    local status=$(billionverify verify "$1" --json 2>/dev/null | jq -r '.status')
    if [ "$status" = "valid" ]; then
        echo "✅ $1 is valid"
    else
        echo "❌ $1 may have issues - running detailed check..."
        gemini "Verify $1 with billionverify and explain why it might be problematic"
    fi
}
```

## Automation Examples

### Pre-send Verification

```bash
# Before sending a campaign, verify the list
gemini "Read marketing-list.csv, verify all emails in the 'email' column using billionverify,
and create a new file with only the valid ones"
```

### Data Cleanup

```bash
# Clean up a database export
gemini "Process exported-users.json, verify each email field,
and generate a report of users with invalid emails"
```

### CI Integration

```yaml
# .github/workflows/verify-emails.yml
- name: Verify test emails
  run: |
    npm install -g billionverify
    gemini "Find all email addresses in test/fixtures/ and verify them with billionverify.
    Fail if any real emails are found in test data."
```

## Best Practices

### 1. Structured Output

Ask Gemini for structured results:

```
"Verify these emails and output as CSV: email,status,score"
```

### 2. Explanations

Get context on results:

```
"Why might test@mailinator.com be flagged as disposable?"
```

### 3. Batch Processing

For large lists, use file input:

```
"Verify the emails in large-list.csv in batches of 50"
```

## Next Steps

  - [Codex CLI](/docs/ai/cli/codex/): Use with OpenAI Codex
  - [Claude Code](/docs/ai/cli/claude-code/): Use with Claude Code
  - [API Reference](/docs/api/): Full API documentation