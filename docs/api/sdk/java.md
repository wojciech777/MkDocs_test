# Java SDK

`orion-sdk-java` version 4.2.1 is the official client library for Orion Platform 4.2 LTS. It requires Java 17 or newer and brings no dependencies on application frameworks; under the hood it uses `java.net.http.HttpClient`. The artifact is published on Maven Central and signed with the Nimbus Software GPG key.

## Installation

=== "Maven"

    ```xml title="pom.xml"
    <dependency>
        <groupId>com.nimbus.orion</groupId>
        <artifactId>orion-sdk-java</artifactId>
        <version>4.2.1</version>
    </dependency>
    ```

=== "Gradle"

    ```kotlin title="build.gradle.kts"
    dependencies {
        implementation("com.nimbus.orion:orion-sdk-java:4.2.1")
    }
    ```

## Initializing the client

The client is created through `OrionClientBuilder`. The choice of authentication method depends on whether the integration runs in a server environment with access to Keycloak (OAuth2) or uses an API key.

=== "API key"

    ```java title="OrionConfig.java"
    import com.nimbus.orion.OrionClient;
    import com.nimbus.orion.OrionClientBuilder;

    import java.time.Duration;

    public final class OrionConfig {

        public static OrionClient create() {
            return OrionClientBuilder.create()
                    .baseUrl("https://api.orion.example.com/v1")
                    .apiKey(System.getenv("ORION_API_KEY"))
                    .tenant("ten_01HQ8ZV3KXN4M2T9YB7C6D")
                    .connectTimeout(Duration.ofSeconds(3))
                    .readTimeout(Duration.ofSeconds(15))
                    .maxRetries(3)
                    .build();
        }
    }
    ```

=== "OAuth2 client credentials"

    ```java title="OrionConfig.java" hl_lines="9 10 11"
    import com.nimbus.orion.OrionClient;
    import com.nimbus.orion.OrionClientBuilder;

    import java.time.Duration;
    import java.util.List;

    public final class OrionConfig {

        public static OrionClient create() {
            return OrionClientBuilder.create()
                    .baseUrl("https://api.orion.example.com/v1")
                    .clientCredentials(
                            System.getenv("ORION_CLIENT_ID"),
                            System.getenv("ORION_CLIENT_SECRET"))
                    .scopes(List.of("orders:read", "orders:write", "payments:write"))
                    .tenant("ten_01HQ8ZV3KXN4M2T9YB7C6D")
                    .readTimeout(Duration.ofSeconds(15))
                    .build();
        }
    }
    ```

The token is fetched on the first request and refreshed 60 s before the 3600 s TTL elapses. For the test environment, set `baseUrl("https://api.sandbox.orion.example.com/v1")` and an `ok_test_...` key.

## Basic operations

### Creating a customer and an order

```java title="CheckoutService.java" hl_lines="24 25"
import com.nimbus.orion.OrionClient;
import com.nimbus.orion.model.Customer;
import com.nimbus.orion.model.CustomerCreateParams;
import com.nimbus.orion.model.Order;
import com.nimbus.orion.model.OrderCreateParams;
import com.nimbus.orion.model.OrderLineParams;

public class CheckoutService {

    private final OrionClient client;

    public CheckoutService(OrionClient client) {
        this.client = client;
    }

    public Order createOrder() {
        Customer customer = client.customers().create(
                CustomerCreateParams.builder()
                        .email("anna.kowalska@example.com")
                        .name("Anna Kowalska")
                        .phone("+48512340099")
                        .build());

        return client.orders().create(
                OrderCreateParams.builder()
                        .customerId(customer.getId())
                        .currency("PLN")
                        .addLine(OrderLineParams.builder()
                                .sku("NB-14-PRO")
                                .name("Notebook 14 Pro")
                                .quantity(1)
                                .unitAmount(24990L)
                                .build())
                        .addLine(OrderLineParams.builder()
                                .sku("ACC-MOUSE-01")
                                .name("Wireless mouse")
                                .quantity(2)
                                .unitAmount(12900L)
                                .build())
                        .putMetadata("channel", "web")
                        .build());
    }
}
```

