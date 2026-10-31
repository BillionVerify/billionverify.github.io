# Agent Skills

Canonical page: https://billionverify.github.io/docs/ai/agent-skills/

Agent Skills are packaged instructions that tell an AI agent how to do a task. BillionVerify provides an email verification skill, and the sections below show how to add the same verification call as a tool in common agent frameworks.

## Why SMTP Verification Matters for AI Agents

Syntax and DNS checks alone cannot tell whether a mailbox exists. The BillionVerify API can also ask the recipient's mail server about the mailbox over SMTP (the `check_smtp` option):

1. It connects to the recipient's mail server
2. It asks whether the specific mailbox would accept mail
3. It reports the result in `status`, and reports domains that accept every address as `catchall` instead of `valid`

Running this yourself needs outbound port 25, which many cloud providers restrict. With the API, the checks run on BillionVerify's servers and your agent only makes an HTTPS call.

## What are Agent Skills?

Agent Skills are modular, reusable components that give AI agents specific capabilities:

- **Plug-and-play**: The skill tells the agent which API calls to make, so you do not write verification logic
- **Standardized format**: A `SKILL.md` file that skill-aware agents (Claude Code, Codex CLI, Cursor and others) can load
- **Mailbox check**: Single, batch (up to 50 emails) and file verification, with optional SMTP checking
- **Account tools**: Credit balance, verification history, statistics and webhooks

## Get the BillionVerify Skill

Install the BillionVerify email verification skill for your AI coding assistant:

```bash
npx skills add BillionVerify/billionverify-skill

# Or install for specific agents
npx skills add BillionVerify/billionverify-skill -a cursor -a claude-code
```

The skill calls the REST API with `curl`, so the agent needs shell access and the `BILLIONVERIFY_API_KEY` environment variable:

```bash
export BILLIONVERIFY_API_KEY=your_api_key_here
```

- **ClawHub**: [clawhub.ai/billionverifier/billionverify-skill](https://clawhub.ai/billionverifier/billionverify-skill)
- **GitHub**: [github.com/BillionVerify/billionverify-skill](https://github.com/BillionVerify/billionverify-skill)

On ChatGPT web and claude.ai, the hosted [MCP server](/docs/ai/mcp/) is simpler: it signs in with OAuth and needs no API key.

## Supported Agent Frameworks

These are code examples that call the REST API, not packaged plugins.

| Framework | Skill Type | Where |
|-----------|-----------|--------|
| Claude (Anthropic SDK) | Tool | Example below |
| AutoGPT | Plugin | Example below |
| LangChain Agents | Tool | [LangChain](/docs/ai/sdks/langchain/) |
| CrewAI | Tool | Example below |

## Claude Agent SDK

The Anthropic SDK supports tool use in the Messages API. Add email verification as a tool and run the tool loop yourself:

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: "verify_email",
    description: `Verify if an email address is valid and deliverable.

Returns:
- status: "valid", "invalid", "unknown", "risky", "disposable", "catchall" or "role"
- is_deliverable: whether the email can receive messages
- is_disposable: whether it's a temporary email service
- is_role: whether it's a role-based address (info@, support@)
- score: a number from 0 to 1`,
    input_schema: {
      type: "object",
      properties: {
        email: {
          type: "string",
          description: "The email address to verify",
        },
      },
      required: ["email"],
    },
  },
];

async function verifyEmail(email: string) {
  const response = await fetch("https://api.billionverify.com/v1/verify/single", {
    method: "POST",
    headers: {
      "BV-API-KEY": process.env.BV_API_KEY,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ email }),
  });
  const { data } = await response.json(); // the API wraps results in { success, code, message, data }
  return data;
}

async function processToolCall(
  toolName: string,
  toolInput: Record<string, string>
) {
  if (toolName === "verify_email") {
    return await verifyEmail(toolInput.email);
  }
  throw new Error(`Unknown tool: ${toolName}`);
}

