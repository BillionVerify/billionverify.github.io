# PHP SDK

Canonical page: https://billionverify.github.io/docs/sdks/php/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-php](https://github.com/BillionVerify/billionverify-php)

The PHP SDK is a thin client built on Guzzle. Methods return the decoded JSON response as a PHP array, so results are read as `$result['data']['status']`. The SDK has no Laravel package; the Laravel section below shows how to wire the client into an app yourself.

## Installation

```bash
composer require billionverify/billionverify-php:dev-main
```

## Requirements

- PHP 8.1 or higher
- Composer
- Guzzle 7 (installed by Composer)

## Quick Start

```php
<?php

require 'vendor/autoload.php';

use BillionVerify\Client;

$client = new Client(getenv('BV_API_KEY'));

$result = $client->verify('user@example.com');
echo "Status: " . $result['data']['status'] . "\n";
echo "Score: " . $result['data']['score'] . "\n";
```

## Configuration

### Client Options

```php
$client = new Client(
    apiKey: getenv('BV_API_KEY'),                 // Required
    baseUrl: 'https://api.billionverify.com/v1',  // Optional, default shown (includes /v1)
    timeout: 30,                                  // Optional: request timeout in seconds (default: 30)
    retries: 3                                    // Optional: maximum attempts for retryable errors (default: 3)
);
```

An empty `$apiKey` throws `AuthenticationException`.

### Environment Configuration

The SDK does not read environment variables by itself. Keep the key in one and pass it in:

```bash
# .env
BV_API_KEY=your_api_key_here
```

## Single Verification

### Basic Verification

```php
$result = $client->verify('user@example.com');
$data = $result['data'];

echo "Email: " . $data['email'] . "\n";
echo "Status: " . $data['status'] . "\n";
echo "Score: " . $data['score'] . "\n";
echo "Deliverable: " . ($data['is_deliverable'] ? 'Yes' : 'No') . "\n";
echo "Disposable: " . ($data['is_disposable'] ? 'Yes' : 'No') . "\n";
```

### Verification with Options

```php
$result = $client->verify('user@example.com', checkSmtp: true); // default: true
```

### Response Structure

`$result['data']` contains these fields (optional fields can be missing):

```php
[
    'email'             => 'user@example.com',
    'status'            => 'valid',   // 'valid', 'invalid', 'unknown', 'risky', 'disposable', 'catchall', 'role'
    'score'             => 0.95,
    'is_deliverable'    => true,
    'is_disposable'     => false,
    'is_catchall'       => false,
    'is_role'           => false,
    'is_free'           => false,
    'domain'            => 'example.com',
    'domain_age'        => 9500,      // optional
    'mx_records'        => ['mx1.example.com'], // optional
    'domain_reputation' => [          // optional
        'mx_ip' => '93.184.216.34',
        'is_listed' => false,
        'blacklists' => [],
        'checked' => true,
    ],
    'smtp_check'        => true,
    'reason'            => '',        // optional
    'suggestion'        => '',        // optional
    'response_time'     => 1234,
    'credits_used'      => 1,
]
```

## Bulk Verification

### Verify a Batch Synchronously

`verifyBatch()` verifies up to 50 emails in one request and returns the results directly. With more than 50 emails it throws `ValidationException` before calling the API.

```php
$emails = [
    'user1@example.com',
    'user2@example.com',
    'user3@example.com',
];

$batch = $client->verifyBatch($emails)['data'];

echo "Total: " . $batch['total_emails'] . ", valid: " . $batch['valid_emails'] . ", invalid: " . $batch['invalid_emails'] . "\n";

foreach ($batch['results'] as $item) {
    echo $item['email'] . ": " . $item['status'] . "\n";
}
```

### Upload a File for Larger Lists

For larger lists, upload a file (CSV, Excel or TXT). The job runs asynchronously. `uploadFile()` takes a path on disk.

```php
$upload = $client->uploadFile(
    filePath: 'emails.csv',
    checkSmtp: true,
    emailColumn: 'email',      // Optional: auto-detected when null
    preserveOriginal: true     // Keep the original columns in the results (default: true)
)['data'];

echo "Task ID: " . $upload['task_id'] . "\n";
```

### Check Status

