# Python SDK

Canonical page: https://billionverify.github.io/docs/sdks/python/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-python](https://github.com/BillionVerify/billionverify-python)

The Python SDK has a synchronous client (`BillionVerify`) and an asynchronous client (`AsyncBillionVerify`), both built on `httpx`. It requires Python 3.8 or later.

## Installation

```bash
pip install billionverify
```

## Quick Start

```python
import os
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])

# Verify a single email
result = client.verify('user@example.com')
print(f"Status: {result.status}")  # 'valid', 'invalid', 'unknown', 'risky', 'disposable', 'catchall', or 'role'
```

## Configuration

### Client Options

```python
from billionverify import BillionVerify

client = BillionVerify(
    api_key='your-api-key',  # Required

    # Optional configurations
    base_url='https://api.billionverify.com/v1',  # Default; includes /v1
    timeout=30.0,            # Request timeout in seconds (default: 30)
    retries=3,               # Maximum attempts for retryable errors (default: 3)
    max_connections=100,     # HTTP connection pool size (default: 100)
)
```

An empty `api_key` raises `AuthenticationError`.

### Environment Variables

The SDK does not read environment variables by itself. Keep the key in one and pass it in:

```bash
# .env
BV_API_KEY=your_api_key_here
```

```python
import os
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])
```

## Single Email Verification

### Basic Verification

```python
result = client.verify('user@example.com')

print({
    'email': result.email,
    'status': result.status,
    'score': result.score,
    'deliverable': result.is_deliverable,
    'disposable': result.is_disposable,
    'role': result.is_role,
})
```

### Verification with Options

```python
result = client.verify(
    'user@example.com',
    check_smtp=True,                  # Perform SMTP verification (default: True)
    force_refresh=False,              # Bypass cached results (default: False)
    include_domain_reputation=False,  # Include domain reputation details (default: False)
)
```

### Response Object

```python
@dataclass
class VerificationResult:
    email: str
    status: str  # 'valid', 'invalid', 'unknown', 'risky', 'disposable', 'catchall', 'role'
    score: float
    is_deliverable: bool
    is_disposable: bool
    is_catchall: bool
    is_role: bool
    is_free: bool
    domain: str
    check_smtp: bool
    reason: str
    response_time: int
    credits_used: int
    domain_age: Optional[int]
    mx_records: List[str]
    domain_reputation: Optional[DomainReputation]
    domain_suggestion: Optional[str]
    has_gravatar: bool
    gravatar_url: Optional[str]
    smtp_response: Optional[str]
    error_message: Optional[str]
    response_metadata: Optional[ResponseMetadata]  # HTTP status, request ID, rate-limit headers
```

## Async Usage

For concurrent workloads, use the async client:

```python
import asyncio
import os
from billionverify import AsyncBillionVerify

async def main():
    client = AsyncBillionVerify(api_key=os.environ['BV_API_KEY'])

    # Single verification
    result = await client.verify('user@example.com')
    print(result.status)

    # Verify multiple emails concurrently
    emails = ['user1@example.com', 'user2@example.com', 'user3@example.com']
    results = await asyncio.gather(*[client.verify(email) for email in emails])

    for result in results:
        print(f"{result.email}: {result.status}")

    await client.close()

asyncio.run(main())
```

### Context Manager

```python
async with AsyncBillionVerify(api_key=os.environ['BV_API_KEY']) as client:
    result = await client.verify('user@example.com')
```

## Bulk Verification

### Verify a Batch Synchronously

`verify_bulk()` verifies up to 50 emails in one request and returns the results directly. With more than 50 emails it raises `ValidationError` before calling the API.

```python
emails = [
    'user1@example.com',
    'user2@example.com',
    'user3@example.com',
]

batch = client.verify_bulk(emails)
print(f"Total: {batch.total_emails}, valid: {batch.valid_emails}, invalid: {batch.invalid_emails}")
print(f"Credits used: {batch.credits_used}")

for item in batch.results:
    print(f"{item.email}: {item.status}")
```

### Upload a File for Larger Lists

For larger lists, upload a file (CSV, Excel or TXT). The job runs asynchronously. `upload_file()` takes a path on disk.

```python
upload = client.upload_file(
    'emails.csv',
    email_column='email',     # Column that contains the emails
    check_smtp=True,
    preserve_original=True,   # Keep the original columns in the results
)

print(f"Task ID: {upload.task_id}")
print(f"Status: {upload.status}")
```

### Check Job Status

```python
status = client.get_file_task_status(upload.task_id)

print({
    'id': status.task_id,
    'status': status.status,      # 'pending', 'parsing', 'processing', 'completed', 'failed'
    'progress': status.progress,
    'total': status.total_emails,
    'processed': status.processed_emails,
    'valid': status.valid_emails,
    'invalid': status.invalid_emails,
})
```