async function agentLoop(userMessage: string) {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  while (true) {
    const response = await client.messages.create({
      model: "claude-sonnet-5-5",
      max_tokens: 4096,
      tools,
      messages,
    });

    // Check if we need to process tool calls
    if (response.stop_reason === "tool_use") {
      const toolUseBlocks = response.content.filter(
        (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
      );

      const toolResults: Anthropic.ToolResultBlockParam[] = [];

      for (const toolUse of toolUseBlocks) {
        const result = await processToolCall(
          toolUse.name,
          toolUse.input as Record<string, string>
        );
        toolResults.push({
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: JSON.stringify(result),
        });
      }

      messages.push({ role: "assistant", content: response.content });
      messages.push({ role: "user", content: toolResults });
    } else {
      // Agent is done
      const textBlock = response.content.find(
        (block): block is Anthropic.TextBlock => block.type === "text"
      );
      return textBlock?.text;
    }
  }
}

// Usage
const result = await agentLoop(
  "I have a list of emails to verify: john@google.com, test@mailinator.com, support@microsoft.com. Check each one and tell me which are safe to use."
);
console.log(result);
```

## CrewAI Integration

Add email verification to your CrewAI agents:

```python
from crewai import Agent, Task, Crew
from crewai.tools import tool
import requests
import os

@tool("Email Verification Tool")
def verify_email(email: str) -> str:
    """Verify if an email address is valid and deliverable.

    Args:
        email: The email address to verify

    Returns:
        Verification result with status, score and deliverability
    """
    response = requests.post(
        'https://api.billionverify.com/v1/verify/single',
        headers={
            'BV-API-KEY': os.environ["BV_API_KEY"],
            'Content-Type': 'application/json',
        },
        json={'email': email}
    )
    result = response.json()['data']
    return f"Email: {email}, Status: {result['status']}, Score: {result['score']}"

# Create an agent with the verification skill
data_quality_agent = Agent(
    role='Data Quality Specialist',
    goal='Ensure all email addresses in the database are valid and deliverable',
    backstory='Expert in data validation and email deliverability',
    tools=[verify_email],
    verbose=True
)

# Create a task
verification_task = Task(
    description='Verify the following email addresses and report which ones are invalid: {emails}',
    expected_output='A report of all emails with their verification status',
    agent=data_quality_agent
)

# Run the crew
crew = Crew(
    agents=[data_quality_agent],
    tasks=[verification_task]
)

result = crew.kickoff(inputs={
    'emails': 'john@google.com, fake@invalid.xyz, test@mailinator.com'
})
print(result)
```

## AutoGPT Plugin

A sketch of an AutoGPT plugin for email verification. It follows the legacy Auto-GPT plugin template (`post_prompt`); a complete plugin also has to implement the other `can_handle_*` methods of `AutoGPTPluginTemplate`:

```python
# autogpt_plugins/billionverify/__init__.py
from typing import Any, Dict, List, Optional, Tuple, TypeVar
import requests
import os

PromptGenerator = TypeVar("PromptGenerator")

class BillionVerifyPlugin:
    def __init__(self):
        self.api_key = os.environ.get("BV_API_KEY")

    @staticmethod
    def get_name() -> str:
        return "BillionVerify Email Verification"

    def can_handle_post_prompt(self) -> bool:
        return True

    def post_prompt(self, prompt: PromptGenerator) -> PromptGenerator:
        prompt.add_command(
            "verify_email",
            "Verify if an email is valid",
            {"email": "<email_address>"},
            self.verify_email
        )
        return prompt

    def verify_email(self, email: str) -> str:
        response = requests.post(
            'https://api.billionverify.com/v1/verify/single',
            headers={
                'BV-API-KEY': self.api_key,
                'Content-Type': 'application/json',
            },
            json={'email': email}
        )
        result = response.json()['data']

        if result['status'] == 'valid':
            return f"{email} is valid (score: {result['score']})"
        elif result['status'] == 'invalid':
            return f"{email} is invalid"
        else:
            return f"{email} status: {result['status']}"
```

## Best Practices

### 1. Skill Description

Write clear descriptions so the agent knows when to use the skill:

```typescript
description: `Verify if an email address is valid and deliverable.

Use this skill when:
- User asks to check an email address
- Validating contact information before saving
- Cleaning email lists
- Checking if an email is disposable or role-based

Returns: status, is_deliverable, is_disposable, is_role, score`
```

### 2. Error Handling

Build robust error handling into your skill:

```python
@tool("Email Verification")
def verify_email(email: str) -> str:
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
            return "Rate limit exceeded. Please try again later."

        response.raise_for_status()
        return response.json()['data']

    except requests.exceptions.Timeout:
        return "Verification timed out. Please try again."
    except Exception as e:
        return f"Verification failed: {str(e)}"
```

### 3. Batch Operations

For multiple emails, use the bulk endpoint:

```typescript
const verifyBulkSkill = {
  name: "verify_emails_bulk",
  description: "Verify multiple email addresses at once. More efficient for lists.",
  input_schema: {
    type: "object",
    properties: {
      emails: {
        type: "array",
        items: { type: "string" },
        description: "Array of emails to verify (max 50)",
      },
    },
    required: ["emails"],
  },
};
```

## Next Steps

  - [CLI Tools](/docs/ai/cli/): Use verification from the command line
  - [AI SDKs](/docs/ai/sdks/): Integrate with AI frameworks
  - [MCP Integration](/docs/ai/mcp/): Use with Claude Desktop