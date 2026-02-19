---
layout: default
title: BillionVerify - Email Verification API & SDKs
---

<img src="/assets/images/logo-text.svg" alt="BillionVerify.com" class="logo">

**Professional Email Verification API with 99.9% Accuracy**

Validate email addresses in real-time or clean millions of records with our fast, reliable, and affordable email verification service.

[Get Started Free](https://billionverify.com){: .btn} [View API Docs](https://billionverify.com/docs){: .btn}

---

## Why BillionVerify?

| Feature | BillionVerify |
|---------|----------------|
| **Accuracy** | 99.9% verification accuracy |
| **Speed** | Sub-300ms response time |
| **Throughput** | 50,000 emails/hour bulk processing |
| **Pricing** | Starting at $0.0003/email |

### Multi-Layer Verification

Every email passes through our comprehensive verification stack:

- **Syntax Validation** - Catch formatting errors and typos
- **Domain Verification** - Confirm domain exists and is configured
- **MX Record Check** - Verify mail servers are active
- **SMTP Verification** - Validate specific mailbox existence
- **AI-Powered Detection** - Identify disposables, spam traps, and catch-all domains

---

## REST API

Our REST API enables custom integrations with any platform. Simple HTTP requests, predictable JSON responses.

### Quick Example

```bash
curl -X GET "https://api.billionverify.com/v1/verify?email=test@example.com" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Response

```json
{
  "email": "test@example.com",
  "is_valid": true,
  "is_disposable": false,
  "is_role_based": false,
  "is_catch_all": false,
  "mx_found": true,
  "smtp_check": true
}
```

[Full API Reference →](https://billionverify.com/docs/api-reference)

---

## Official SDKs

We provide official SDKs for popular programming languages. Each SDK includes single and bulk verification, credit management, webhook configuration, automatic retry logic, and rate limit handling.

### Node.js / TypeScript

Full JavaScript and TypeScript support for Node.js applications.

```bash
npm install @emailverify/node
# or
yarn add @emailverify/node
```

```javascript
import { BillionVerify } from '@emailverify/node';

const client = new BillionVerify('YOUR_API_KEY');
const result = await client.verify('test@example.com');

console.log(result.is_valid);
```

[Node.js SDK Docs →](https://billionverify.com/docs/sdks/node)

---

### Python

Clean Pythonic interface for scripts and applications.

```bash
pip install emailverify
```

```python
from emailverify import BillionVerify

client = BillionVerify('YOUR_API_KEY')
result = client.verify('test@example.com')

print(result.is_valid)
```

[Python SDK Docs →](https://billionverify.com/docs/sdks/python)

---

### Go

Native Go SDK with goroutine concurrency support.

```bash
go get github.com/emailverify/go-sdk
```

```go
package main

import "github.com/emailverify/go-sdk"

func main() {
    client := emailverify.NewClient("YOUR_API_KEY")
    result, _ := client.Verify("test@example.com")

    fmt.Println(result.IsValid)
}
```

[Go SDK Docs →](https://billionverify.com/docs/sdks/go)

---

### PHP

PHP SDK with Laravel framework integration.

```bash
composer require emailverify/php-sdk
```

```php
<?php
use BillionVerify\Client;

$client = new Client('YOUR_API_KEY');
$result = $client->verify('test@example.com');

echo $result->isValid;
```

[PHP SDK Docs →](https://billionverify.com/docs/sdks/php)

---

### Java

Java SDK supporting Spring Boot applications.

```xml
<dependency>
  <groupId>com.emailverify</groupId>
  <artifactId>emailverify-java</artifactId>
  <version>1.0.0</version>
</dependency>
```

```java
import com.emailverify.BillionVerifyClient;

BillionVerifyClient client = new BillionVerifyClient("YOUR_API_KEY");
VerificationResult result = client.verify("test@example.com");

System.out.println(result.isValid());
```

[Java SDK Docs →](https://billionverify.com/docs/sdks/java)

---

## SDK Features

All SDKs include:

| Feature | Description |
|---------|-------------|
| **Single Verification** | Verify individual emails in real-time |
| **Bulk Verification** | Process large lists efficiently |
| **Credit Management** | Check balance, purchase credits |
| **Webhooks** | Configure notification endpoints |
| **Error Handling** | Comprehensive exception handling |
| **Auto Retry** | Automatic retry on transient failures |
| **Rate Limiting** | Built-in rate limit handling |
| **Timeouts** | Customizable request timeouts |

---

## Pricing

Transparent credit-based pricing that scales with your needs.

| Plan | Monthly | Credits | Per Email |
|------|---------|---------|-----------|
| Starter | $4 | 1,000 | $0.004 |
| Popular | $16 | 5,000 | $0.0032 |
| Professional | $28 | 10,000 | $0.0028 |
| Business | $48 | 20,000 | $0.0024 |
| Enterprise | $60 | 50,000 | $0.0012 |
| Premium | $80 | 100,000 | $0.0008 |

**50% off annual plans** • Credits never expire • 100 free credits on signup

[View All Plans →](https://billionverify.com/pricing)

---

## Integrations

Native integrations with 25+ platforms:

**Email Marketing:** Mailchimp, SendGrid, Klaviyo, Brevo, Campaign Monitor

**CRM:** Salesforce, HubSpot, Pipedrive

**Automation:** Zapier, Make (Integromat)

**Forms:** Typeform, JotForm, Gravity Forms

---

## Get Started

1. **Sign Up** - Create an account at [billionverify.com](https://billionverify.com)
2. **Get API Key** - Find your key in the dashboard
3. **Install SDK** - Choose your language and install
4. **Start Verifying** - Make your first API call

**100 free credits** on signup. No credit card required.

[Create Free Account →](https://billionverify.com)

---

## Resources

- [API Documentation](https://billionverify.com/docs)
- [SDK Reference](https://billionverify.com/docs/sdks)
- [Pricing](https://billionverify.com/pricing)
- [Blog](https://billionverify.com/blog)
- [Support](mailto:support@billionverify.com)
