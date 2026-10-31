# Vercel Connect Connector

Canonical page: https://billionverify.github.io/docs/connectors/vercel/

The BillionVerify connector for Vercel Connect stores your API key once at the team level. Linked projects request the key at runtime, so no project keeps its own copy.

- **Listing:** [vercel.com/connect/billionverify](https://vercel.com/connect/billionverify)
- **Details:** [BillionVerify Is Now a Vercel Connect Connector](https://billionverify.com/blog/billionverify-vercel-connector)

## Add the connector

1. Create an API key on the [API Keys page](https://billionverify.com/auth/sign-in?next=/settings/api-keys). Give it a name such as `vercel-production`.
2. Open the [BillionVerify listing](https://vercel.com/connect/billionverify), or search for "billionverify" under Connect in your Vercel dashboard.
3. Name the connector, for example `billionverify-prod`. Your connector ID becomes `billionverify/billionverify-prod`.
4. Choose Shared API Keys or Per-User API Keys, paste the key and create the connector.
5. Link the projects and environments that may verify emails. A project that is not linked gets an error instead of a key.

### Set it up from the CLI

Run these commands from the project that will use the connector.

```bash
vercel link
vercel connect create billionverify --name billionverify-prod
vercel env pull
```

`vercel env pull` writes a short-lived development token to `.env.local`, so `getToken` also works on your own machine. That file holds a Vercel token, not your BillionVerify key.

## Choose a key type

| Key type | Use it when | Credits come from |
| --- | --- | --- |
| Shared API Keys | Your own product does the verifying | Your BillionVerify account |
| Per-User API Keys | Your users have their own BillionVerify accounts | Each user's account |

## Verify an email from your code

Install the Vercel Connect SDK.

```bash
npm install @vercel/connect
```

Request the key with `getToken`, then send it in the `BV-API-KEY` header.

```ts
// app/api/verify-email/route.ts
import { getToken } from '@vercel/connect';

export async function POST(req: Request) {
  const { email } = await req.json();

  const apiKey = await getToken('billionverify/billionverify-prod', {
    subject: { type: 'app' },
  });

  const res = await fetch('https://api.billionverify.com/v1/verify/single', {
    method: 'POST',
    headers: { 'BV-API-KEY': apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, check_smtp: true }),
  });

  const body = await res.json();
  return Response.json(body.data);
}
```

The SDK caches the key in memory, so it does not call Vercel Connect again for every address.

### Use per-user keys

Pass the signed-in user's ID as the subject.

```ts
const apiKey = await getToken('billionverify/billionverify-prod', {
  subject: { type: 'user', id: session.user.id },
});
```

If the user has not provided a key yet, the SDK throws `UserAuthorizationRequiredError`. Catch it and show a step that connects their BillionVerify account.

### Other endpoints

The same key works for every endpoint.

- [Verify a single email](/docs/api/verify-single/)
- [Verify in bulk](/docs/api/verify-bulk/), up to 50 addresses per request
- [Upload a file](/docs/api/upload-file/) for a background job

## Credits

Verifications through the connector use the credits in your BillionVerify account at the normal rate. Unknown results cost nothing. Vercel bills its own token requests separately under [Vercel Connect pricing](https://vercel.com/docs/connect/pricing).