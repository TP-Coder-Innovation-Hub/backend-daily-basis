# Retry — Exponential Backoff with Resilience4j

## Why Retry

Network calls fail transiently: timeouts, temporary DNS issues, overloaded servers. Retrying with backoff gives the system time to recover. Not all errors should be retried — business logic errors (404, validation) will fail again.

## When to Retry vs Fail Fast

| Retry | Fail Fast |
|-------|-----------|
| Connection timeout | 4xx client errors |
| 5xx server errors (transient) | Business validation errors |
| Socket timeout | Not found (404) |
| Rate limited (429 with Retry-After) | Unauthorized (401) |

## Step 1: Configuration

```yaml
resilience4j:
  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.BusinessException
          - com.example.ValidationException
    instances:
      productService:
        base-config: default
        max-attempts: 3
        wait-duration: 500ms
      paymentService:
        max-attempts: 5
        wait-duration: 1s
```

## Step 2: Apply to Methods

```java
@Service
@RequiredArgsConstructor
public class ExternalApiService {
    private final RestClient restClient;

    @Retry(name = "productService", fallbackMethod = "productFallback")
    public ProductResponse getProduct(Long id) {
        return restClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .body(ProductResponse.class);
    }

    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return restClient.post()
            .uri("/api/payments")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }

    private ProductResponse productFallback(Long id, Throwable t) {
        log.warn("Product service unavailable after retries: {}",
            t.getMessage());
        return new ProductResponse(id, "Temporarily Unavailable",
            BigDecimal.ZERO, 0);
    }

    private PaymentResult paymentFallback(
            PaymentRequest request, Throwable t) {
        log.error("Payment processing failed after retries");
        return PaymentResult.failed(request.transactionId(),
            "Payment service unavailable. Please try again later.");
    }
}
```

## Step 3: Exponential Backoff with Jitter

The `wait-duration` is the base. Resilience4j uses exponential backoff by default: each retry waits longer. Add jitter to prevent the thundering herd problem (all clients retrying simultaneously):

```java
@Configuration
public class RetryConfig {
    @Bean
    public RetryCustomizer retryCustomizer() {
        return (id, builder) -> {
            IntervalFunction intervalWithJitter =
                IntervalFunction.ofExponentialBackoff(
                    Duration.ofSeconds(1), 2.0,
                    Duration.ofSeconds(30));
            builder.waitDuration(Duration.ofSeconds(1))
                .intervalFunction(intervalWithJitter);
            return builder;
        };
    }
}
```

Backoff sequence with jitter: ~1s, ~2s, ~4s (capped at 30s).

## Step 4: Retry with Circuit Breaker

Combine retry and circuit breaker. Retries handle transient failures; the circuit breaker handles sustained failures:

```java
@CircuitBreaker(name = "productService")
@Retry(name = "productService", fallbackMethod = "productFallback")
public ProductResponse getProduct(Long id) {
    return restClient.get()
        .uri("/api/products/{id}", id)
        .retrieve()
        .body(ProductResponse.class);
}
```

Execution order: Circuit Breaker wraps Retry. If retries exhaust, the circuit breaker counts it as a failure. After enough failures, the circuit opens.

## Step 5: Retry Events for Observability

```java
@Configuration
@RequiredArgsConstructor
@Slf4j
public class RetryEventConfig {
    @PostConstruct
    public void registerEventConsumers() {
        var registry = RetryRegistry.ofDefaults();
        registry.getEventPublisher()
            .onRetry(event -> log.warn(
                "Retry #{} for '{}': {}",
                event.getNumberOfRetryAttempts(),
                event.getName(),
                event.getLastThrowable().getMessage()))
            .onError(event -> log.error(
                "All retries exhausted for '{}'",
                event.getName()))
            .onSuccess(event -> log.debug(
                "Call succeeded for '{}' after {} attempts",
                event.getName(),
                event.getNumberOfRetryAttempts()));
    }
}
```

## Key Points

- Only retry transient errors — business errors will fail again
- Use exponential backoff with jitter to avoid thundering herd
- Always set a max-attempts limit — infinite retries will hang your service
- Combine retry with circuit breaker: retry handles brief hiccups, circuit breaker handles outages
- Log retry events to detect services that are frequently failing and recovering
