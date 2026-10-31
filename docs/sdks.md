# SDKs

Canonical page: https://billionverify.github.io/docs/sdks/

BillionVerify provides SDKs for Node.js, TypeScript, Python, Go, PHP and Java. They wrap the REST API so you can add email verification to an application without writing the HTTP calls yourself.

## Available SDKs

The REST API is the source of truth. The SDKs below are open-source reference implementations, so if an SDK differs from the [API reference](/docs/api/), follow the API. You can read, copy or adapt the code, or call the API directly from any language.

| Language | Source code | Docs |
| --- | --- | --- |
| Node.js | [billionverify-node](https://github.com/BillionVerify/billionverify-node) | [Node.js SDK](/docs/sdks/node/) |
| TypeScript | [billionverify-node](https://github.com/BillionVerify/billionverify-node) | [TypeScript SDK](/docs/sdks/typescript/) |
| Python | [billionverify-python](https://github.com/BillionVerify/billionverify-python) | [Python SDK](/docs/sdks/python/) |
| Go | [billionverify-go](https://github.com/BillionVerify/billionverify-go) | [Go SDK](/docs/sdks/go/) |
| PHP | [billionverify-php](https://github.com/BillionVerify/billionverify-php) | [PHP SDK](/docs/sdks/php/) |
| Java | [billionverify-java](https://github.com/BillionVerify/billionverify-java) | [Java SDK](/docs/sdks/java/) |

## Why Use Our SDKs?

### Simplified Integration

The SDKs handle the HTTP requests, the API key header, response parsing and error mapping for you.

```javascript
// Without SDK
const response = await fetch('https://api.billionverify.com/v1/verify/single', {
  method: 'POST',
  headers: {
    'BV-API-KEY': apiKey,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ email: 'user@example.com' }),
});
const result = await response.json();

// With SDK
const result = await client.verify('user@example.com');
```

### Type Safety

The TypeScript, Go and Java SDKs have typed responses, and the Python SDK uses typed dataclasses. The PHP SDK returns plain arrays.

### Automatic Retries

Every SDK retries 429, 500, 502 and 503 responses with exponential backoff. The `retries` setting is the maximum number of attempts, including the first one.

### Rate Limit Handling

On a 429 response the SDKs wait for the `Retry-After` header (or back off exponentially) and retry. When the attempts run out they raise a rate limit error that carries the retry-after value.

## Common Features

All our SDKs provide:

| Feature | Description |
|---------|-------------|
| Single Verification | Verify individual email addresses |
| Bulk Verification | Verify up to 50 emails per request, or upload a file for an asynchronous job |
| Credit Management | Check the credit balance |
| Webhook Support | Create, list and delete webhooks |
| Error Handling | A separate error type for each HTTP status |
| Retry Logic | Automatic retry for 429, 500, 502 and 503 responses |
| Timeout Configuration | Configurable request timeout |

The webhook signature helper in each SDK signs the request body only, while the API also signs the timestamp. See the webhook section of each SDK page for a working verification snippet.

## Quick Comparison

| Language | Package Manager | Async Support | Examples in the docs |
|----------|----------------|---------------|----------------------|
| Node.js | npm / yarn | Promise-based | Express, Fastify, Next.js |
| Python | pip | Sync and async clients | Django, Flask, FastAPI |
| TypeScript | npm / yarn | Promise-based | Next.js, NestJS, Express |
| Go | go modules | Blocking calls with `context`, use goroutines | Gin, Echo, Fiber |
| PHP | Composer | None | Laravel |
| Java | Build from source | Blocking calls, wrap in `CompletableFuture` | Spring Boot |

## Installation Overview

<div data-code-tabs>

  
<div data-tab="Node.js">

```bash
npm install billionverify-sdk
# or
yarn add billionverify-sdk
```
  
</div>

  
<div data-tab="Python">

```bash
pip install billionverify
```
  
</div>

  
<div data-tab="Go">

```bash
go get github.com/BillionVerify/billionverify-go
```
  
</div>

  
<div data-tab="PHP">

```bash
composer require billionverify/billionverify-php:dev-main
```
  
</div>

  
<div data-tab="Java">

```bash
git clone https://github.com/BillionVerify/billionverify-java.git
cd billionverify-java
mvn install
```

```xml
<dependency>
  <groupId>io.github.billionverifier</groupId>
  <artifactId>billionverify-java</artifactId>
  <version>1.0.0</version>
</dependency>
```
  
</div>

</div>

## Getting Started

1. **Get your API Key** - Sign up at [BillionVerify](https://billionverify.com) and get your API key from the dashboard
2. **Install the SDK** - Use your language's package manager
3. **Initialize the client** - Pass your API key to the constructor (the SDKs do not read environment variables themselves)
4. **Start verifying** - Call the verify method

## Need Help?

- Check our [API Reference](/docs/api/) for detailed endpoint documentation
- View [Integration Guides](https://billionverify.com/docs/integration-guides) for platform integrations
- Contact [support](mailto:support@billionverify.com) for assistance