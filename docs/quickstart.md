# Quickstart

Canonical page: https://billionverify.github.io/docs/quickstart/

This guide walks you through verifying your first email address with the BillionVerify API.

## Prerequisites

Before you begin, you'll need:
- A BillionVerify account ([Sign up](https://billionverify.com/auth/sign-up))
- Familiarity with HTTP requests or one of the supported SDK languages

## Step 1: Create an Account

1. Go to [BillionVerify](https://billionverify.com/auth/sign-up)
2. Sign up and confirm your email address

## Step 2: Get Your API Key

1. Log in to your [BillionVerify dashboard](https://billionverify.com/auth/sign-in?next=/home)
2. Navigate to the **API Keys** section
3. Click **Create API Key**
4. Give your key a descriptive name (e.g., "Development", "Production")
5. Copy your API key and store it securely

> Never put your API key in client-side code or a public repository. Store it in an environment variable.

## Step 3: Verify Your First Email

Choose your preferred method to make your first verification request:

<div data-code-tabs>

  
<div data-tab="cURL">

```bash
curl -X POST https://api.billionverify.com/v1/verify/single \
  -H "BV-API-KEY: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com"}'
```
  
</div>

  
<div data-tab="JavaScript">

```javascript
const response = await fetch('https://api.billionverify.com/v1/verify/single', {
  method: 'POST',
  headers: {
    'BV-API-KEY': process.env.BV_API_KEY,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    email: 'test@example.com',
  }),
});

const result = await response.json();
console.log(result);
```
  
</div>

  
<div data-tab="Python">

```python
import os
import requests

response = requests.post(
    'https://api.billionverify.com/v1/verify/single',
    headers={
        'BV-API-KEY': os.environ['BV_API_KEY'],
        'Content-Type': 'application/json',
    },
    json={'email': 'test@example.com'}
)

result = response.json()
print(result)
```
  
</div>

  
<div data-tab="Go">

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
)

func main() {
    data := map[string]string{"email": "test@example.com"}
    body, _ := json.Marshal(data)

    req, _ := http.NewRequest("POST",
        "https://api.billionverify.com/v1/verify/single",
        bytes.NewBuffer(body))

    req.Header.Set("BV-API-KEY", os.Getenv("BV_API_KEY"))
    req.Header.Set("Content-Type", "application/json")

    client := &http.Client{}
    resp, _ := client.Do(req)
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    fmt.Println(result)
}
```
  
</div>

  
<div data-tab="PHP">

```php
<?php
$apiKey = getenv('BV_API_KEY');

$ch = curl_init('https://api.billionverify.com/v1/verify/single');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => [
        'BV-API-KEY: ' . $apiKey,
        'Content-Type: application/json',
    ],
    CURLOPT_POSTFIELDS => json_encode(['email' => 'test@example.com']),
]);

$response = curl_exec($ch);
curl_close($ch);

$result = json_decode($response, true);
print_r($result);
```
  
</div>

</div>

## Step 4: Understand the Response

A successful verification returns a response like this:

```json
{
  "success": true,
  "code": "0",
  "message": "Success",
  "data": {
    "email": "test@example.com",
    "status": "valid",
    "score": 1,
    "is_deliverable": true,
    "is_disposable": false,
    "is_catchall": false,
    "is_role": false,
    "is_free": false,
    "domain": "example.com",
    "mx_records": ["mail.example.com"],
    "check_smtp": true,
    "smtp_result": "deliverable",
    "reason": "smtp_deliverable",
    "response_time": 250,
    "credits_used": 1,
    "verification_mode": "smtp",
    "retryable": false,
    "smtputf8": false
  }
}
```

### Key Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Overall status: `valid`, `invalid`, `unknown`, `catchall`, `role`, `disposable` (or `risky` on results created before 2026-09-11) |
| `is_deliverable` | boolean | Whether the verification confirmed that the address can receive mail |
| `is_disposable` | boolean | Whether it is a temporary/disposable email |
| `is_role` | boolean | Whether it is a role-based address (info@, support@) |
| `is_catchall` | boolean | Whether the domain accepts all addresses |
| `score` | number | Quality score from 0 to 1. It is a heuristic, not a probability. Branch on `status`, not on `score` |
| `reason` | string | Why the address got this status. See [Verification Reasons](/docs/verification-reasons/) |
| `credits_used` | integer | Credits charged for this request |

### Status Values Explained

| Status | Meaning | Recommended Action |
|--------|---------|-------------------|
| `valid` | Email exists and can receive messages | Safe to send |
| `invalid` | Email doesn't exist or can't receive messages | Remove from list |
| `unknown` | Deliverability could not be confirmed | Retry later or use caution |
| `catchall` | Domain accepts all addresses | Monitor for bounces |
| `role` | Role-based email (info@, support@) | Usually deliverable |
| `disposable` | Temporary email service | Usually remove |

`unknown` results and addresses with invalid syntax are not charged.

## Step 5: Use an SDK (Recommended)

For easier integration, install one of our official SDKs:

<div data-code-tabs>

  
<div data-tab="Node.js">

```bash
npm install billionverify-sdk
```

```javascript
import { BillionVerify } from 'billionverify-sdk';

const client = new BillionVerify({
  apiKey: process.env.BV_API_KEY,
});

const result = await client.verify('test@example.com');
console.log(result.status); // 'valid'
```
  
</div>

  
<div data-tab="Python">

```bash
pip install billionverify
```

```python
from billionverify import BillionVerify
import os

client = BillionVerify(api_key=os.environ['BV_API_KEY'])
result = client.verify('test@example.com')
print(result.status)  # 'valid'
```
  
</div>

  
<div data-tab="Go">

```bash
go get github.com/BillionVerify/billionverify-go
```

```go
import billionverify "github.com/BillionVerify/billionverify-go"

client, err := billionverify.NewClient(billionverify.Config{
    APIKey: os.Getenv("BV_API_KEY"),
})
if err != nil {
    log.Fatal(err)
}

result, err := client.Verify(ctx, "test@example.com", nil)
fmt.Println(result.Status) // "valid"
```
  
</div>

  
<div data-tab="PHP">

```bash
composer require billionverify/billionverify-php:dev-main
```

```php
use BillionVerify\Client;

$client = new Client(getenv('BV_API_KEY'));
$result = $client->verify('test@example.com');
echo $result['data']['status']; // 'valid'
```
  
</div>

</div>

## What's Next?

Now that you've verified your first email, explore more features:

  - [API Reference](/docs/api/): Complete API documentation with all endpoints
  - [SDK Documentation](/docs/sdks/): Detailed guides for each SDK
  - [Verification Types](https://billionverify.com/docs/product/verification-types): Understand different result types
  - [Verification Reasons](/docs/verification-reasons/): What each `reason` value means
  - [AI Integration](/docs/ai/): Use the API from Claude, ChatGPT and other AI tools

## Common Use Cases

### Form Validation
Verify emails in real time during user registration. For a free check that only detects disposable domains, use [`/verify/disposable`](/docs/api/disposable-check/):

```javascript
const onEmailBlur = async (email) => {
  const result = await client.verify(email);

  if (result.status === 'invalid') {
    showError('Please enter a valid email');
  } else if (result.is_disposable) {
    showError('Disposable emails are not allowed');
  }
};
```

### List Cleaning
Check each address before sending a campaign. For long lists, [upload a file](/docs/api/upload-file/) instead of looping over single requests:

```python
for email in email_list:
    result = client.verify(email)
    if result.status == 'valid':
        clean_list.append(email)
    else:
        remove_list.append(email)
```

### Lead Qualification
Combine result fields into your own lead score:

```javascript
const qualifyLead = (result) => {
  let score = result.score * 50; // Base score

  if (!result.is_free) score += 20;      // Business email
  if (!result.is_role) score += 15;      // Personal address
  if (!result.is_disposable) score += 15; // Permanent email

  return score;
};
```

## Need Help?

- Browse our [Integration Guides](https://billionverify.com/docs/integration-guides) for implementation examples
- Contact [support@billionverify.com](mailto:support@billionverify.com) for assistance