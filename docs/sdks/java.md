# Java SDK

Canonical page: https://billionverify.github.io/docs/sdks/java/

> The REST API is the source of truth. This SDK is a reference implementation, so if it differs from the [API reference](/docs/api/), follow the API.

**Source code:** [github.com/BillionVerify/billionverify-java](https://github.com/BillionVerify/billionverify-java)

The Java SDK is a small client built on OkHttp and Jackson. Responses are Java records. All methods are blocking and throw the checked `BillionVerifyException`. The SDK has no Spring Boot starter; the Spring Boot section below shows how to register the client as a bean yourself.

## Installation

The Java SDK is not published to a package registry. Build it from [source on GitHub](https://github.com/BillionVerify/billionverify-java), then add it to your project.

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

## Requirements

- Java 17 or higher
- Maven or Gradle

## Quick Start

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.model.VerifyResponse;

public class Main {
    public static void main(String[] args) throws Exception {
        try (BillionVerifyClient client = BillionVerifyClient.builder(System.getenv("BV_API_KEY")).build()) {
            VerifyResponse result = client.verify("user@example.com");
            System.out.println("Status: " + result.status());
            System.out.println("Score: " + result.score());
        }
    }
}
```

## Configuration

### Client Builder

```java
import com.billionverify.sdk.BillionVerifyClient;
import java.time.Duration;

BillionVerifyClient client = BillionVerifyClient.builder(System.getenv("BV_API_KEY"))
    .timeout(Duration.ofSeconds(30))
    .retries(3)
    .baseUrl("https://api.billionverify.com")
    .build();
```

`builder()` throws `IllegalArgumentException` if the API key is null or empty.

### Configuration Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `apiKey` | String | Required | Your API key (argument of `builder()`) |
| `timeout` | Duration | 30s | Connect, read and write timeout |
| `retries` | int | 3 | Maximum attempts for retryable errors |
| `baseUrl` | String | `https://api.billionverify.com` | API host, without `/v1`; the SDK adds `/v1` to every path |

## Types

### VerifyResponse

```java
public record VerifyResponse(
    String email,
    Status status,            // VALID, INVALID, UNKNOWN, RISKY, DISPOSABLE, CATCHALL, ROLE
    double score,
    boolean isDeliverable,
    boolean isDisposable,
    boolean isCatchall,
    boolean isRole,
    boolean isFree,
    String domain,
    Integer domainAge,
    List<String> mxRecords,
    DomainReputation domainReputation,
    boolean smtpCheck,
    String reason,
    String suggestion,
    Integer responseTime,
    int creditsUsed
) {}
```

### FileJobResponse

```java
public record FileJobResponse(
    String jobId,
    String status,            // "pending", "processing", "completed", "failed"
    String fileName,
    int totalEmails,
    int processedEmails,
    int progressPercent,
    int validEmails,
    int invalidEmails,
    int unknownEmails,
    int roleEmails,
    int catchallEmails,
    int disposableEmails,
    int creditsUsed,
    Double processTimeSeconds,
    String resultFilePath,
    String downloadUrl,
    String createdAt,
    String completedAt
) {}
```

## Single Verification

### Basic Verification

```java
VerifyResponse result = client.verify("user@example.com");

System.out.println("Email: " + result.email());
System.out.println("Status: " + result.status());
System.out.println("Score: " + result.score());
System.out.println("Deliverable: " + result.isDeliverable());
System.out.println("Disposable: " + result.isDisposable());
```

### Verification with Options

```java
// Second argument: checkSmtp (default: true)
VerifyResponse result = client.verify("user@example.com", true);
```

### Async Verification

The client is blocking and has no async methods. Wrap calls in a `CompletableFuture` and run them on your own executor:

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture<VerifyResponse> future = CompletableFuture.supplyAsync(() -> {
    try {
        return client.verify("user@example.com");
    } catch (BillionVerifyException e) {
        throw new RuntimeException(e);
    }
});

future.thenAccept(result -> {
    System.out.println("Status: " + result.status());
}).exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return null;
});
```

## Bulk Verification

### Verify a Batch Synchronously

`verifyBatch()` verifies up to 50 emails in one request and returns the results directly. With more than 50 emails it throws `ValidationException` before calling the API.

```java
import com.billionverify.sdk.model.BatchVerifyResponse;
import com.billionverify.sdk.model.VerificationResult;
import java.util.List;

List<String> emails = List.of(
    "user1@example.com",
    "user2@example.com",
    "user3@example.com"
);

BatchVerifyResponse batch = client.verifyBatch(emails); // optional second argument: checkSmtp

System.out.println("Total: " + batch.totalEmails());
System.out.println("Valid: " + batch.validEmails());
System.out.println("Invalid: " + batch.invalidEmails());
System.out.println("Credits used: " + batch.creditsUsed());

