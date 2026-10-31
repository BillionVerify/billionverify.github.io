# Claude Code

Canonical page: https://billionverify.github.io/docs/ai/cli/claude-code/

Claude Code is Anthropic's CLI programming assistant that can directly call BillionVerify to verify emails during your development workflow.

## What is Claude Code?

Claude Code is a terminal-based AI assistant that helps developers with:

- Writing and debugging code
- Running shell commands
- File operations
- API interactions

## Setup

### Option 1: Remote MCP Server (Recommended)

Add the hosted BillionVerify MCP endpoint to Claude Code:

```bash
claude mcp add --transport http billionverify https://mcp.billionverify.com/mcp
```

Then, inside Claude Code, run:

```text
/mcp
```

Complete OAuth authentication in your browser.

The hosted server uses OAuth and does not accept API keys. If you want to authenticate with an API key instead, use the BillionVerify CLI option below.

### Option 2: BillionVerify CLI

Install the BillionVerify CLI:

```bash
npm install -g billionverify
billionverify config set api_key YOUR_API_KEY
```

Claude Code can then run the `billionverify` command directly. See [CLI Tools](/docs/ai/cli/) for the commands.

## Usage Examples

### Verify a Single Email

**You:** Verify if john@example.com is a valid email

**Claude Code:**

```
I'll verify that email address for you.

Using the verify_single_email tool...

✅ john@example.com is VALID
- Deliverable: Yes
- Disposable: No
- Score: 0.95

This email appears to be valid and safe to use.
```

### Verify Multiple Emails

**You:** Check these emails from my .env file

**Claude Code:**

```
Let me read your .env file and verify any email addresses found.

Reading .env...
Found 3 email addresses:
- TEST_EMAIL=user@example.com
- ADMIN_EMAIL=admin@company.com
- SUPPORT_EMAIL=help@temp-mail.org

Verifying each one...

| Email | Status | Notes |
|-------|--------|-------|
| user@example.com | Unknown | Example domain |
| admin@company.com | Valid | Corporate email |
| help@temp-mail.org | Disposable | Disposable email |

Recommendation: Replace the temp-mail address with a real email for testing.
```

### Clean Email List

**You:** Clean the email list in contacts.csv and remove invalid addresses

**Claude Code:**

```
I'll process your contacts.csv file and verify each email.

Reading contacts.csv... Found 150 email addresses.

Verifying emails (this may take a moment)...

Results:
- Valid: 142 emails
- Invalid: 3 emails
- Disposable: 2 emails
- Unknown: 3 emails

Creating cleaned_contacts.csv with valid emails only...
Done! Removed 8 problematic emails.

Emails removed:
- fake@asdfgh.xyz (invalid: domain doesn't exist)
- test@mailinator.com (disposable)
- bounce@invalid.net (invalid: mailbox not found)
...
```

### Check Before Commit

**You:** Before I commit, verify the test emails in my test fixtures

**Claude Code:**

```
Let me check your test fixtures for email addresses.

Scanning test/ directory...
Found emails in test/fixtures/users.json

Verifying 5 test emails...

⚠️ Warning: Found potential issues:

1. test@mailinator.com - This is a disposable email
   Consider using @example.com for test fixtures (RFC 2606 reserved)

2. admin@real-company.com - This appears to be a real company email
   You probably don't want real emails in your test fixtures

Suggestions:
- Use @example.com, @example.org for test emails
- Use user{n}@example.com pattern
- Never use real company emails in tests
```

## Workflow Patterns

### Pre-commit Hook

Ask Claude Code to set up email validation in your workflow:

**You:** Add a pre-commit hook that verifies any new email addresses in config files

**Claude Code:**

```
I'll create a pre-commit hook for email verification.

Creating .husky/pre-commit...

#!/bin/sh
# Check for new email addresses in config files
emails=$(git diff --cached --diff-filter=ACMR -- "*.json" "*.yaml" "*.env*" | \
  grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}')

if [ -n "$emails" ]; then
  echo "Verifying new email addresses..."
  for email in $emails; do
    status=$(billionverify verify "$email" --json | jq -r '.status')
    if [ "$status" = "invalid" ]; then
      echo "❌ Invalid email found: $email"
      exit 1
    fi
  done
  echo "✅ All emails verified"
fi

Done! The hook will verify emails before each commit.
```

### CI/CD Integration

**You:** Add email verification to my GitHub Actions workflow

**Claude Code:**

```
I'll add an email verification step to your workflow.

Updating .github/workflows/ci.yml...

- name: Verify Test Emails
  env:
    BILLIONVERIFY_API_KEY: ${{ secrets.BILLIONVERIFY_API_KEY }}
  run: |
    npm install -g billionverify
    # Extract and verify emails from test fixtures
    grep -rhoE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' test/ | \
      sort -u | while read email; do
        status=$(billionverify verify "$email" --json | jq -r '.status')
        if [ "$status" = "invalid" ]; then
          echo "Invalid email in tests: $email"
          exit 1
        fi
      done

Don't forget to add BILLIONVERIFY_API_KEY to your repository secrets.
```

## Tips

### 1. Natural Language

Just describe what you need:

- "Check if this email is real"
- "Verify the signup emails in my database seed"
- "Find and validate all email addresses in this project"

### 2. Batch Operations

For multiple emails, ask Claude Code to batch them:

- "Verify all emails in contacts.json"
- "Check the email column in users.csv"

### 3. Integration Suggestions

Ask for best practices:

- "How should I validate emails in my signup form?"
- "What's the best way to clean my mailing list?"

## Next Steps

  - [Gemini CLI](/docs/ai/cli/gemini-cli/): Use with Google's Gemini CLI
  - [MCP Integration](/docs/ai/mcp/): Full MCP setup guide
  - [API Reference](/docs/api/): Direct API documentation