# BillionVerify Docs

Canonical page: https://billionverify.github.io/docs/

BillionVerify verifies email addresses through a REST API. These docs cover the API, the official SDKs and the AI integrations.

To see what a result looks like before you write any code, you can [test a single address by hand](https://billionverify.com/email-checker) on the BillionVerify site. It needs no sign-up.

## Getting Started

  - [Quickstart Guide](/docs/quickstart/): Verify your first email
  - [API Reference](/docs/api/): Endpoints, parameters and responses

## Key Features

- **Single verification** - One email per request with `POST /verify/single`
- **Bulk and file verification** - Up to 50 emails per synchronous request, or upload a CSV, TXT or XLSX file for an asynchronous job
- **Detection** - Disposable, role-based and catch-all addresses are reported as separate statuses
- **Unknown results are free** - You are not charged when a check cannot reach a definite answer
- **Disposable check** - `/verify/disposable` is a free domain lookup

The free single-address page, [the BillionVerify verifier](https://billionverify.com/email-verifier), shows the status and the disposable, role and catch-all flags for one address.

## Quick Example

```javascript
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({ apiKey: process.env.BV_API_KEY });

const result = await client.verify('user@example.com');

console.log(result.status, result.is_deliverable);
// valid true
```

## Need Help?

- Contact [Support](mailto:support@billionverify.com)

## All pages

- [Quickstart](https://billionverify.github.io/docs/quickstart/)
- [API Overview](https://billionverify.github.io/docs/api/)
- [Verify a single email](https://billionverify.github.io/docs/api/verify-single/)
- [Verify in bulk](https://billionverify.github.io/docs/api/verify-bulk/)
- [Disposable email check](https://billionverify.github.io/docs/api/disposable-check/)
- [Upload a file](https://billionverify.github.io/docs/api/upload-file/)
- [File job status](https://billionverify.github.io/docs/api/file-status/)
- [File results](https://billionverify.github.io/docs/api/file-results/)
- [Credits](https://billionverify.github.io/docs/api/credits/)
- [Webhooks](https://billionverify.github.io/docs/api/webhooks/)
- [Status and Error Codes](https://billionverify.github.io/docs/api/errors/)
- [Verification Reasons](https://billionverify.github.io/docs/verification-reasons/)
- [Vercel Connect Connector](https://billionverify.github.io/docs/connectors/vercel/)
- [Claude Connector](https://billionverify.github.io/docs/connectors/claude/)
- [SDKs](https://billionverify.github.io/docs/sdks/)
- [Node.js SDK](https://billionverify.github.io/docs/sdks/node/)
- [TypeScript SDK](https://billionverify.github.io/docs/sdks/typescript/)
- [Python SDK](https://billionverify.github.io/docs/sdks/python/)
- [Go SDK](https://billionverify.github.io/docs/sdks/go/)
- [PHP SDK](https://billionverify.github.io/docs/sdks/php/)
- [Java SDK](https://billionverify.github.io/docs/sdks/java/)
- [AI Guides](https://billionverify.github.io/docs/ai/)
- [MCP Server](https://billionverify.github.io/docs/ai/mcp/)
- [Agent Skills](https://billionverify.github.io/docs/ai/agent-skills/)
- [AI SDKs](https://billionverify.github.io/docs/ai/sdks/)
- [Anthropic Claude](https://billionverify.github.io/docs/ai/sdks/anthropic/)
- [OpenAI](https://billionverify.github.io/docs/ai/sdks/openai/)
- [LangChain](https://billionverify.github.io/docs/ai/sdks/langchain/)
- [Vercel AI SDK](https://billionverify.github.io/docs/ai/sdks/vercel-ai-sdk/)
- [CLI Tools](https://billionverify.github.io/docs/ai/cli/)
- [Claude Code](https://billionverify.github.io/docs/ai/cli/claude-code/)
- [Codex](https://billionverify.github.io/docs/ai/cli/codex/)
- [Gemini CLI](https://billionverify.github.io/docs/ai/cli/gemini-cli/)
- [Droid](https://billionverify.github.io/docs/ai/cli/droid/)
- [OpenCode](https://billionverify.github.io/docs/ai/cli/opencode/)