for (VerificationResult item : batch.results()) {
    System.out.println(item.email() + ": " + item.status());
}
```

### Upload a File for Larger Lists

For larger lists, upload a file. The job runs asynchronously.

```java
import com.billionverify.sdk.model.FileUploadResponse;
import java.io.File;

FileUploadResponse upload = client.uploadFile(new File("emails.csv"));

// Or with options: checkSmtp, emailColumn (auto-detected if null), preserveOriginal
FileUploadResponse upload2 = client.uploadFile(new File("emails.csv"), true, "email", true);

System.out.println("Task ID: " + upload.taskId());
```

### Check Status

```java
import com.billionverify.sdk.model.FileJobResponse;

FileJobResponse status = client.getFileJobStatus(upload.taskId());

System.out.println("Status: " + status.status());
System.out.println("Progress: " + status.processedEmails() + "/" + status.totalEmails());
System.out.println("Valid: " + status.validEmails());
System.out.println("Invalid: " + status.invalidEmails());
```

`getFileJobStatus(jobId, timeoutSeconds)` long-polls for up to that many seconds (0 to 300) instead of returning immediately. The client timeout applies to that request too, so keep the client `timeout` longer than the long-poll value. `waitForFileJobCompletion()` long-polls in 60-second steps, so build the client with a timeout above 60 seconds (for example `Duration.ofSeconds(90)`) before you use it.

### Get Results

```java
import com.billionverify.sdk.model.ResultFilters;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;

// Waits until the job is completed or failed (default maximum wait: 10 minutes)
FileJobResponse done = client.waitForFileJobCompletion(upload.taskId(), Duration.ofMinutes(30));

if ("completed".equals(done.status())) {
    // Download the result CSV. Filters are optional.
    ResultFilters filters = ResultFilters.builder().valid(true).build();
    byte[] csv = client.downloadFileResults(upload.taskId(), filters);
    Files.write(Path.of("results.csv"), csv);

    // Or only get the download URL
    String url = client.getFileResultsUrl(upload.taskId(), filters);
}
```

### With Webhook

Instead of polling, register a webhook. It is called when a file job finishes (`file.completed`) or fails (`file.failed`):

```java
import com.billionverify.sdk.model.Webhook;
import com.billionverify.sdk.model.WebhookEvent;

Webhook webhook = client.createWebhook(
    "https://your-domain.com/webhooks/billionverify",
    List.of(WebhookEvent.FILE_COMPLETED, WebhookEvent.FILE_FAILED)
);

System.out.println(webhook.id());
System.out.println(webhook.secret()); // Returned only when the webhook is created. Store it.

List<Webhook> webhooks = client.listWebhooks();
client.deleteWebhook(webhook.id());
```

Each delivery is a `POST` with a JSON body of the form `{event, timestamp, data}`. The API signs it with HMAC-SHA256 over `"<timestamp>.<raw body>"`, using the webhook secret, and sends these headers:

- `X-Webhook-Signature`: `sha256=<hex digest>`
- `X-Webhook-Timestamp`: Unix time in seconds, the value used in the signature
- `X-Webhook-Event`: the event name

Verify against the raw body before parsing it:

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.HexFormat;

static boolean isValidWebhook(String rawBody, String timestamp, String signatureHeader, String secret)
        throws Exception {
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
    byte[] hash = mac.doFinal((timestamp + "." + rawBody).getBytes(StandardCharsets.UTF_8));
    String expected = "sha256=" + HexFormat.of().formatHex(hash);
    return MessageDigest.isEqual(
        expected.getBytes(StandardCharsets.UTF_8),
        signatureHeader.getBytes(StandardCharsets.UTF_8));
}
```

> The SDK's `BillionVerifyClient.verifyWebhookSignature(payload, signature, secret)` signs the body only, without the timestamp prefix, so it does not match what the API sends. Use the method above for real deliveries.

## Credit Management

```java
import com.billionverify.sdk.model.CreditsResponse;

CreditsResponse credits = client.getCredits();

System.out.println("Balance: " + credits.creditsBalance());
System.out.println("Consumed: " + credits.creditsConsumed());
System.out.println("Added: " + credits.creditsAdded());
```

## Error Handling

### Exception Types

All exceptions are in `com.billionverify.sdk.exception`, extend the checked `BillionVerifyException`, and have `getErrorCode()`, `getStatusCode()` and `getDetails()`.

