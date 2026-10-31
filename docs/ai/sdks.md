# AI SDKs

Canonical page: https://billionverify.github.io/docs/ai/sdks/

Build AI applications with built-in email verification capabilities using popular AI frameworks and SDKs.

## Supported Frameworks

  - [Vercel AI SDK](/docs/ai/sdks/vercel-ai-sdk/): Tool calling with Next.js and React applications
  - [LangChain](/docs/ai/sdks/langchain/): Custom tools for LangChain agents in Python and JavaScript
  - [OpenAI Function Calling](/docs/ai/sdks/openai/): Native function calling with OpenAI GPT models
  - [Anthropic Tool Use](/docs/ai/sdks/anthropic/): Tool use with Claude models

## Comparison

| Framework | Language | Best For | Complexity |
|-----------|----------|----------|------------|
| Vercel AI SDK | TypeScript | Next.js apps, streaming | Low |
| LangChain | Python/JS | Complex agents, chains | Medium |
| OpenAI Functions | Any | Direct GPT integration | Medium |
| Anthropic Tools | Any | Direct Claude integration | Medium |

## Quick Example

Here's a simple tool definition that works across frameworks:

```typescript
const verifyEmailTool = {
  name: 'verify_email',
  description: 'Verify if an email address is valid and deliverable',
  parameters: {
    type: 'object',
    properties: {
      email: {
        type: 'string',
        description: 'The email address to verify',
      },
    },
    required: ['email'],
  },
  execute: async ({ email }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });
    const { data } = await response.json(); // the API wraps results in { success, code, message, data }
    return data;
  },
};
```

## Best Practices

### 1. Error Handling

Always handle API errors gracefully in your tools:

```typescript
execute: async ({ email }) => {
  try {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });

    if (!response.ok) {
      if (response.status === 429) {
        return { error: 'Rate limit exceeded. Please try again later.' };
      }
      return { error: `Verification failed: ${response.statusText}` };
    }

    const { data } = await response.json();
    return data;
  } catch (error) {
    return { error: 'Network error. Could not reach verification service.' };
  }
}
```

### 2. Clear Tool Descriptions

Write descriptive tool definitions so the AI knows when to use them:

```typescript
description: `Verify if an email address is valid and deliverable.

Returns:
- status: "valid", "invalid", "unknown", "risky", "disposable", "catchall" or "role"
- is_deliverable: whether the email can receive messages
- is_disposable: whether it's a temporary email service
- is_catchall: whether the domain accepts mail for any address
- is_role: whether it's a role-based address like info@ or support@
- score: a number from 0 to 1

Use this tool when:
- User asks to verify an email
- Checking if an email is real
- Validating contact information`
```

### 3. Batch for Efficiency

When verifying multiple emails, use the bulk endpoint:

```typescript
const verifyEmailsBulkTool = {
  name: 'verify_emails_bulk',
  description: 'Verify multiple emails at once. More efficient than verifying one by one.',
  parameters: {
    type: 'object',
    properties: {
      emails: {
        type: 'array',
        items: { type: 'string' },
        maxItems: 50,
      },
    },
    required: ['emails'],
  },
  execute: async ({ emails }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/bulk', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ emails }),
    });
    const { data } = await response.json(); // data.results has one entry per email
    return data;
  },
};
```

### 4. Cache Results

Consider caching verification results to save credits:

```typescript
const cache = new Map<string, { result: any; timestamp: number }>();
const CACHE_TTL = 3600000; // 1 hour

async function verifyWithCache(email: string) {
  const cached = cache.get(email);
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.result;
  }

  const result = await verifyEmail(email); // your function that calls /v1/verify/single
  cache.set(email, { result, timestamp: Date.now() });
  return result;
}
```

## Next Steps

  - [Vercel AI SDK](/docs/ai/sdks/vercel-ai-sdk/): Get started with Vercel AI SDK
  - [MCP Integration](/docs/ai/mcp/): Use with Claude Desktop and Code
  - [API Reference](/docs/api/): Full API documentation