Amounts are passed as `long` values in the smallest unit of the currency: `24990` means 249.90 PLN.

### Capturing a payment

```java title="PaymentService.java"
import com.nimbus.orion.OrionClient;
import com.nimbus.orion.model.Payment;
import com.nimbus.orion.model.PaymentCaptureParams;

public class PaymentService {

    private final OrionClient client;

    public PaymentService(OrionClient client) {
        this.client = client;
    }

    public Payment capture(String paymentId, long amount) {
        Payment payment = client.payments().capture(
                paymentId,
                PaymentCaptureParams.builder()
                        .amount(amount)
                        .idempotencyKey("capture-" + paymentId)
                        .build());

        if (!"captured".equals(payment.getStatus())) {
            throw new IllegalStateException(
                    "Unexpected payment status: " + payment.getStatus());
        }
        return payment;
    }
}
```

### Iterating over result pages

`orders().list(...)` returns an `OrionPage<Order>` that implements `Iterable<Order>`. Iteration fetches subsequent pages automatically, using the cursor returned by the API.

```java title="OrderReport.java"
import com.nimbus.orion.OrionClient;
import com.nimbus.orion.model.Order;
import com.nimbus.orion.model.OrderListParams;

import java.time.Instant;

public class OrderReport {

    public long countPaid(OrionClient client, Instant since) {
        OrderListParams params = OrderListParams.builder()
                .status("paid")
                .createdAfter(since)
                .limit(100)
                .build();

        long total = 0;
        for (Order order : client.orders().list(params).autoPaging()) {
            total += order.getAmountTotal();
        }
        return total;
    }
}
```

!!! note "Page limit"
    The default `limit` is 25 and the maximum is 100. `autoPaging()` performs as many requests as needed; with wide time ranges it is worth narrowing the filters so as not to exhaust the limit described in [Rate limits](../rest/rate-limits.md).

## Error handling

All API response errors are signaled by an `OrionApiException` or one of its subclasses. The exception exposes the domain code and the request identifier.

```java title="ErrorHandlingExample.java" hl_lines="12 15"
import com.nimbus.orion.OrionApiException;
import com.nimbus.orion.OrionRateLimitException;
import com.nimbus.orion.OrionValidationException;

try {
    client.orders().create(params);
} catch (OrionValidationException e) {
    log.warn("Invalid input data: {} ({})", e.getMessage(), e.getCode());
} catch (OrionRateLimitException e) {
    log.warn("Rate limit exceeded, retry after {} s", e.getRetryAfterSeconds());
} catch (OrionApiException e) {
    log.error("API error {} status={} requestId={}",
            e.getCode(), e.getStatusCode(), e.getRequestId(), e);
    throw e;
}
```

| Code | HTTP status | Exception | Recommended reaction |
| --- | --- | --- | --- |
| `ORN-1004` | 400 | `OrionValidationException` | fix the payload, do not retry |
| `ORN-2001` | 401 | `OrionAuthException` | refresh the token or check the API key |
| `ORN-2003` | 403 | `OrionAuthException` | add the missing scope to the OAuth2 client |
| `ORN-3001` | 429 | `OrionRateLimitException` | retry after `getRetryAfterSeconds()` |
| `ORN-4002` | 409 | `OrionConflictException` | use a new `Idempotency-Key` or read the existing resource |
| `ORN-5002` | 503 | `OrionApiException` | retry with exponential backoff, report it if the error persists |

The `getRequestId()` value corresponds to the `X-Orion-Request-Id` header and is required for support tickets.

## Builder options