| Exception | HTTP status | Notes |
| --- | --- | --- |
| `AuthenticationException` | 401 | Missing or invalid API key |
| `InsufficientCreditsException` | 402 | Not enough credits |
| `NotFoundException` | 404 | For example an unknown job ID |
| `ValidationException` | 400 | Also thrown locally for more than 50 emails |
| `RateLimitException` | 429 | `getRetryAfter()` returns seconds |
| `TimeoutException` | 504 | Request timed out, or the file job wait ran out of time |

```java
import com.billionverify.sdk.exception.*;

try {
    VerifyResponse result = client.verify("user@example.com");
} catch (AuthenticationException e) {
    System.err.println("Invalid API key");
} catch (InsufficientCreditsException e) {
    System.err.println("Not enough credits");
} catch (RateLimitException e) {
    System.err.println("Rate limited. Retry after " + e.getRetryAfter() + " seconds");
} catch (ValidationException e) {
    System.err.println("Invalid input: " + e.getMessage());
} catch (BillionVerifyException e) {
    System.err.println("API error: " + e.getMessage());
}
```

The SDK retries 429, 500, 502 and 503 responses with exponential backoff before it throws. `retries` is the maximum number of attempts, including the first one.

### Custom Exception Handler

```java
public class VerificationService {
    private final BillionVerifyClient client;

    public Optional<VerifyResponse> safeVerify(String email) {
        try {
            return Optional.of(client.verify(email));
        } catch (RateLimitException e) {
            logger.warn("Rate limited, retry after: {}", e.getRetryAfter());
            return Optional.empty();
        } catch (BillionVerifyException e) {
            logger.error("Verification failed: {}", e.getMessage());
            return Optional.empty();
        }
    }
}
```

## Spring Boot Integration

The SDK ships no Spring Boot starter or auto-configuration. Register the client as a bean in your own configuration class.

### Dependency

Build and install the SDK as described above, then add the `io.github.billionverifier:billionverify-java:1.0.0` dependency to your project. No other BillionVerify artifact is needed.

### Configuration

```yaml
# application.yml
billionverify:
  api-key: ${BV_API_KEY}
  timeout-seconds: 30
  retries: 3
```

Or in `application.properties`:

```properties
billionverify.api-key=${BV_API_KEY}
billionverify.timeout-seconds=30
billionverify.retries=3
```

### Auto-Configuration

Define the bean yourself. `close()` is called on shutdown because the client implements `AutoCloseable`:

```java
import com.billionverify.sdk.BillionVerifyClient;
import java.time.Duration;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BillionVerifyConfig {

    @Bean(destroyMethod = "close")
    public BillionVerifyClient billionVerifyClient(
            @Value("${billionverify.api-key}") String apiKey,
            @Value("${billionverify.timeout-seconds:30}") long timeoutSeconds,
            @Value("${billionverify.retries:3}") int retries) {
        return BillionVerifyClient.builder(apiKey)
            .timeout(Duration.ofSeconds(timeoutSeconds))
            .retries(retries)
            .build();
    }
}
```

Then inject it:

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.exception.BillionVerifyException;
import com.billionverify.sdk.model.Status;
import com.billionverify.sdk.model.VerifyResponse;
import org.springframework.stereotype.Service;

@Service
public class EmailVerificationService {
    private final BillionVerifyClient client;

    public EmailVerificationService(BillionVerifyClient client) {
        this.client = client;
    }

    public boolean isValidEmail(String email) throws BillionVerifyException {
        VerifyResponse result = client.verify(email);
        return result.status() == Status.VALID;
    }
}
```

### REST Controller

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.model.VerifyResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/verify")
public class VerificationController {
    private final BillionVerifyClient client;

    public VerificationController(BillionVerifyClient client) {
        this.client = client;
    }

    @PostMapping
    public ResponseEntity<?> verify(@RequestBody VerifyRequest request) {
        try {
            VerifyResponse result = client.verify(request.email());
            return ResponseEntity.ok(new VerifyReply(
                result.email(),
                result.status().toString(),
                result.score()
            ));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(new ErrorResponse("Verification failed"));
        }
    }
}

record VerifyRequest(String email) {}
record VerifyReply(String email, String status, double score) {}
record ErrorResponse(String error) {}
```

### Custom Validator Annotation

This uses Jakarta Bean Validation (Spring Boot 3). On Spring Boot 2, import `javax.validation` instead.

```java
// Annotation
import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidEmailValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidEmail {
    String message() default "Invalid email address";
    boolean rejectDisposable() default true;
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.model.Status;
import com.billionverify.sdk.model.VerifyResponse;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class ValidEmailValidator implements ConstraintValidator<ValidEmail, String> {
    private final BillionVerifyClient client;
    private boolean rejectDisposable;

    public ValidEmailValidator(BillionVerifyClient client) {
        this.client = client;
    }

    @Override
    public void initialize(ValidEmail annotation) {
        this.rejectDisposable = annotation.rejectDisposable();
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return false;
        }

        try {
            VerifyResponse result = client.verify(email);

            if (result.status() == Status.INVALID) {
                return false;
            }

            return !(rejectDisposable && result.isDisposable());
        } catch (Exception e) {
            return true; // Do not block users when the API is unavailable
        }
    }
}

// Usage
public class RegisterRequest {
    @ValidEmail(rejectDisposable = true)
    private String email;

    // getters, setters...
}
```