```php
$status = $client->getFileJobStatus($upload['task_id'])['data'];

echo "Status: " . $status['status'] . "\n"; // 'pending', 'processing', 'completed', 'failed'
echo "Progress: " . $status['processed_emails'] . "/" . $status['total_emails'] . "\n";
echo "Valid: " . $status['valid_emails'] . "\n";
echo "Invalid: " . $status['invalid_emails'] . "\n";
```

`getFileJobStatus($jobId, $timeout)` long-polls for up to `$timeout` seconds (capped at 300) instead of returning immediately.

### Get Results

```php
// Poll every 5 s, up to 600 s by default
$done = $client->waitForFileJobCompletion($upload['task_id']);

if ($done['data']['status'] === 'completed') {
    // Returns the result CSV as a string. Filters are optional.
    $csv = $client->getFileJobResults($upload['task_id'], valid: true);
    file_put_contents('results.csv', $csv);
}
```

`waitForFileJobCompletion($jobId, $pollInterval, $maxWait)` takes seconds and throws `TimeoutException` if the job is still running after `$maxWait`.

### With Webhook

Instead of polling, register a webhook. It is called when a file job finishes (`file.completed`) or fails (`file.failed`):

```php
$webhook = $client->createWebhook(
    'https://your-domain.com/webhooks/billionverify',
    ['file.completed', 'file.failed']
)['data'];

echo $webhook['id'] . "\n";
echo $webhook['secret'] . "\n"; // Returned only when the webhook is created. Store it.

$list = $client->listWebhooks();
$client->deleteWebhook($webhook['id']);
```

Each delivery is a `POST` with a JSON body of the form `{event, timestamp, data}`. The API signs it with HMAC-SHA256 over `"<timestamp>.<raw body>"`, using the webhook secret, and sends these headers:

- `X-Webhook-Signature`: `sha256=<hex digest>`
- `X-Webhook-Timestamp`: Unix time in seconds, the value used in the signature
- `X-Webhook-Event`: the event name

Verify against the raw body before parsing it:

```php
$rawBody   = file_get_contents('php://input');
$timestamp = $_SERVER['HTTP_X_WEBHOOK_TIMESTAMP'] ?? '';
$received  = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';

$expected = 'sha256=' . hash_hmac('sha256', $timestamp . '.' . $rawBody, $webhookSecret);

if (!hash_equals($expected, $received)) {
    http_response_code(401);
    exit;
}

$payload = json_decode($rawBody, true);
```

> The SDK's `Client::verifyWebhookSignature($payload, $signature, $secret)` signs the body only, without the timestamp prefix, so it does not match what the API sends. Use the code above for real deliveries.

## Credit Management

```php
$credits = $client->getCredits()['data'];

echo "Balance: " . $credits['credits_balance'] . "\n";
echo "Consumed: " . $credits['credits_consumed'] . "\n";
echo "Added: " . $credits['credits_added'] . "\n";
```

## Error Handling

### Exception Types

All exceptions are in the `BillionVerify\Exception` namespace and extend `BillionVerifyException`, which has `getErrorCode()`, `getStatusCode()` and `getDetails()`.

| Exception | HTTP status | Notes |
| --- | --- | --- |
| `AuthenticationException` | 401 | Missing or invalid API key |
| `InsufficientCreditsException` | 402 | Not enough credits |
| `NotFoundException` | 404 | For example an unknown job ID |
| `ValidationException` | 400 | Also thrown locally for more than 50 emails or a missing file |
| `RateLimitException` | 429 | `getRetryAfter()` returns seconds |
| `TimeoutException` | 504 | Connection failed or timed out, or `waitForFileJobCompletion()` ran out of time |

```php
use BillionVerify\Client;
use BillionVerify\Exception\AuthenticationException;
use BillionVerify\Exception\InsufficientCreditsException;
use BillionVerify\Exception\RateLimitException;
use BillionVerify\Exception\ValidationException;
use BillionVerify\Exception\BillionVerifyException;

try {
    $result = $client->verify('user@example.com');
} catch (AuthenticationException $e) {
    echo "Invalid API key\n";
} catch (InsufficientCreditsException $e) {
    echo "Not enough credits\n";
} catch (RateLimitException $e) {
    echo "Rate limited. Retry after " . $e->getRetryAfter() . " seconds\n";
} catch (ValidationException $e) {
    echo "Invalid input: " . $e->getMessage() . "\n";
} catch (BillionVerifyException $e) {
    echo "API error: " . $e->getMessage() . "\n";
}
```

