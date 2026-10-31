# OpenAI

Canonical page: https://billionverify.github.io/docs/ai/sdks/openai/

Integrate BillionVerify directly with OpenAI's function calling API for GPT models.

## Installation

```bash
npm install openai
```

## Basic Setup

```typescript
import OpenAI from 'openai';

const openai = new OpenAI();

const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
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
    },
  },
];
```

## Implementation

```typescript
import OpenAI from 'openai';

const openai = new OpenAI();

const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
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
  const messages: OpenAI.ChatCompletionMessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo',
    messages,
    tools,
  });

  const message = response.choices[0].message;

  // Handle tool calls
  if (message.tool_calls) {
    messages.push(message);

    for (const toolCall of message.tool_calls) {
      if (toolCall.function.name === 'verify_email') {
        const args = JSON.parse(toolCall.function.arguments);
        const result = await verifyEmail(args.email);

        messages.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result),
        });
      }
    }

    // Get final response
    const finalResponse = await openai.chat.completions.create({
      model: 'gpt-4-turbo',
      messages,
    });

    return finalResponse.choices[0].message.content;
  }

  return message.content;
}

// Usage
const result = await chat('Is john@example.com a valid email?');
console.log(result);
```

### Multiple tools

Add multiple email verification related tools:

```typescript
const tools: OpenAI.ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'verify_email',
      description: 'Verify a single email address',
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
    },
  },
  {
    type: 'function',
    function: {
      name: 'verify_emails_bulk',
      description: 'Verify up to 50 email addresses at once',
      parameters: {
        type: 'object',
        properties: {
          emails: {
            type: 'array',
            items: { type: 'string' },
            description: 'Array of email addresses to verify',
          },
        },
        required: ['emails'],
      },
    },
  },
  {
    type: 'function',
    function: {
      name: 'check_credits',
      description: 'Check remaining verification credits',
      parameters: {
        type: 'object',
        properties: {},
      },
    },
  },
];
```

### With streaming

```typescript
async function chatStream(userMessage: string) {
  const messages: OpenAI.ChatCompletionMessageParam[] = [
    { role: 'user', content: userMessage },
  ];

  const stream = await openai.chat.completions.create({
    model: 'gpt-4-turbo',
    messages,
    tools,
    stream: true,
  });

  let toolCalls: any[] = [];

  for await (const chunk of stream) {
    const delta = chunk.choices[0].delta;

    if (delta.tool_calls) {
      // Accumulate tool calls
      for (const tc of delta.tool_calls) {
        if (!toolCalls[tc.index]) {
          toolCalls[tc.index] = { id: '', function: { name: '', arguments: '' } };
        }
        if (tc.id) toolCalls[tc.index].id = tc.id;
        if (tc.function?.name) toolCalls[tc.index].function.name = tc.function.name;
        if (tc.function?.arguments) toolCalls[tc.index].function.arguments += tc.function.arguments;
      }
    }

    if (delta.content) {
      process.stdout.write(delta.content);
    }
  }

  // Handle tool calls if any
  if (toolCalls.length > 0) {
    // Process tool calls and continue conversation
    // ...
  }
}
```

## Python Example

```python
from openai import OpenAI
import requests
import json
import os

client = OpenAI()

tools = [
    {
        "type": "function",
        "function": {
            "name": "verify_email",
            "description": "Verify if an email address is valid and deliverable",
            "parameters": {
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

    response = client.chat.completions.create(
        model="gpt-4-turbo",
        messages=messages,
        tools=tools
    )

    message = response.choices[0].message

    if message.tool_calls:
        messages.append(message)

        for tool_call in message.tool_calls:
            if tool_call.function.name == "verify_email":
                args = json.loads(tool_call.function.arguments)
                result = verify_email(args["email"])

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })

        final_response = client.chat.completions.create(
            model="gpt-4-turbo",
            messages=messages
        )
        return final_response.choices[0].message.content

    return message.content

# Usage
result = chat("Is john@google.com a valid email?")
print(result)
```

## Next Steps

  - [Anthropic Tools](/docs/ai/sdks/anthropic/): Claude tool use integration
  - [Vercel AI SDK](/docs/ai/sdks/vercel-ai-sdk/): Simplified tool calling
  - [API Reference](/docs/api/): Full API documentation