### Async Service

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.model.VerifyResponse;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;

@Service
public class AsyncVerificationService {
    private final BillionVerifyClient client;

    public AsyncVerificationService(BillionVerifyClient client) {
        this.client = client;
    }

    @Async
    public CompletableFuture<VerifyResponse> verifyAsync(String email) {
        try {
            return CompletableFuture.completedFuture(client.verify(email));
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

`@Async` needs `@EnableAsync` on a configuration class.

## Complete Examples

### Registration Service

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.exception.BillionVerifyException;
import com.billionverify.sdk.model.Status;
import com.billionverify.sdk.model.VerifyResponse;

public class RegistrationService {
    private final BillionVerifyClient client;

    public RegistrationService(BillionVerifyClient client) {
        this.client = client;
    }

    public ValidationResult validateEmail(String email) throws BillionVerifyException {
        VerifyResponse result = client.verify(email);

        // Reject invalid emails
        if (result.status() == Status.INVALID) {
            return ValidationResult.invalid("invalid_email",
                "This email address is invalid");
        }

        // Reject disposable emails
        if (result.isDisposable()) {
            return ValidationResult.invalid("disposable",
                "Please use a permanent email address");
        }

        // Warn about role-based emails
        if (result.isRole()) {
            return ValidationResult.warning("role_based",
                "Using a personal email is recommended",
                result.score());
        }

        return ValidationResult.valid(result.score());
    }
}

class ValidationResult {
    private final boolean valid;
    private final String reason;
    private final String message;
    private final Double score;

    private ValidationResult(boolean valid, String reason, String message, Double score) {
        this.valid = valid;
        this.reason = reason;
        this.message = message;
        this.score = score;
    }

    public static ValidationResult valid(double score) {
        return new ValidationResult(true, null, null, score);
    }

    public static ValidationResult invalid(String reason, String message) {
        return new ValidationResult(false, reason, message, null);
    }

    public static ValidationResult warning(String reason, String message, double score) {
        return new ValidationResult(true, reason, message, score);
    }

    // Getters...
}
```

### Concurrent Verification

```java
import com.billionverify.sdk.BillionVerifyClient;
import com.billionverify.sdk.exception.BillionVerifyException;
import com.billionverify.sdk.model.VerifyResponse;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class ConcurrentVerifier {
    private final BillionVerifyClient client;
    private final ExecutorService executor;

    public ConcurrentVerifier(BillionVerifyClient client, int threads) {
        this.client = client;
        this.executor = Executors.newFixedThreadPool(threads);
    }

    public List<VerifyResponse> verifyAll(List<String> emails) {
        List<CompletableFuture<VerifyResponse>> futures = emails.stream()
            .map(email -> CompletableFuture.supplyAsync(() -> {
                try {
                    return client.verify(email);
                } catch (BillionVerifyException e) {
                    throw new RuntimeException(e);
                }
            }, executor))
            .collect(Collectors.toList());

        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }

    public void shutdown() {
        executor.shutdown();
    }
}

// Usage
ConcurrentVerifier verifier = new ConcurrentVerifier(client, 10);
try {
    List<String> emails = Arrays.asList("user1@example.com", "user2@example.com");
    List<VerifyResponse> results = verifier.verifyAll(emails);
    results.forEach(r -> System.out.println(r.email() + ": " + r.status()));
} finally {
    verifier.shutdown();
}
```

## Best Practices

### 1. Reuse One Client

```java
// Create the client once and share it. It wraps one OkHttpClient,
// which keeps a connection pool for you.
BillionVerifyClient client = BillionVerifyClient.builder(apiKey).build();
```

### 2. Handle Nullable Fields

```java
VerifyResponse result = client.verify(email);

// domainAge, mxRecords, reason and suggestion can be null
Optional.ofNullable(result.suggestion())
    .ifPresent(suggestion -> System.out.println("Did you mean: " + suggestion));
```

### 3. Resource Management

```java
// The client implements AutoCloseable
try (BillionVerifyClient client = BillionVerifyClient.builder(apiKey).build()) {
    VerifyResponse result = client.verify(email);
    // Process result
}
```

## Next Steps

  - [API Reference](/docs/api/): Detailed API documentation
  - [Integration Guides](https://billionverify.com/docs/integration-guides): Platform integrations