The SDK retries 429, 500, 502 and 503 responses with exponential backoff (`2^attempt` seconds, or the `Retry-After` header on 429) before it throws. `retries` is the maximum number of attempts, including the first one.

## Laravel Integration

The SDK ships no service provider, facade or validation rule. The examples below use the plain `Client` class and add the Laravel glue in your own app.

### Service Provider

Register the client as a singleton, for example in `app/Providers/AppServiceProvider.php`:

```php
use BillionVerify\Client;

public function register(): void
{
    $this->app->singleton(Client::class, function () {
        return new Client(
            apiKey: config('services.billionverify.key'),
            timeout: (int) config('services.billionverify.timeout', 30),
            retries: (int) config('services.billionverify.retries', 3),
        );
    });
}
```

### Configuration

Add the settings to `config/services.php`:

```php
'billionverify' => [
    'key' => env('BV_API_KEY'),
    'timeout' => env('BV_TIMEOUT', 30),
    'retries' => env('BV_RETRIES', 3),
],
```

### Dependency Injection

```php
use BillionVerify\Client;

class UserController extends Controller
{
    public function __construct(
        private Client $client
    ) {}

    public function register(Request $request)
    {
        $result = $this->client->verify($request->email)['data'];

        if ($result['status'] === 'invalid') {
            return back()->withErrors(['email' => 'Invalid email address']);
        }

        // Continue registration...
    }
}
```

### Validation Rule

A custom rule in your app (`php artisan make:rule ValidEmail`) that calls the client:

```php
<?php

namespace App\Rules;

use BillionVerify\Client;
use BillionVerify\Exception\BillionVerifyException;
use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ValidEmail implements ValidationRule
{
    public function __construct(
        private bool $rejectDisposable = true,
        private bool $rejectRole = false,
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        try {
            $result = app(Client::class)->verify($value)['data'];
        } catch (BillionVerifyException $e) {
            return; // Do not block users when the API is unavailable
        }

        if ($result['status'] === 'invalid') {
            $fail('The :attribute is not a valid email address.');
        } elseif ($this->rejectDisposable && $result['is_disposable']) {
            $fail('Disposable email addresses are not allowed.');
        } elseif ($this->rejectRole && $result['is_role']) {
            $fail('Role-based email addresses are not allowed.');
        }
    }
}
```

Use it in a controller:

```php
use App\Rules\ValidEmail;

$request->validate([
    'email' => ['required', 'email', new ValidEmail(rejectDisposable: true, rejectRole: false)],
]);
```

### Form Request

```php
<?php

namespace App\Http\Requests;

use App\Rules\ValidEmail;
use Illuminate\Foundation\Http\FormRequest;

class RegisterRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => [
                'required',
                'email',
                'unique:users',
                new ValidEmail(rejectDisposable: true),
            ],
            'password' => ['required', 'min:8', 'confirmed'],
        ];
    }
}
```

### Queued Verification

```php
<?php

namespace App\Jobs;

use BillionVerify\Client;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class VerifyUserEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private int $userId,
        private string $email
    ) {}

    public function handle(Client $client): void
    {
        $result = $client->verify($this->email)['data'];

        \App\Models\User::where('id', $this->userId)->update([
            'email_verified' => $result['status'] === 'valid',
            'email_score' => $result['score'],
            'email_disposable' => $result['is_disposable'],
        ]);
    }
}

// Dispatch
VerifyUserEmail::dispatch($user->id, $user->email);
```

### Middleware

```php
<?php

namespace App\Http\Middleware;

use BillionVerify\Client;
use Closure;
use Illuminate\Http\Request;

class VerifyEmailMiddleware
{
    public function __construct(
        private Client $client
    ) {}

    public function handle(Request $request, Closure $next)
    {
        if ($request->has('email')) {
            $result = $this->client->verify($request->email)['data'];

            if ($result['status'] === 'invalid') {
                return response()->json([
                    'error' => 'Invalid email address'
                ], 422);
            }

            if ($result['is_disposable']) {
                return response()->json([
                    'error' => 'Disposable emails are not allowed'
                ], 422);
            }
        }

        return $next($request);
    }
}
```