| Method | Default | Description |
| --- | --- | --- |
| `baseUrl(String)` | `https://api.orion.example.com/v1` | Base API address. |
| `apiKey(String)` | - | An `ok_live_...` or `ok_test_...` key, sent in `X-Orion-Key`. |
| `clientCredentials(String, String)` | - | OAuth2 client identifier and secret. |
| `scopes(List<String>)` | all assigned | Scopes requested when fetching the token. |
| `tenant(String)` | - | Value of the `X-Orion-Tenant` header. |
| `connectTimeout(Duration)` | 5 s | Connection handshake limit. |
| `readTimeout(Duration)` | 30 s | Limit for waiting on a response. |
| `maxRetries(int)` | 2 | Number of retries for `429` and `5xx`. |
| `retryBackoff(Duration)` | 200 ms | Base delay, multiplied exponentially with jitter. |
| `maxConnections(int)` | 32 | Size of the HTTP connection pool. |
| `idempotencyKeySupplier(Supplier<String>)` | UUID v4 | Generator of `Idempotency-Key` values. |
| `debug(boolean)` | `false` | Logging of requests and responses at the `DEBUG` level. |

## Webhook verification

The `WebhookVerifier` class implements the `v2` signature algorithm. In a Spring MVC controller the request body must be accepted as `byte[]` so that it is not modified by message converters.

```java title="OrionWebhookController.java" hl_lines="20 23 24"
import com.nimbus.orion.webhooks.OrionEvent;
import com.nimbus.orion.webhooks.WebhookVerifier;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OrionWebhookController {

    private final WebhookVerifier verifier;
    private final EventQueue queue;

    public OrionWebhookController(EventQueue queue) {
        this.queue = queue;
        this.verifier = WebhookVerifier.builder()
                .addSecret(System.getenv("ORION_WEBHOOK_SECRET"))
                .addSecret(System.getenv("ORION_WEBHOOK_SECRET_PREVIOUS"))
                .toleranceSeconds(300)
                .build();
    }

    @PostMapping(path = "/webhooks/orion", consumes = "application/json")
    public ResponseEntity<Void> receive(
            @RequestHeader("X-Orion-Signature") String signature,
            @RequestHeader("X-Orion-Event-Id") String eventId,
            @RequestBody byte[] rawBody) {

        if (!verifier.isValid(signature, rawBody)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        OrionEvent event = verifier.parse(rawBody);
        queue.enqueueIfAbsent(eventId, event);
        return ResponseEntity.accepted().build();
    }
}
```

Calling `addSecret` twice covers the overlap period after a secret rotation. A `null` value is skipped, so a missing `ORION_WEBHOOK_SECRET_PREVIOUS` variable does not cause an error.

## Threading and connection pools

!!! tip "One client per application"
    `OrionClient` is thread-safe and manages its own connection pool and token cache. Create it once, as a singleton or a Spring bean, and inject it into your services. Creating a client per request causes repeated token fetches, descriptor exhaustion and `ORN-3001` errors.

The client should be closed when the application shuts down, to release the pool threads:

```java title="OrionClientConfiguration.java"
@Bean(destroyMethod = "close")
public OrionClient orionClient() {
    return OrionConfig.create();
}
```

## Logging and debug mode

The SDK uses SLF4J. Enabling `debug(true)` logs the method, path, status, response time and `X-Orion-Request-Id`; request bodies are redacted, and the `Authorization` and `X-Orion-Key` headers as well as `whsec_...` values are never written.

```xml title="logback-spring.xml"
<logger name="com.nimbus.orion" level="DEBUG"/>
<logger name="com.nimbus.orion.http.wire" level="INFO"/>
```

!!! danger "Do not enable the wire log in production"
    The `com.nimbus.orion.http.wire` category at the `TRACE` level writes full request and response bodies, including customers' personal data. Use it only in a test environment with synthetic data.

## See also

- [Client libraries](index.md)
- [Python library](python.md)
- [Signature verification](../webhooks/signature-verification.md)
- [Error codes](../rest/error-codes.md)
