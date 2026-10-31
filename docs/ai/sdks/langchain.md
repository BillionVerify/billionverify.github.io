# LangChain

Canonical page: https://billionverify.github.io/docs/ai/sdks/langchain/

Integrate email verification as a custom tool in LangChain for both Python and JavaScript/TypeScript.

## Python

### Installation

```bash
pip install langchain langchain-openai requests
```

### Basic Tool Definition

```python
from langchain.tools import tool
import requests
import os

@tool
def verify_email(email: str) -> dict:
    """Verify if an email address is valid and deliverable.

    Args:
        email: The email address to verify

    Returns:
        Verification result including status, score and is_deliverable
    """
    response = requests.post(
        'https://api.billionverify.com/v1/verify/single',
        headers={
            'BV-API-KEY': os.environ["BV_API_KEY"],
            'Content-Type': 'application/json',
        },
        json={'email': email}
    )
    return response.json()['data']
```

### Using with an Agent

```python
from langchain.tools import tool
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_functions_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
import requests
import os

@tool
def verify_email(email: str) -> dict:
    """Verify if an email address is valid and deliverable.

    Args:
        email: The email address to verify

    Returns:
        Verification result including status, score and is_deliverable
    """
    response = requests.post(
        'https://api.billionverify.com/v1/verify/single',
        headers={
            'BV-API-KEY': os.environ["BV_API_KEY"],
            'Content-Type': 'application/json',
        },
        json={'email': email}
    )
    return response.json()['data']

# Create agent
llm = ChatOpenAI(model="gpt-4-turbo")
tools = [verify_email]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant that can verify email addresses."),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_openai_functions_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools)

# Use the agent
result = agent_executor.invoke({
    "input": "Is john@google.com a valid email?"
})
print(result["output"])
```

### Bulk Verification Tool

```python
from typing import List

@tool
def verify_emails_bulk(emails: List[str]) -> dict:
    """Verify multiple email addresses at once.

    Args:
        emails: List of email addresses to verify (max 50)

    Returns:
        Bulk verification results
    """
    response = requests.post(
        'https://api.billionverify.com/v1/verify/bulk',
        headers={
            'BV-API-KEY': os.environ["BV_API_KEY"],
            'Content-Type': 'application/json',
        },
        json={'emails': emails[:50]}
    )
    return response.json()['data']
```

### With Error Handling

```python
@tool
def verify_email(email: str) -> dict:
    """Verify if an email address is valid and deliverable.

    Args:
        email: The email address to verify

    Returns:
        Verification result or error message
    """
    try:
        response = requests.post(
            'https://api.billionverify.com/v1/verify/single',
            headers={
                'BV-API-KEY': os.environ["BV_API_KEY"],
                'Content-Type': 'application/json',
            },
            json={'email': email},
            timeout=30
        )

        if response.status_code == 429:
            return {'error': 'Rate limit exceeded. Please try again later.'}

        response.raise_for_status()
        return response.json()['data']

    except requests.exceptions.Timeout:
        return {'error': 'Request timed out. Please try again.'}
    except requests.exceptions.RequestException as e:
        return {'error': f'Request failed: {str(e)}'}
```

## JavaScript/TypeScript

### Installation

```bash
npm install @langchain/core @langchain/openai langchain zod
```

### Basic Tool Definition

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { z } from 'zod';

const verifyEmailTool = new DynamicStructuredTool({
  name: 'verify_email',
  description: 'Verify if an email address is valid and deliverable',
  schema: z.object({
    email: z.string().email().describe('The email address to verify'),
  }),
  func: async ({ email }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });
    return JSON.stringify((await response.json()).data);
  },
});
```

### Using with an Agent

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { ChatOpenAI } from '@langchain/openai';
import { AgentExecutor, createOpenAIFunctionsAgent } from 'langchain/agents';
import { ChatPromptTemplate, MessagesPlaceholder } from '@langchain/core/prompts';
import { z } from 'zod';

const verifyEmailTool = new DynamicStructuredTool({
  name: 'verify_email',
  description: 'Verify if an email address is valid and deliverable',
  schema: z.object({
    email: z.string().email().describe('The email address to verify'),
  }),
  func: async ({ email }) => {
    const response = await fetch('https://api.billionverify.com/v1/verify/single', {
      method: 'POST',
      headers: {
        'BV-API-KEY': process.env.BV_API_KEY,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    });
    return JSON.stringify((await response.json()).data);
  },
});

const llm = new ChatOpenAI({ modelName: 'gpt-4-turbo' });
const tools = [verifyEmailTool];

const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'You are a helpful assistant that can verify email addresses.'],
  new MessagesPlaceholder('chat_history'),
  ['human', '{input}'],
  new MessagesPlaceholder('agent_scratchpad'),
]);

const agent = await createOpenAIFunctionsAgent({ llm, tools, prompt });
const agentExecutor = new AgentExecutor({ agent, tools });

const result = await agentExecutor.invoke({
  input: 'Check if test@example.com is valid',
  chat_history: [],
});

console.log(result.output);
```

### With Streaming

```typescript
import { AgentExecutor, createOpenAIFunctionsAgent } from 'langchain/agents';

// ... setup code ...

const stream = await agentExecutor.stream({
  input: 'Verify john@google.com for me',
  chat_history: [],
});

for await (const chunk of stream) {
  if (chunk.output) {
    console.log(chunk.output);
  }
}
```

## Next Steps

  - [OpenAI Functions](/docs/ai/sdks/openai/): Direct OpenAI integration
  - [Anthropic Tools](/docs/ai/sdks/anthropic/): Claude tool use
  - [API Reference](/docs/api/): Full API documentation