`get_file_task_status(task_id, timeout=30)` long-polls for up to `timeout` seconds (0 to 300) instead of returning immediately.

### Get Results

```python
# Poll until the task is completed or failed (every 5 s, up to 600 s by default)
done = client.wait_for_file_task(upload.task_id)

if done.status == 'completed':
    # Saves the result CSV and returns the path.
    # Filters are optional: valid, invalid, catchall, role, unknown, disposable, risky.
    path = client.download_file_results(upload.task_id, 'results.csv', valid=True)
    print(f"Saved to {path}")
```

`wait_for_file_task(task_id, poll_interval, max_wait)` takes seconds and raises `TimeoutError` if the task is still running after `max_wait`.

### With Webhook

Instead of polling, register a webhook. It is called when a file job finishes (`file.completed`) or fails (`file.failed`):

```python
webhook = client.create_webhook(
    url='https://your-domain.com/webhooks/billionverify',
    events=['file.completed', 'file.failed'],
)

print(webhook.id)
print(webhook.secret)  # Returned only when the webhook is created. Store it.

webhooks = client.list_webhooks()
client.delete_webhook(webhook.id)
```

Each delivery is a `POST` with a JSON body of the form `{event, timestamp, data}`. The API signs it with HMAC-SHA256 over `"<timestamp>.<raw body>"`, using the webhook secret, and sends these headers:

- `X-Webhook-Signature`: `sha256=<hex digest>`
- `X-Webhook-Timestamp`: Unix time in seconds, the value used in the signature
- `X-Webhook-Event`: the event name

Verify against the raw body before parsing it:

```python
import hashlib
import hmac

def is_valid_webhook(raw_body: bytes, headers: dict, secret: str) -> bool:
    timestamp = headers['X-Webhook-Timestamp']
    received = headers['X-Webhook-Signature']
    signed = timestamp.encode() + b'.' + raw_body
    expected = 'sha256=' + hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()
    return hmac.compare_digest(received, expected)
```

> The SDK's `BillionVerify.verify_webhook_signature(payload, signature, secret)` signs the body only, without the timestamp prefix, so it does not match what the API sends. Use the function above for real deliveries.

## Credit Management

```python
credits = client.get_credits()

print({
    'balance': credits.credits_balance,
    'consumed': credits.credits_consumed,
    'added': credits.credits_added,
})
```

## Error Handling

### Exception Types

All exceptions extend `BillionVerifyError`, which has `code`, `status_code`, `details` and `response_metadata`.

| Exception | HTTP status | Notes |
| --- | --- | --- |
| `AuthenticationError` | 401 | Missing or invalid API key |
| `InsufficientCreditsError` | 402 | Not enough credits |
| `NotFoundError` | 404 | For example an unknown task ID |
| `ValidationError` | 400 | Also raised locally for more than 50 emails or a missing file |
| `RateLimitError` | 429 | Has `retry_after` in seconds |
| `TimeoutError` | 504 | Request timed out, or `wait_for_file_task()` ran out of time |

`TimeoutError` here is `billionverify.TimeoutError`, not the Python built-in.

```python
from billionverify import (
    BillionVerify,
    BillionVerifyError,
    AuthenticationError,
    InsufficientCreditsError,
    RateLimitError,
    ValidationError,
)

try:
    result = client.verify('user@example.com')
except AuthenticationError:
    print('Invalid API key')
except InsufficientCreditsError:
    print('Not enough credits')
except RateLimitError as e:
    print(f'Rate limited. Retry after {e.retry_after} seconds')
except ValidationError as e:
    print(f'Invalid input: {e}')
except BillionVerifyError as e:
    print(f'API error: {e}')
```

### Retry Configuration

The SDK retries 429, 500, 502 and 503 responses with exponential backoff (`2 ** attempt` seconds, or the `Retry-After` header on 429). `retries` is the maximum number of attempts, including the first one. There are no other retry options.

```python
client = BillionVerify(
    api_key=os.environ['BV_API_KEY'],
    retries=5,  # Up to 5 attempts for retryable errors
)
```

## Framework Integration

### Django

```python
# views.py
import json
import os

from django.http import JsonResponse
from django.views.decorators.http import require_POST
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])

@require_POST
def verify_email(request):
    data = json.loads(request.body)
    email = data.get('email')

    try:
        result = client.verify(email)

        if result.status == 'invalid':
            return JsonResponse({
                'valid': False,
                'message': 'Please enter a valid email address',
            }, status=400)

        if result.is_disposable:
            return JsonResponse({
                'valid': False,
                'message': 'Disposable emails are not allowed',
            }, status=400)

        return JsonResponse({'valid': True, 'score': result.score})
    except Exception as e:
        return JsonResponse({'error': str(e)}, status=500)
```

