# MCP Server

Canonical page: https://billionverify.github.io/docs/ai/mcp/

Model Context Protocol (MCP) is an open standard that lets AI clients call external tools through a shared interface. BillionVerify exposes a remote MCP server for email verification, account reporting, result downloads, and webhook management over **Streamable HTTP**.

|                                 |                                                                      |
| ------------------------------- | -------------------------------------------------------------------- |
| **MCP Server Version**          | `2.1.0`                                                              |
| **Endpoint**                    | `https://mcp.billionverify.com/mcp`                                  |
| **Auth Model**                  | OAuth 2.1 + Bearer access token                                      |
| **Protected Resource Metadata** | `https://mcp.billionverify.com/.well-known/oauth-protected-resource` |
| **Available Tools**             | 11                                                                   |
| **Available Resources**         | 3                                                                    |

> **Important**
>
> The BillionVerify MCP server does **not** support `?api_key=...` query parameters, resource URIs with embedded API keys, or tool-level `api_key` arguments.
>
> If you previously used `curl --stdio "https://mcp.billionverify.com/mcp?api_key=..."`, that flow is no longer supported. Use an OAuth-capable remote MCP client instead. If your client only supports local stdio and API keys, run the self-hosted `billionverify-mcp` npm package (see [When To Use REST API Instead](#when-to-use-rest-api-instead)) or call the REST API directly.

## What You Can Do

- Verify a single email address
- Verify up to 50 email addresses in one batch call
- Check account credit balance
- Track asynchronous verification job status
- Generate filtered download links for completed verification jobs
- Read verification history and account-level verification stats
- Create, list, and delete webhooks for lifecycle notifications

## How Authentication Works

BillionVerify MCP uses a standard remote OAuth flow:

1. Your MCP client discovers the protected resource metadata from `https://mcp.billionverify.com/.well-known/oauth-protected-resource`.
2. The client follows the declared authorization server metadata automatically.
3. You sign in with your BillionVerify account and approve the requested scopes.
4. The client calls `https://mcp.billionverify.com/mcp` with `Authorization: Bearer <access_token>`.

### Available Scopes

| Scope                    | Purpose                                                            |
| ------------------------ | ------------------------------------------------------------------ |
| `billionverify.read`     | Read balances, job status, history, statistics, and download links |
| `billionverify.verify`   | Run email verification tools                                       |
| `billionverify.webhooks` | Create, list, and delete webhooks                                  |

## Connect From ChatGPT

If you are connecting BillionVerify from ChatGPT, use the remote MCP server URL:

```text
https://mcp.billionverify.com/mcp
```

ChatGPT will discover the OAuth-protected resource metadata, redirect you to BillionVerify sign-in and consent, and then call the MCP server with an access token.

After the connection is approved, you can ask for things like:

- `Verify jane@company.com`
- `Check my current BillionVerify credit balance`
- `Show my recent verification history`
- `Get a download link for only valid emails from job 36f68e67-ddcb-441a-a407-22f826e72443`

## Connect From Other MCP Clients

Any MCP client that supports:

- remote MCP over Streamable HTTP
- OAuth-protected resources
- standard Bearer token authorization

can connect to the same endpoint:

```text
https://mcp.billionverify.com/mcp
```

Use your client's remote MCP configuration flow and provide the endpoint above. Do not wrap the server behind `curl --stdio`, and do not append an API key in the URL.

## When To Use REST API Instead

Use the BillionVerify REST API instead of MCP if:

- your client only supports local stdio MCP servers
- your integration depends on API keys instead of OAuth
- you are building a backend-to-backend automation without an MCP client

API keys still exist for the BillionVerify REST API. They are not accepted by the hosted MCP server.

If you want MCP with an API key, run the separate self-hosted server from the `billionverify-mcp` npm package. It runs locally over stdio and reads the key from `BILLIONVERIFY_API_KEY`. `BILLIONVERIFY_API_URL` is optional and defaults to `https://api.billionverify.com`. It requires Node.js 18 or later. Example Claude Desktop configuration:

```json
{
  "mcpServers": {
    "billionverify": {
      "command": "npx",
      "args": ["-y", "billionverify-mcp"],
      "env": {
        "BILLIONVERIFY_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

The self-hosted server exposes the same 11 tool names. Its tools also accept an optional `api_key` argument, and it has no OAuth scopes or resources.

## Available Tools

| Tool                       | Scope                    | Description                                                 |
| -------------------------- | ------------------------ | ----------------------------------------------------------- |
| `health_check`             | none                     | Check whether the MCP server is reachable and healthy       |
| `verify_single_email`      | `billionverify.verify`   | Verify one email address; supports `check_smtp` and `force_refresh` |
| `verify_batch_emails`      | `billionverify.verify`   | Verify up to 50 email addresses                             |
| `get_account_balance`      | `billionverify.read`     | Read current credit balance and usage summary               |
| `get_task_status`          | `billionverify.read`     | Read the status of an asynchronous verification task        |
| `get_download_url`         | `billionverify.read`     | Generate a result download URL for a completed job; filters: `valid`, `invalid`, `catchall`, `role`, `disposable`, `unknown` |
| `get_verification_history` | `billionverify.read`     | List verification history for the authenticated account     |
| `get_verification_stats`   | `billionverify.read`     | Read aggregated verification statistics                     |
| `create_webhook`           | `billionverify.webhooks` | Create a verification webhook                               |
| `list_webhooks`            | `billionverify.webhooks` | List configured webhooks                                    |
| `delete_webhook`           | `billionverify.webhooks` | Delete a webhook by ID                                      |

## Available Resources

| Resource URI                         | Scope                | Description                             |
| ------------------------------------ | -------------------- | --------------------------------------- |
| `billionverify://account/info`       | `billionverify.read` | Account-level credits and usage summary |
| `billionverify://history/summary`    | `billionverify.read` | Recent verification history summary     |
| `billionverify://stats/verification` | `billionverify.read` | Aggregated verification statistics      |

## Example Prompts

### Single Email Verification

> `Verify john@google.com`

### Batch Verification

> `Verify these emails in one request: john@google.com, test@mailinator.com, support@microsoft.com`

### Account Balance

> `How many BillionVerify credits do I have left?`

### Verification History

> `Show my 20 most recent verification jobs`

### Download Filtered Results

> `Get a download link for only valid emails from job job-123`

### Webhook Management

> `Create a webhook for file.completed and file.failed events at https://example.com/webhooks/billionverify`

## Example MCP Responses

### `verify_single_email`

#### Parameters

| Parameter       | Type    | Default | Description                                              |
| --------------- | ------- | ------- | -------------------------------------------------------- |
| `email`         | string  | —       | Email address to verify (required)                       |
| `check_smtp`    | boolean | `false` | Perform live SMTP mailbox check                          |
| `force_refresh` | boolean | `false` | Bypass cache and force a fresh live verification         |

#### Example Response

```json
{
  "email": "user@example.com",
  "status": "valid",
  "score": 1,
  "is_deliverable": true,
  "is_disposable": false,
  "is_catchall": false,
  "is_role": false,
  "is_free": true,
  "domain": "example.com",
  "mx_records": ["has_mx_records"],
  "smtp_check": true,
  "response_time": 2,
  "credits_used": 1
}
```

### `get_task_status`

The tool takes `task_id` (the job ID returned by a file upload) and returns the file job status from the REST API:

```json
{
  "job_id": "job-uuid-xxxxx",
  "status": "completed",
  "file_name": "emails.csv",
  "progress_percent": 100,
  "total_emails": 1000,
  "processed_emails": 1000,
  "valid_emails": 850,
  "invalid_emails": 100
}
```

### `get_account_balance`

```json
{
  "account_id": "acc_xxx",
  "credits_balance": 842740,
  "credits_consumed": 157260,
  "credits_added": 1000000,
  "last_updated": "2026-02-05T04:51:35Z"
}
```

## Troubleshooting

### I get `401`, an OAuth prompt, or `mcp/www_authenticate`

This usually means the client has not completed OAuth authorization yet, the access token expired, or the client is missing the required scope. Reconnect the MCP server and approve the requested scopes.

### My client only supports `curl --stdio` or local JSON config with API keys

That client is not compatible with the hosted server's OAuth model. Run the self-hosted `billionverify-mcp` npm package with your API key, or use the BillionVerify REST API.

### I used to connect with `?api_key=...` and it stopped working

That legacy flow has been removed from the MCP surface. Switch to an OAuth-capable MCP client and connect with the remote endpoint directly.

### How do credits work?

- Valid, invalid, catch-all and role results: 1 credit each
- Known disposable domains: free, unless you explicitly set `check_smtp` to true and SMTP gives a definite result
- Unknown results and syntactically invalid addresses: free

Use `get_account_balance` to inspect your remaining balance.

## FAQ

### Does the MCP server support API keys?

No. The hosted remote MCP server requires OAuth access tokens. API keys work with the REST API and with the self-hosted `billionverify-mcp` package, not with the hosted endpoint.

### Which clients are supported?

Any client that supports remote MCP over Streamable HTTP and OAuth-protected resources can connect. ChatGPT is one example. For clients without remote OAuth MCP support, use the REST API instead.

### Do I need to install an npm package?

Not for the hosted server. You connect to `https://mcp.billionverify.com/mcp` directly. The npm package `billionverify-mcp` is only needed for the self-hosted, API-key variant.

### Can I fetch history and stats from MCP?

Yes. Use `get_verification_history`, `get_verification_stats`, or the corresponding read-only resources.

## Next Steps

  - [AI SDK Integration](/docs/ai/sdks/): Build custom AI applications with email verification
  - [CLI Tools](/docs/ai/cli/): Use email verification in terminal
  - [API Reference](/docs/api/): Use the REST API for API-key-based integrations