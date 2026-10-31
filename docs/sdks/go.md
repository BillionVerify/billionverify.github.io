# Go SDK

Canonical page: https://billionverify.github.io/docs/sdks/go/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-go](https://github.com/BillionVerify/billionverify-go)

The Go SDK is a small client built on the standard library only (`net/http`). It requires Go 1.21 or later, and every call takes a `context.Context`. The client is safe to share between goroutines.

## Installation

```bash
go get github.com/BillionVerify/billionverify-go
```

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "os"

    billionverify "github.com/BillionVerify/billionverify-go"
)

func main() {
    client, err := billionverify.NewClient(billionverify.Config{
        APIKey: os.Getenv("BV_API_KEY"),
    })
    if err != nil {
        panic(err)
    }

    ctx := context.Background()
    result, err := client.Verify(ctx, "user@example.com", nil)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Status: %s\n", result.Status)
    fmt.Printf("Score: %.2f\n", result.Score)
}
```

## Configuration

### Client Options

```go
client, err := billionverify.NewClient(billionverify.Config{
    APIKey:  os.Getenv("BV_API_KEY"),
    Timeout: 30 * time.Second,
    Retries: 3,
    BaseURL: "https://api.billionverify.com",
})
if err != nil {
    panic(err)
}
```

`NewClient` returns an `*AuthenticationError` if `APIKey` is empty.

### Config Fields

```go
type Config struct {
    APIKey  string        // Required: Your BillionVerify API key
    BaseURL string        // Optional: API host, without /v1 (default: https://api.billionverify.com)
    Timeout time.Duration // Optional: HTTP client timeout (default: 30s)
    Retries int           // Optional: Maximum attempts for retryable errors (default: 3)
}
```

The SDK adds `/v1` to every path itself, so `BaseURL` is the host only. A zero value for `Timeout` or `Retries` means the default.

## Types

### Verification Result

```go
type VerifyResponse struct {
    Email            string             `json:"email"`
    Status           VerificationStatus `json:"status"` // StatusValid, StatusInvalid, StatusUnknown, StatusRisky, StatusDisposable, StatusCatchall, StatusRole
    Score            float64            `json:"score"`
    IsDeliverable    bool               `json:"is_deliverable"`
    IsDisposable     bool               `json:"is_disposable"`
    IsCatchall       bool               `json:"is_catchall"`
    IsRole           bool               `json:"is_role"`
    IsFree           bool               `json:"is_free"`
    Domain           string             `json:"domain,omitempty"`
    DomainAge        int                `json:"domain_age,omitempty"`
    MXRecords        []string           `json:"mx_records,omitempty"`
    DomainReputation *DomainReputation  `json:"domain_reputation,omitempty"`
    SMTPCheck        bool               `json:"smtp_check"`
    Reason           string             `json:"reason,omitempty"`
    Suggestion       string             `json:"suggestion,omitempty"`
    ResponseTime     int                `json:"response_time,omitempty"`
    CreditsUsed      int                `json:"credits_used"`
}
```

### File Job

```go
type FileJobResponse struct {
    JobID              string    `json:"job_id"`
    Status             JobStatus `json:"status"` // JobStatusPending, JobStatusProcessing, JobStatusCompleted, JobStatusFailed
    FileName           string    `json:"file_name,omitempty"`
    TotalEmails        int       `json:"total_emails"`
    ProcessedEmails    int       `json:"processed_emails"`
    ProgressPercent    int       `json:"progress_percent"`
    ValidEmails        int       `json:"valid_emails"`
    InvalidEmails      int       `json:"invalid_emails"`
    UnknownEmails      int       `json:"unknown_emails"`
    RoleEmails         int       `json:"role_emails"`
    CatchallEmails     int       `json:"catchall_emails"`
    DisposableEmails   int       `json:"disposable_emails"`
    CreditsUsed        int       `json:"credits_used"`
    ProcessTimeSeconds float64   `json:"process_time_seconds,omitempty"`
    ResultFilePath     string    `json:"result_file_path,omitempty"`
    DownloadURL        string    `json:"download_url,omitempty"`
    CreatedAt          string    `json:"created_at"`
    CompletedAt        string    `json:"completed_at,omitempty"`
}
```

## Single Verification

### Basic Verification

Pass `nil` for the options to use the defaults (SMTP check on).

```go
ctx := context.Background()
result, err := client.Verify(ctx, "user@example.com", nil)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Email: %s\n", result.Email)
fmt.Printf("Status: %s\n", result.Status)
fmt.Printf("Score: %.2f\n", result.Score)
fmt.Printf("Disposable: %v\n", result.IsDisposable)
```

### With Options

`VerifyOptions.CheckSMTP` is a plain `bool`. When you pass an options struct, set it explicitly, because the zero value turns the SMTP check off.

```go
result, err := client.Verify(ctx, "user@example.com", &billionverify.VerifyOptions{
    CheckSMTP: true,
})
```

### Context with Timeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

result, err := client.Verify(ctx, "user@example.com", nil)
if err != nil {
    if ctx.Err() == context.DeadlineExceeded {
        log.Println("Verification timed out")
    }
    log.Fatal(err)
}
```