### Django Form Validation

```python
# forms.py
import os

from django import forms
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])

class RegistrationForm(forms.Form):
    email = forms.EmailField()

    def clean_email(self):
        email = self.cleaned_data['email']
        result = client.verify(email)

        if result.status == 'invalid':
            raise forms.ValidationError('Please enter a valid email address')

        if result.is_disposable:
            raise forms.ValidationError('Disposable emails are not allowed')

        return email
```

### Flask

```python
import os

from flask import Flask, request, jsonify
from billionverify import BillionVerify

app = Flask(__name__)
client = BillionVerify(api_key=os.environ['BV_API_KEY'])

@app.route('/verify', methods=['POST'])
def verify_email():
    email = request.json.get('email')

    try:
        result = client.verify(email)
        return jsonify({
            'email': result.email,
            'status': result.status,
            'score': result.score,
        })
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### FastAPI

```python
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from billionverify import AsyncBillionVerify

client = AsyncBillionVerify(api_key=os.environ['BV_API_KEY'])

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await client.close()

app = FastAPI(lifespan=lifespan)

class EmailRequest(BaseModel):
    email: str

@app.post('/verify')
async def verify_email(request: EmailRequest):
    try:
        result = await client.verify(request.email)
        return {
            'email': result.email,
            'status': result.status,
            'score': result.score,
            'deliverable': result.is_deliverable,
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Complete Examples

### User Registration Validation

```python
import os
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])

def validate_registration_email(email: str) -> dict:
    """Validate email for user registration."""
    result = client.verify(email)

    # Reject invalid emails
    if result.status == 'invalid':
        return {
            'valid': False,
            'reason': 'invalid_email',
            'message': 'This email address is invalid',
        }

    # Reject disposable emails
    if result.is_disposable:
        return {
            'valid': False,
            'reason': 'disposable',
            'message': 'Please use a permanent email address',
        }

    # Warn about role-based emails
    if result.is_role:
        return {
            'valid': True,
            'warning': 'role_based',
            'message': 'Using a personal email is recommended',
        }

    return {
        'valid': True,
        'score': result.score,
    }

# Usage
validation = validate_registration_email('user@example.com')
if not validation['valid']:
    print(validation['message'])
```

### Bulk List Cleaning

```python
import os
from billionverify import BillionVerify

client = BillionVerify(api_key=os.environ['BV_API_KEY'])

def clean_email_list(input_file: str, output_file: str):
    """Upload a CSV, wait for the job and save the verified results."""

    # Upload the CSV (it must have an 'email' column)
    upload = client.upload_file(input_file, email_column='email')
    print(f"Submitted task {upload.task_id}")

    # Wait for completion (polls every 5 s, up to 10 min by default)
    status = client.wait_for_file_task(upload.task_id)

    if status.status == 'failed':
        raise Exception(f"Task failed: {status.error_message}")

    # Save the result CSV
    client.download_file_results(upload.task_id, output_file)

    print(f"Results written to {output_file}")
    print(f"Valid: {status.valid_emails}, Invalid: {status.invalid_emails}")

# Usage
clean_email_list('contacts.csv', 'cleaned_contacts.csv')
```

### Async Concurrent Verification

```python
import asyncio
import os
from typing import List
from billionverify import AsyncBillionVerify

async def verify_emails_concurrent(emails: List[str], concurrency: int = 10):
    """Verify multiple emails with controlled concurrency."""

    client = AsyncBillionVerify(api_key=os.environ['BV_API_KEY'])
    semaphore = asyncio.Semaphore(concurrency)

    async def verify_with_semaphore(email):
        async with semaphore:
            return await client.verify(email)

    tasks = [verify_with_semaphore(email) for email in emails]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    await client.close()

    # Process results
    valid = []
    invalid = []
    errors = []

    for email, result in zip(emails, results):
        if isinstance(result, Exception):
            errors.append((email, str(result)))
        elif result.status == 'valid':
            valid.append(email)
        else:
            invalid.append(email)

    return {
        'valid': valid,
        'invalid': invalid,
        'errors': errors,
    }

# Usage
emails = ['user1@example.com', 'user2@example.com', 'user3@example.com']
results = asyncio.run(verify_emails_concurrent(emails))
print(f"Valid: {len(results['valid'])}, Other: {len(results['invalid'])}")
```

## Type Hints

The SDK is type-annotated, so editors and mypy can check your calls:

```python
from typing import Optional
from billionverify import BillionVerify, VerificationResult

def process_verification(result: VerificationResult) -> Optional[str]:
    if result.status == 'valid':
        return result.email
    return None
```

## Next Steps

  - [API Reference](/docs/api/): Detailed API endpoint documentation
  - [Integration Guides](https://billionverify.com/docs/integration-guides): Platform integrations