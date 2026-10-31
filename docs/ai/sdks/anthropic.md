# Anthropic Claude

Canonical page: https://billionverify.github.io/docs/ai/sdks/anthropic/

Use BillionVerify directly with Anthropic's Claude models through their native tool use feature.

## Installation

```bash
npm install @anthropic-ai/sdk
```

## Basic Setup

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: 'verify_email',
    description: 'Verify if an email address is valid and deliverable. Returns status, is_deliverable, is_disposable, is_role and score.',
    input_schema: {
      type: 'object' as const,
      properties: {
        email: {
          type: 'string',
          description: 'The email address to verify',
        },
      },
      required: ['email'],
    },
  },
];
```

## Implementation

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: 'verify_email',
    description: 'Verify if an email address is valid and deliverable. Returns status, is_deliverable, is_disposable, is_role and score.',
    input_schema: {
      type: 'object' as const,
      properties: {
        email: {
          type: 'string',
          description: 'The email address to verify',
        },
      },
      required: ['email'],
    },
  },
];

async function verifyEmail(email: string) {
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
}

async function chat(userMessage: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-5-5',
    max_tokens: 1024,
    tools,
    messages,
  });

  // Handle tool use
  if (response.stop_reason === 'tool_use') {
    const toolUseBlock = response.content.find(
      (block): block is Anthropic.ToolUseBlock => block.type === 'tool_use'
    );

    if (toolUseBlock && toolUseBlock.name === 'verify_email') {
      const result = await verifyEmail((toolUseBlock.input as { email: string }).email);

      messages.push({ role: 'assistant', content: response.content });
      messages.push({
        role: 'user',
        content: [
          {
            type: 'tool_result',
            tool_use_id: toolUseBlock.id,
            content: JSON.stringify(result),
          },
        ],
      });

      const finalResponse = await anthropic.messages.create({
        model: 'claude-sonnet-5-5',
        max_tokens: 1024,
        tools,
        messages,
      });

      const textBlock = finalResponse.content.find(
        (block): block is Anthropic.TextBlock => block.type === 'text'
      );
      return textBlock?.text;
    }
  }

  const textBlock = response.content.find(
    (block): block is Anthropic.TextBlock => block.type === 'text'
  );
  return textBlock?.text;
}

// Usage
const result = await chat('Verify test@example.com for me');
console.log(result);
```

### Multiple tools

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: 'verify_email',
    description: 'Verify a single email address',
    input_schema: {
      type: 'object' as const,
      properties: {
        email: {
          type: 'string',
          description: 'The email address to verify',
        },
      },
      required: ['email'],
    },
  },
  {
    name: 'verify_emails_bulk',
    description: 'Verify up to 50 email addresses at once',
    input_schema: {
      type: 'object' as const,
      properties: {
        emails: {
          type: 'array',
          items: { type: 'string' },
          description: 'Array of email addresses to verify (max 50)',
        },
      },
      required: ['emails'],
    },
  },
  {
    name: 'check_credits',
    description: 'Check remaining verification credits',
    input_schema: {
      type: 'object' as const,
      properties: {},
    },
  },
];

const headers = {
  'BV-API-KEY': process.env.BV_API_KEY!,
  'Content-Type': 'application/json',
};

async function verifyEmailsBulk(emails: string[]) {
  const response = await fetch('https://api.billionverify.com/v1/verify/bulk', {
    method: 'POST',
    headers,
    body: JSON.stringify({ emails }),
  });
  const { data } = await response.json();
  return data;
}

async function checkCredits() {
  const response = await fetch('https://api.billionverify.com/v1/credits', { headers });
  const { data } = await response.json();
  return data; // credits_balance, credits_consumed, credits_added, ...
}
```

### Handling multiple tool calls

Claude may call tools multiple times in a conversation:

```typescript
async function chatWithMultipleTools(userMessage: string) {
  let messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  while (true) {
    const response = await anthropic.messages.create({
      model: 'claude-sonnet-5-5',
      max_tokens: 1024,
      tools,
      messages,
    });

    // Check if Claude wants to use a tool
    if (response.stop_reason === 'tool_use') {
      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const block of response.content) {
        if (block.type === 'tool_use') {
          let result: any;

          switch (block.name) {
            case 'verify_email':
              result = await verifyEmail((block.input as { email: string }).email);
              break;
            case 'verify_emails_bulk':
              result = await verifyEmailsBulk((block.input as { emails: string[] }).emails);
              break;
            case 'check_credits':
              result = await checkCredits();
              break;
          }

          toolResults.push({
            type: 'tool_result',
            tool_use_id: block.id,
            content: JSON.stringify(result),
          });
        }
      }

      messages.push({ role: 'assistant', content: response.content });
      messages.push({ role: 'user', content: toolResults });
    } else {
      // No more tool calls, return the final response
      const textBlock = response.content.find(
        (block): block is Anthropic.TextBlock => block.type === 'text'
      );
      return textBlock?.text;
    }
  }
}
```

### With streaming

```typescript
async function chatStream(userMessage: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  const stream = anthropic.messages.stream({
    model: 'claude-sonnet-5-5',
    max_tokens: 1024,
    tools,
    messages,
  });

  for await (const event of stream) {
    if (event.type === 'content_block_delta') {
      if (event.delta.type === 'text_delta') {
        process.stdout.write(event.delta.text);
      }
    }
  }

  const finalMessage = await stream.finalMessage();

  // Handle tool use if needed
  if (finalMessage.stop_reason === 'tool_use') {
    // Process tool calls...
  }
}
```

## Python Example

```python
import anthropic
import requests
import json
import os

client = anthropic.Anthropic()

tools = [
    {
        "name": "verify_email",
        "description": "Verify if an email address is valid and deliverable",
        "input_schema": {
            "type": "object",
            "properties": {
                "email": {
                    "type": "string",
                    "description": "The email address to verify"
                }
            },
            "required": ["email"]
        }
    }
]

def verify_email(email: str) -> dict:
    response = requests.post(
        'https://api.billionverify.com/v1/verify/single',
        headers={
            'BV-API-KEY': os.environ["BV_API_KEY"],
            'Content-Type': 'application/json',
        },
        json={'email': email}
    )
    return response.json()['data']

def chat(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    response = client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    if response.stop_reason == "tool_use":
        tool_use = next(
            block for block in response.content
            if block.type == "tool_use"
        )

        result = verify_email(tool_use.input["email"])

        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": [{
                "type": "tool_result",
                "tool_use_id": tool_use.id,
                "content": json.dumps(result)
            }]
        })

        final_response = client.messages.create(
            model="claude-sonnet-5-5",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        return next(
            block.text for block in final_response.content
            if hasattr(block, "text")
        )

    return next(
        block.text for block in response.content
        if hasattr(block, "text")
    )

# Usage
result = chat("Is john@google.com a valid email?")
print(result)
```

## Next Steps

  - [MCP Integration](/docs/ai/mcp/): Use with Claude Desktop and Code
  - [OpenAI Functions](/docs/ai/sdks/openai/): OpenAI function calling
  - [API Reference](/docs/api/): Full API documentation