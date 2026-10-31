# AI Guides

Canonical page: https://billionverify.github.io/docs/ai/

BillionVerify provides multiple ways to integrate email verification capabilities into AI assistants, agents, and automated workflows. Let your AI verify emails naturally through conversation.

## Why AI Needs Email Verification

### Data Quality in AI Workflows

AI agents processing contact data, CRM records, or user lists need to ensure data quality before taking actions.

### Real-time Form Validation

AI-powered form assistants can validate emails instantly during data entry.

### Automated Workflows

In automation platforms like n8n, Zapier, or Make, AI can act as a decision node calling email verification.

### Developer Assistance

Developers using AI coding assistants (Cursor, Claude Code) can verify test emails directly in their workflow.

## Integration Methods

  - [MCP Server](/docs/ai/mcp/): For Claude Desktop, Claude Code, Cursor, and other MCP clients
  - [AI SDKs](/docs/ai/sdks/): For Vercel AI SDK, LangChain, OpenAI, and Anthropic
  - [Agent Skills](/docs/ai/agent-skills/): For CrewAI, AutoGPT, and custom AI agents
  - [CLI Tools](/docs/ai/cli/): For Claude Code, Gemini CLI, Codex, and terminal automation

## Comparison

| Integration  | Best For                  | Setup Complexity | Use Case                      |
| ------------ | ------------------------- | ---------------- | ----------------------------- |
| MCP Server   | Claude Desktop/Code users | Low              | Natural language verification |
| AI SDKs      | Custom AI applications    | Medium           | Building AI-powered apps      |
| Agent Skills | AI agent frameworks       | Medium           | Automated workflows           |
| CLI Tools    | Terminal users, scripts   | Low              | Dev workflows, automation     |

## Quick Start

### Option 1: Remote MCP for ChatGPT and Claude Users (Recommended)

If your client supports remote MCP over HTTP, use the hosted BillionVerify endpoint:

```text
https://mcp.billionverify.com/mcp
```

For Claude Code, run:

```bash
claude mcp add --transport http billionverify https://mcp.billionverify.com/mcp
```

Then run `/mcp` inside Claude Code and complete OAuth in your browser.

The hosted server uses OAuth and does not accept API keys. If your client only supports local stdio servers with API keys, run the self-hosted `billionverify-mcp` npm package (it reads `BILLIONVERIFY_API_KEY`), or use the BillionVerify CLI or REST API. See [MCP Server](/docs/ai/mcp/).

### Option 2: AI SDK Tool for Developers

Building a custom AI application? Add email verification as a tool:

```typescript
import { tool } from 'ai';
import { z } from 'zod';

const verifyEmailTool = tool({
  description: 'Verify if an email address is valid and deliverable',
  inputSchema: z.object({
    email: z.string().email(),
  }),
  execute: async ({ email }) => {
    const response = await fetch(
      'https://api.billionverify.com/v1/verify/single',
      {
        method: 'POST',
        headers: {
          'BV-API-KEY': process.env.BV_API_KEY!,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ email }),
      },
    );
    const { data } = await response.json(); // the API wraps results in { success, code, message, data }
    return data;
  },
});
```

### Option 3: CLI for Terminal Users

```bash
# Install
npm install -g billionverify

# Configure
billionverify config set api_key YOUR_API_KEY

# Verify
billionverify verify john@example.com

# JSON output for AI parsing
billionverify verify john@example.com --json
```

## Supported AI Platforms

### MCP-Compatible Clients

| Client            | Connection                                   |
| ----------------- | -------------------------------------------- |
| ChatGPT (web)     | Hosted server, OAuth, Developer mode         |
| Claude / Desktop  | Hosted server as a custom connector, OAuth   |
| Claude Code       | Hosted server, OAuth                         |
| Cursor            | Hosted server by URL, OAuth                  |

Other clients that support remote MCP over Streamable HTTP with OAuth can use the same endpoint. Clients that only run local stdio servers can use the self-hosted `billionverify-mcp` package with an API key.

### AI SDKs & Frameworks

| Framework        | Integration      |
| ---------------- | ---------------- |
| Vercel AI SDK    | Tool calling     |
| LangChain        | Custom tool      |
| OpenAI Functions | Function calling |
| Anthropic Tools  | Tool use         |
| CrewAI           | Tool definition  |
| AutoGPT          | Plugin           |

### CLI Tools

| Tool        | Description                  |
| ----------- | ---------------------------- |
| Claude Code | Anthropic's CLI assistant    |
| Gemini CLI  | Google's terminal AI         |
| Codex CLI   | OpenAI code generation       |
| OpenCode    | Open-source AI assistant     |
| Droid       | Developer terminal assistant |

## Value Scenarios

| Scenario            | Description                                  | Value                  |
| ------------------- | -------------------------------------------- | ---------------------- |
| Sales Lead Cleaning | AI validates CRM prospect emails             | Improve deliverability |
| Registration Forms  | Real-time block of invalid/disposable emails | Reduce fake signups    |
| Data Migration      | Bulk verify imported historical data         | Ensure data quality    |
| Marketing Campaigns | Verify email lists before sending            | Lower bounce rate      |
| Development Testing | AI assistant verifies test emails            | Improve dev efficiency |

## API Quick Reference

For AI integration, use these API endpoints:

### Single Verification

```bash
curl -X POST https://api.billionverify.com/v1/verify/single \
  -H "BV-API-KEY: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com"}'
```

**Response** (abridged; the result is in `data`):

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "email": "test@example.com",
    "status": "valid",
    "score": 0.95,
    "is_deliverable": true,
    "is_disposable": false,
    "is_catchall": false,
    "is_role": false,
    "is_free": false,
    "domain": "example.com",
    "credits_used": 1
  }
}
```

### Key Response Fields

These fields are inside `data`.

| Field            | Description                                                   | AI Interpretation      |
| ---------------- | ------------------------------------------------------------- | ---------------------- |
| `status`         | valid, invalid, unknown, risky, disposable, catchall, role    | Primary decision field |
| `is_deliverable` | Can receive email                                             | Safe to send?          |
| `is_disposable`  | Temporary email                                               | Block registration?    |
| `is_catchall`    | The domain accepts mail for any address                       | Cannot confirm the mailbox |
| `is_role`        | info@, support@, etc.                                         | Shared mailbox warning |
| `score`          | A number from 0 to 1                                          | Risk assessment        |

## Next Steps

  - [MCP Integration](/docs/ai/mcp/): Set up MCP for Claude Desktop and Code
  - [AI SDKs](/docs/ai/sdks/): Integrate with Vercel AI SDK, LangChain, and more
  - [Agent Skills](/docs/ai/agent-skills/): Build AI agent capabilities
  - [CLI Tools](/docs/ai/cli/): Terminal-based verification