## Complete Examples

### Registration Validation

```php
<?php

use BillionVerify\Client;

class EmailValidator
{
    public function __construct(
        private Client $client
    ) {}

    public function validateForRegistration(string $email): array
    {
        $result = $this->client->verify($email)['data'];

        // Reject invalid emails
        if ($result['status'] === 'invalid') {
            return [
                'valid' => false,
                'reason' => 'invalid_email',
                'message' => 'This email address is invalid',
            ];
        }

        // Reject disposable emails
        if ($result['is_disposable']) {
            return [
                'valid' => false,
                'reason' => 'disposable',
                'message' => 'Please use a permanent email address',
            ];
        }

        // Warn about role-based emails
        if ($result['is_role']) {
            return [
                'valid' => true,
                'warning' => 'role_based',
                'message' => 'Using a personal email is recommended',
            ];
        }

        return [
            'valid' => true,
            'score' => $result['score'],
        ];
    }
}

// Usage
$validator = new EmailValidator($client);
$validation = $validator->validateForRegistration('user@example.com');

if (!$validation['valid']) {
    echo $validation['message'];
}
```

### Bulk List Cleaning

```php
<?php

use BillionVerify\Client;

class ListCleaner
{
    public function __construct(
        private Client $client
    ) {}

    /** Upload a CSV, wait for the job and save the result CSV. */
    public function cleanFile(string $inputFile, string $outputFile): array
    {
        $upload = $this->client->uploadFile($inputFile, emailColumn: 'email')['data'];

        $done = $this->client->waitForFileJobCompletion($upload['task_id'])['data'];

        if ($done['status'] === 'failed') {
            throw new \RuntimeException('File job failed');
        }

        file_put_contents($outputFile, $this->client->getFileJobResults($upload['task_id']));

        return [
            'total' => $done['total_emails'],
            'valid' => $done['valid_emails'],
            'invalid' => $done['invalid_emails'],
        ];
    }
}
```

### API Endpoint

```php
<?php
// api/verify.php

require __DIR__ . '/../vendor/autoload.php';

use BillionVerify\Client;

header('Content-Type: application/json');

$client = new Client(getenv('BV_API_KEY'));

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method not allowed']);
    exit;
}

$input = json_decode(file_get_contents('php://input'), true);
$email = $input['email'] ?? null;

if (!$email) {
    http_response_code(400);
    echo json_encode(['error' => 'Email is required']);
    exit;
}

try {
    $data = $client->verify($email)['data'];
    echo json_encode([
        'email' => $data['email'],
        'status' => $data['status'],
        'score' => $data['score'],
        'deliverable' => $data['is_deliverable'],
    ]);
} catch (\Exception $e) {
    http_response_code(500);
    echo json_encode(['error' => 'Verification failed']);
}
```

## Best Practices

### 1. Cache Results

```php
use Illuminate\Support\Facades\Cache;

function verifyWithCache(Client $client, string $email): array
{
    $cacheKey = 'email_verification_' . md5($email);

    return Cache::remember($cacheKey, 3600, function () use ($client, $email) {
        return $client->verify($email)['data'];
    });
}
```

### 2. Handle Errors Gracefully

```php
use BillionVerify\Exception\BillionVerifyException;
use BillionVerify\Exception\RateLimitException;
use Illuminate\Support\Facades\Log;

function safeVerify(Client $client, string $email): ?array
{
    try {
        return $client->verify($email)['data'];
    } catch (RateLimitException $e) {
        Log::warning('Rate limited', ['retry_after' => $e->getRetryAfter()]);
        return null;
    } catch (BillionVerifyException $e) {
        Log::error('Verification failed', ['error' => $e->getMessage()]);
        return null;
    }
}
```

## Next Steps

  - [API Reference](/docs/api/): Detailed API documentation
  - [Laravel Integration](https://billionverify.com/docs/integration-guides/web-frameworks/laravel): Advanced Laravel patterns