## Bulk Verification

### Verify a Batch Synchronously

`VerifyBatch` verifies up to 50 emails in one request and returns the results directly. With more than 50 emails it returns a `*ValidationError` without calling the API.

```go
emails := []string{
    "user1@example.com",
    "user2@example.com",
    "user3@example.com",
}

batch, err := client.VerifyBatch(ctx, emails, nil)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Total: %d, valid: %d, invalid: %d\n", batch.TotalEmails, batch.ValidEmails, batch.InvalidEmails)
for _, item := range batch.Results {
    fmt.Printf("%s: %s\n", item.Email, item.Status)
}
```

### Upload a File for Larger Lists

For larger lists, upload a file. The job runs asynchronously. `UploadFile` takes a path on disk; `UploadFileReader` takes an `io.Reader` and a file name.

```go
upload, err := client.UploadFile(ctx, "emails.csv", &billionverify.FileUploadOptions{
    CheckSMTP:        true,
    EmailColumn:      "email",
    PreserveOriginal: true,
})
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Task ID: %s\n", upload.TaskID)
```

### Check Status

```go
status, err := client.GetFileJobStatus(ctx, upload.TaskID, nil)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Status: %s\n", status.Status)
fmt.Printf("Progress: %d/%d\n", status.ProcessedEmails, status.TotalEmails)
```

To long-poll instead of returning immediately, pass `&billionverify.FileJobStatusOptions{Timeout: 30}` (seconds, capped at 300). The client `Timeout` applies to that request too, so keep it longer than the long-poll value.

### Get Results

```go
// Poll every 5 s, for up to 10 min (zero values mean these defaults)
status, err := client.WaitForFileJobCompletion(ctx, upload.TaskID, 5*time.Second, 10*time.Minute)
if err != nil {
    log.Fatal(err)
}

if status.Status == billionverify.JobStatusCompleted {
    // Returns the result CSV as bytes. Filters are optional *bool fields.
    valid := true
    csv, err := client.GetFileJobResults(ctx, upload.TaskID, &billionverify.FileResultsOptions{Valid: &valid})
    if err != nil {
        log.Fatal(err)
    }
    os.WriteFile("results.csv", csv, 0o644)
}
```

### With Webhook

Instead of polling, register a webhook. It is called when a file job finishes (`file.completed`) or fails (`file.failed`):

```go
webhook, err := client.CreateWebhook(ctx, billionverify.WebhookConfig{
    URL:    "https://your-domain.com/webhooks/billionverify",
    Events: []billionverify.WebhookEvent{billionverify.EventFileCompleted, billionverify.EventFileFailed},
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(webhook.ID)
fmt.Println(webhook.Secret) // Returned only when the webhook is created. Store it.

list, _ := client.ListWebhooks(ctx) // list.Webhooks, list.Total
_ = client.DeleteWebhook(ctx, webhook.ID)
```

Each delivery is a `POST` with a JSON body of the form `{event, timestamp, data}`. The API signs it with HMAC-SHA256 over `"<timestamp>.<raw body>"`, using the webhook secret, and sends these headers:

- `X-Webhook-Signature`: `sha256=<hex digest>`
- `X-Webhook-Timestamp`: Unix time in seconds, the value used in the signature
- `X-Webhook-Event`: the event name

Verify against the raw body before parsing it:

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"
)

