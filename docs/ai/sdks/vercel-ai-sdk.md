# Vercel AI SDK

Canonical page: https://billionverify.github.io/docs/ai/sdks/vercel-ai-sdk/

The Vercel AI SDK makes it easy to add tools that AI models can call. This guide shows how to integrate BillionVerify email verification as a tool. The examples use AI SDK 7 (`ai@7`); earlier major versions use different names for some of the options shown here.

## Installation

```bash
npm install ai @ai-sdk/react @ai-sdk/anthropic zod
```

## Basic Tool Definition

```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const verifyEmailTool = tool({
  description: 'Verify if an email address is valid and deliverable. Returns the status, whether it is deliverable, and whether it is disposable or role-based.',
  inputSchema: z.object({
    email: z.string().email().describe('The email address to verify'),
  }),
  execute: async ({ email }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY!,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });

    if (!response.ok) {
      throw new Error(`Verification failed: ${response.statusText}`);
    }

    const { data } = await response.json(); // the API wraps results in { success, code, message, data }
    return data;
  },
});
```

### Using with streamText

For streaming responses:

```typescript
import { streamText, isStepCount } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { verifyEmailTool } from './tools/verify-email';

const result = streamText({
  model: anthropic('claude-sonnet-5-5'),
  tools: {
    verifyEmail: verifyEmailTool,
  },
  stopWhen: isStepCount(3), // let the model read the tool result and answer
  prompt: 'Verify if user@example.com is a valid email address',
});

for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

### Using with generateText

For non-streaming responses with tool results:

```typescript
import { generateText, isStepCount } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { verifyEmailTool } from './tools/verify-email';

const { text, toolCalls, toolResults } = await generateText({
  model: anthropic('claude-sonnet-5-5'),
  tools: {
    verifyEmail: verifyEmailTool,
  },
  stopWhen: isStepCount(3),
  prompt: 'Check these emails and tell me which are valid: john@google.com, test@mailinator.com',
});

console.log(text);
console.log('Tool calls:', toolCalls);
console.log('Tool results:', toolResults);
```

## Next.js API Route

Create an API route for AI chat with email verification:

```typescript
// app/api/chat/route.ts
import {
  streamText,
  isStepCount,
  convertToModelMessages,
  createUIMessageStreamResponse,
  toUIMessageStream,
  type UIMessage,
} from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { verifyEmailTool } from '@/lib/tools/verify-email';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-5-5'),
    messages: await convertToModelMessages(messages),
    stopWhen: isStepCount(5),
    tools: {
      verifyEmail: verifyEmailTool,
    },
  });

  return createUIMessageStreamResponse({
    stream: toUIMessageStream({ stream: result.stream }),
  });
}
```

### React client component

Use with the `useChat` hook. By default it posts to `/api/chat`:

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage } = useChat();

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          <strong>{m.role}:</strong>{' '}
          {m.parts.map((part, i) =>
            part.type === 'text' ? <span key={i}>{part.text}</span> : null
          )}
        </div>
      ))}

      <form
        onSubmit={(e) => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput('');
        }}
      >
        <input
          value={input}
          onChange={(e) => setInput(e.currentTarget.value)}
          placeholder="Ask me to verify an email..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

## Bulk Verification Tool

For verifying multiple emails at once:

```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const verifyEmailsBulkTool = tool({
  description: 'Verify up to 50 email addresses at once. Use this when checking a list of emails.',
  inputSchema: z.object({
    emails: z.array(z.string().email()).max(50).describe('Array of email addresses to verify (max 50)'),
  }),
  execute: async ({ emails }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/bulk', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY!,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ emails }),
    });

    const { data } = await response.json(); // data.results has one entry per email
    return data;
  },
});
```

## Complete Example

Here's a full example with multiple tools:

```typescript
// lib/tools/index.ts
import { tool } from 'ai';
import { z } from 'zod';

export const verifyEmailTool = tool({
  description: `Verify if an email address is valid and deliverable.

Returns:
- status: "valid", "invalid", "unknown", "risky", "disposable", "catchall" or "role"
- is_deliverable: whether the email can receive messages
- is_disposable: whether it's a temporary email service
- is_role: whether it's a role-based address like info@ or support@
- score: a number from 0 to 1

Use this tool when:
- User asks to verify an email
- Checking if an email is real
- Validating contact information`,
  inputSchema: z.object({
    email: z.string().email().describe('The email address to verify'),
  }),
  execute: async ({ email }) => {
    try {
      const response = await fetch('https://api.billionverify.com/v1/verify/single', {
        method: 'POST',
        headers: {
          'BV-API-KEY': process.env.BV_API_KEY!,
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
  },
});

export const checkCreditsTool = tool({
  description: 'Check remaining email verification credits',
  inputSchema: z.object({}),
  execute: async () => {
    const response = await fetch('https://api.billionverify.com/v1/credits', {
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY!,
      },
    });
    const { data } = await response.json();
    return data; // credits_balance, credits_consumed, credits_added, ...
  },
});
```

```typescript
// app/api/chat/route.ts
import {
  streamText,
  isStepCount,
  convertToModelMessages,
  createUIMessageStreamResponse,
  toUIMessageStream,
  type UIMessage,
} from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { verifyEmailTool, checkCreditsTool } from '@/lib/tools';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: anthropic('claude-sonnet-5-5'),
    system: 'You are a helpful assistant that can verify email addresses. When asked to verify emails, use the verifyEmail tool.',
    messages: await convertToModelMessages(messages),
    stopWhen: isStepCount(5),
    tools: {
      verifyEmail: verifyEmailTool,
      checkCredits: checkCreditsTool,
    },
  });

  return createUIMessageStreamResponse({
    stream: toUIMessageStream({ stream: result.stream }),
  });
}
```

## Next Steps

  - [LangChain Integration](/docs/ai/sdks/langchain/): Use with LangChain agents
  - [OpenAI Functions](/docs/ai/sdks/openai/): Direct OpenAI integration
  - [API Reference](/docs/api/): Full API documentation