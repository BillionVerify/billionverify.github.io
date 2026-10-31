# CLI Tools

Canonical page: https://billionverify.github.io/docs/ai/cli/

AI-powered CLI tools can directly call BillionVerify to verify emails during development, debugging, and automation workflows.

## Supported CLI Tools

  - [Claude Code](/docs/ai/cli/claude-code/): Anthropic's CLI programming assistant
  - [Gemini CLI](/docs/ai/cli/gemini-cli/): Google's AI assistant for the terminal
  - [Codex CLI](/docs/ai/cli/codex/): OpenAI's code generation tool
  - [OpenCode](/docs/ai/cli/opencode/): Open-source AI coding assistant
  - [Droid](/docs/ai/cli/droid/): Terminal AI assistant for developers

## BillionVerify CLI

In addition to AI-powered tools, BillionVerify provides its own CLI for direct verification:

### Installation

```bash
npm install -g billionverify
```

Or run it without installing:

```bash
npx billionverify verify user@example.com
```

The npm package is `billionverify` and the command is `billionverify`. It requires Node.js 18 or later.

### Configuration

```bash
billionverify config set api_key YOUR_API_KEY
```

This saves the key to `~/.billionverify/config.json`. Or use an environment variable:

```bash
export BILLIONVERIFY_API_KEY=your-api-key
```

The CLI looks for the key in this order: the `--api-key <key>` flag, the `BILLIONVERIFY_API_KEY` variable, then the config file.

### Basic Commands

```bash
# Verify single email
billionverify verify user@example.com

# JSON output
billionverify verify user@example.com --json

# Verify multiple emails (up to 50)
billionverify verify user1@example.com user2@test.com

# Upload a CSV file for bulk verification
billionverify verify -f emails.csv --email-column email

# Check a file job, optionally wait for it to finish
billionverify status <task_id> --wait

# Download results, optionally filtered by status
billionverify download <task_id> --valid --invalid -o results.csv

# Check credits
billionverify credits
```

`--no-smtp` skips the SMTP check on `verify`. `download` accepts `--valid`, `--invalid`, `--catchall`, `--role`, `--unknown`, `--disposable` and `--risky`.

### Output Formats

**Human-readable (default):**
```
  ✅ user@example.com
  Status:       valid
  Score:        0.95
  Deliverable:  yes
  Domain:       example.com (10y)
  MX:           mx1.example.com
  Credits:      1
```

**JSON output:**
```bash
billionverify verify user@example.com --json
```
```json
{
  "email": "user@example.com",
  "status": "valid",
  "score": 0.95,
  "is_deliverable": true,
  "is_disposable": false,
  "is_catchall": false,
  "is_role": false,
  "domain": "example.com",
  "credits_used": 1
}
```

The JSON output is the `data` object returned by the API, so it may contain more fields than shown here.

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | The command ran. A verification result of invalid or unknown still exits 0, so read `status` |
| 1 | Error: missing API key, API error, bad arguments, or a file job that ended `failed` with `status --wait` |

Use `--json` in scripts and read `status`:

```bash
status=$(billionverify verify user@example.com --json | jq -r '.status')
if [ "$status" = "valid" ]; then
  echo "Email is valid"
else
  echo "Email is $status or verification failed"
fi
```

## Shell Integration

Add these functions to your `.bashrc` or `.zshrc`:

```bash
# Quick email verification
vmail() {
  billionverify verify "$1"
}

# Verify and copy result
vmail-copy() {
  billionverify verify "$1" --json | pbcopy
  echo "Result copied to clipboard"
}

# Check if email is valid (returns 0 or 1)
is-valid-email() {
  [ "$(billionverify verify "$1" --json | jq -r '.status')" = "valid" ]
}
```

## Piping and Automation

```bash
# Pipe to other commands
billionverify verify test@example.com --json | jq '.score'

# Extract emails from a file and verify them in one batch (up to 50)
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' document.txt | \
  head -50 | xargs billionverify verify --json
```

## Next Steps

  - [Claude Code](/docs/ai/cli/claude-code/): Learn more about Claude Code integration
  - [MCP Integration](/docs/ai/mcp/): Use with Claude Desktop
  - [API Reference](/docs/api/): Full API documentation