func webhookHandler(secret string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)

        mac := hmac.New(sha256.New, []byte(secret))
        mac.Write([]byte(r.Header.Get("X-Webhook-Timestamp") + "."))
        mac.Write(body)
        expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

        if !hmac.Equal([]byte(r.Header.Get("X-Webhook-Signature")), []byte(expected)) {
            http.Error(w, "invalid signature", http.StatusUnauthorized)
            return
        }

        // Parse body as billionverify.WebhookPayload and handle the event.
        w.WriteHeader(http.StatusOK)
    }
}
```

> The SDK's `billionverify.VerifyWebhookSignature(payload, signature, secret)` signs the body only, without the timestamp prefix, so it does not match what the API sends. Use the handler above for real deliveries.

## Concurrent Verification

### Using Goroutines

```go
func verifyEmails(ctx context.Context, client *billionverify.Client, emails []string) []*billionverify.VerifyResponse {
    results := make([]*billionverify.VerifyResponse, len(emails))
    var wg sync.WaitGroup

    for i, email := range emails {
        wg.Add(1)
        go func(idx int, email string) {
            defer wg.Done()
            result, err := client.Verify(ctx, email, nil)
            if err != nil {
                log.Printf("Error verifying %s: %v", email, err)
                return
            }
            results[idx] = result
        }(i, email)
    }

    wg.Wait()
    return results
}
```

Failed emails leave a `nil` entry in the slice.

### With Worker Pool

```go
func verifyWithWorkerPool(ctx context.Context, client *billionverify.Client, emails []string, workers int) <-chan *billionverify.VerifyResponse {
    emailChan := make(chan string, len(emails))
    resultChan := make(chan *billionverify.VerifyResponse, len(emails))

    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for email := range emailChan {
                result, err := client.Verify(ctx, email, nil)
                if err != nil {
                    log.Printf("Error: %v", err)
                    continue
                }
                resultChan <- result
            }
        }()
    }

    // Send emails
    go func() {
        for _, email := range emails {
            emailChan <- email
        }
        close(emailChan)
    }()

    // Close results when done
    go func() {
        wg.Wait()
        close(resultChan)
    }()

    return resultChan
}

// Usage
results := verifyWithWorkerPool(ctx, client, emails, 10)
for result := range results {
    fmt.Printf("%s: %s\n", result.Email, result.Status)
}
```

### Using errgroup

`errgroup` comes from `golang.org/x/sync`, which is a separate module (`go get golang.org/x/sync`).

```go
import "golang.org/x/sync/errgroup"

