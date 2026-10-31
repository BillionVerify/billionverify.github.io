# Claude Connector

Canonical page: https://billionverify.github.io/docs/connectors/claude/

The BillionVerify connector is listed in Claude's connector directory. Connect it once with your BillionVerify account, then verify email addresses in chat. You do not need an API key or a server URL.

- **Listing:** [claude.ai/directory/billionverify](https://claude.ai/directory/billionverify)
- **Details:** [BillionVerify Is Now in Claude's Connector Directory](https://billionverify.com/blog/billionverify-claude-connector)

## Connect BillionVerify

1. Open the [BillionVerify listing](https://claude.ai/directory/billionverify), or search for BillionVerify in Claude's connector directory.
2. Click Connect, sign in with your BillionVerify account and approve access.
3. Start a chat and ask something like "Is this address safe to send to?"

If you do not have an account yet, [sign up at billionverify.com](https://billionverify.com/auth/sign-up) first. The connector uses the same account, credits and verification history as your dashboard.

## What Claude can do

The connector gives Claude 11 tools that cover four jobs.

| Job | What it does |
| --- | --- |
| Verify addresses | Checks one address, or up to 50 in a single request |
| Review results | Shows recent verifications and totals for the last 7, 30 or 90 days or the last year |
| Follow bulk jobs | Tracks a file uploaded in the dashboard and gets a download link for the results |
| Manage webhooks | Creates, lists and deletes the webhooks that report when a job finishes |

Each result says whether the address is valid, invalid, catch-all, disposable, a role account or unknown. See [Status and error codes](/docs/api/errors/) for what each status means.

### Prompts to start with

> Is support@billionverify.com a deliverable email address?

> Check these emails and tell me which ones will bounce: anna@example.com, sales@example.org

> Summarize my email verification results for the last 30 days.

> How many verification credits do I have left?

### Verify a bigger list

Claude checks up to 50 addresses in a request. For a bigger list, upload the file in the [BillionVerify dashboard](https://billionverify.com/auth/sign-in), then ask Claude to track the job and fetch the results.

## Other ways to use BillionVerify in an agent

- [MCP server](/docs/ai/mcp/): add the server to other MCP clients
- [Agent Skills](/docs/ai/agent-skills/): install the skill in a coding agent
- [Vercel Connect](/docs/connectors/vercel/): verify addresses from apps and agents you deploy on Vercel