func verifyEmailsConcurrently(ctx context.Context, client *billionverify.Client, emails []string) ([]*billionverify.VerifyResponse, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]*billionverify.VerifyResponse, len(emails))

    for i, email := range emails {
        i, email := i, email // capture loop variables
        g.Go(func() error {
            result, err := client.Verify(ctx, email, nil)
            if err != nil {
                return fmt.Errorf("verify %s: %w", email, err)
            }
            results[i] = result
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return results, nil
}
```

## Error Handling

### Error Types

The error types live in the main package. Each one embeds `BillionVerifyError` (fields `Code`, `Message`, `StatusCode`, `Details`), and they are returned as pointers.

| Type | HTTP status | Notes |
| --- | --- | --- |
| `*AuthenticationError` | 401 | Missing or invalid API key |
| `*InsufficientCreditsError` | 402 | Not enough credits |
| `*NotFoundError` | 404 | For example an unknown job ID |
| `*ValidationError` | 400 | Also returned locally for more than 50 emails |
| `*RateLimitError` | 429 | Has `RetryAfter` in seconds (`int`) |
| `*TimeoutError` | 504 | Context deadline exceeded, or `WaitForFileJobCompletion` ran out of time |
| `*BillionVerifyError` | other | Anything else, including network errors |

```go
result, err := client.Verify(ctx, email, nil)
if err != nil {
    switch e := err.(type) {
    case *billionverify.AuthenticationError:
        log.Fatal("Invalid API key")
    case *billionverify.InsufficientCreditsError:
        log.Print("Not enough credits")
    case *billionverify.RateLimitError:
        log.Printf("Rate limited. Retry after %d seconds", e.RetryAfter)
    case *billionverify.ValidationError:
        log.Printf("Invalid input: %s", e.Message)
    case *billionverify.BillionVerifyError:
        log.Printf("API error: %s (code: %s)", e.Message, e.Code)
    default:
        log.Printf("Unknown error: %v", err)
    }
}
```

The SDK already retries 429, 500, 502 and 503 responses with exponential backoff (`2^attempt` seconds, or the `Retry-After` header on 429) before it returns the error. `Retries` is the maximum number of attempts, including the first one.

### Custom Error Handler

```go
func verifyWithRetry(ctx context.Context, client *billionverify.Client, email string) (*billionverify.VerifyResponse, error) {
    var lastErr error

    for attempt := 0; attempt < 3; attempt++ {
        result, err := client.Verify(ctx, email, nil)
        if err == nil {
            return result, nil
        }

        lastErr = err

        // Wait as long as the API asks, then try again
        if rateLimitErr, ok := err.(*billionverify.RateLimitError); ok {
            time.Sleep(time.Duration(rateLimitErr.RetryAfter) * time.Second)
            continue
        }

        // Don't retry validation errors
        if _, ok := err.(*billionverify.ValidationError); ok {
            return nil, err
        }

        // Exponential backoff for other errors
        time.Sleep(time.Duration(attempt+1) * time.Second)
    }

    return nil, lastErr
}
```

## Credit Management

```go
credits, err := client.GetCredits(ctx)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Balance: %d\n", credits.CreditsBalance)
fmt.Printf("Consumed: %d\n", credits.CreditsConsumed)
fmt.Printf("Added: %d\n", credits.CreditsAdded)
```

## Framework Integration

### Gin

```go
import "github.com/gin-gonic/gin"

func setupRouter(client *billionverify.Client) *gin.Engine {
    r := gin.Default()

    r.POST("/verify", func(c *gin.Context) {
        var req struct {
            Email string `json:"email" binding:"required,email"`
        }

        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(400, gin.H{"error": err.Error()})
            return
        }

        result, err := client.Verify(c.Request.Context(), req.Email, nil)
        if err != nil {
            c.JSON(500, gin.H{"error": "Verification failed"})
            return
        }

        c.JSON(200, result)
    })

    return r
}
```

### Echo

```go
import "github.com/labstack/echo/v4"

func setupRoutes(e *echo.Echo, client *billionverify.Client) {
    e.POST("/verify", func(c echo.Context) error {
        var req struct {
            Email string `json:"email"`
        }

        if err := c.Bind(&req); err != nil {
            return c.JSON(400, map[string]string{"error": "Invalid request"})
        }

        result, err := client.Verify(c.Request().Context(), req.Email, nil)
        if err != nil {
            return c.JSON(500, map[string]string{"error": "Verification failed"})
        }

        return c.JSON(200, result)
    })
}
```

### Fiber

```go
import "github.com/gofiber/fiber/v2"

func setupFiber(client *billionverify.Client) *fiber.App {
    app := fiber.New()

    app.Post("/verify", func(c *fiber.Ctx) error {
        var req struct {
            Email string `json:"email"`
        }

        if err := c.BodyParser(&req); err != nil {
            return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
        }

        result, err := client.Verify(context.Background(), req.Email, nil)
        if err != nil {
            return c.Status(500).JSON(fiber.Map{"error": "Verification failed"})
        }

        return c.JSON(result)
    })

    return app
}
```

## Complete Example

### Email Validation Service

```go
package main

import (
    "context"
    "log"
    "os"

    billionverify "github.com/BillionVerify/billionverify-go"
)

type ValidationResult struct {
    Valid   bool    `json:"valid"`
    Reason  string  `json:"reason,omitempty"`
    Message string  `json:"message,omitempty"`
    Score   float64 `json:"score,omitempty"`
}

type EmailValidator struct {
    client *billionverify.Client
}

func NewEmailValidator() (*EmailValidator, error) {
    client, err := billionverify.NewClient(billionverify.Config{
        APIKey: os.Getenv("BV_API_KEY"),
    })
    if err != nil {
        return nil, err
    }
    return &EmailValidator{client: client}, nil
}

func (v *EmailValidator) ValidateForRegistration(ctx context.Context, email string) ValidationResult {
    result, err := v.client.Verify(ctx, email, nil)
    if err != nil {
        return ValidationResult{
            Valid:   false,
            Reason:  "verification_error",
            Message: "Could not verify email address",
        }
    }

    // Reject invalid emails
    if result.Status == billionverify.StatusInvalid {
        return ValidationResult{
            Valid:   false,
            Reason:  "invalid_email",
            Message: "This email address is invalid",
        }
    }

    // Reject disposable emails
    if result.IsDisposable {
        return ValidationResult{
            Valid:   false,
            Reason:  "disposable",
            Message: "Please use a permanent email address",
        }
    }

    // Warn about role-based emails but allow
    if result.IsRole {
        return ValidationResult{
            Valid:   true,
            Reason:  "role_based",
            Message: "Using a personal email is recommended",
            Score:   result.Score,
        }
    }

    return ValidationResult{
        Valid: true,
        Score: result.Score,
    }
}

func main() {
    validator, err := NewEmailValidator()
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()
    result := validator.ValidateForRegistration(ctx, "user@example.com")

    if !result.Valid {
        log.Printf("Invalid: %s - %s", result.Reason, result.Message)
        return
    }

    log.Printf("Valid email with score: %.2f", result.Score)
}
```

## Best Practices

### 1. Always Use Context

```go
// Set timeout for all operations
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

result, err := client.Verify(ctx, email, nil)
```

### 2. Check Errors Before Using Results

```go
result, err := client.Verify(ctx, email, nil)
if err != nil {
    return err
}
// result is never nil when err is nil
fmt.Printf("Deliverable: %v\n", result.IsDeliverable)
```

### 3. Use Connection Pooling

```go
// Create the client once and reuse it. It wraps one http.Client.
var client *billionverify.Client

func init() {
    var err error
    client, err = billionverify.NewClient(billionverify.Config{
        APIKey: os.Getenv("BV_API_KEY"),
    })
    if err != nil {
        panic(err)
    }
}
```

## Next Steps

  - [API Reference](/docs/api/): Detailed API documentation
  - [Integration Guides](https://billionverify.com/docs/integration-guides): Platform integrations