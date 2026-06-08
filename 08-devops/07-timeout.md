# Timeout — Every External Call Needs One

## Why Timeouts Matter

Without a timeout, a slow or unresponsive service blocks your thread indefinitely. One slow database call can consume all your thread pool, making your entire service unresponsive.

Every external call — HTTP, database, message queue — must have a timeout.

## Connection vs Read Timeout

| Timeout | Meaning |
|---------|---------|
| Connection timeout | Max time to establish the TCP connection |
| Read timeout | Max time waiting for a response after connection |
| Total timeout | Max time for the entire operation |

## Step 1: Resilience4j TimeLimiter

```yaml
resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 3s
        cancel-running-future: true
    instances:
      productService:
        timeout-duration: 2s
      paymentService:
        timeout-duration: 5s
      reportService:
        timeout-duration: 30s
```

```java
@Service
@RequiredArgsConstructor
public class ExternalService {
    private final RestClient restClient;

    @TimeLimiter(name = "productService")
    @CircuitBreaker(name = "productService")
    @Retry(name = "productService")
    public CompletableFuture<ProductResponse> getProduct(Long id) {
        return CompletableFuture.supplyAsync(() ->
            restClient.get()
                .uri("/api/products/{id}", id)
                .retrieve()
                .body(ProductResponse.class));
    }
}
```

## Step 2: Spring WebClient Timeout

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient productWebClient() {
        var httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
            .responseTimeout(Duration.ofSeconds(3))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(3))
                .addHandlerLast(new WriteTimeoutHandler(3)));

        return WebClient.builder()
            .baseUrl("http://product-service:8081")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductClient {
    private final WebClient productWebClient;

    public Mono<ProductResponse> getProduct(Long id) {
        return productWebClient.get()
            .uri("/api/products/{id}", id)
            .retrieve()
            .bodyToMono(ProductResponse.class)
            .timeout(Duration.ofSeconds(3))
            .onErrorResume(TimeoutException.class, e ->
                Mono.just(new ProductResponse(id, "Timeout", null, 0)));
    }
}
```

## Step 3: RestClient Timeout

```java
@Configuration
public class RestClientConfig {
    @Bean
    public RestClient productRestClient() {
        var requestFactory = new JdkClientHttpRequestFactory(
            HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(2))
                .build());
        requestFactory.setReadTimeout(Duration.ofSeconds(3));

        return RestClient.builder()
            .baseUrl("http://product-service:8081")
            .requestFactory(requestFactory)
            .build();
    }
}
```

## Step 4: Database Timeout

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 2000
      max-lifetime: 600000
      validation-timeout: 1000
    url: jdbc:postgresql://db:5432/products?connectTimeout=2&socketTimeout=5
```

```properties
# JPA query timeout
spring.jpa.properties.jakarta.persistence.query.timeout=3000
```

## Step 5: Timeout Strategy

| Service Type | Connection | Read |
|-------------|------------|------|
| Internal microservice | 2s | 3s |
| External API (third-party) | 5s | 10s |
| Database | 2s | 5s |
| Report generation | 5s | 30s |

Guidelines:
- Internal services should be fast — set tight timeouts (2-3s)
- External APIs are less predictable — allow more time
- Database queries should have per-query timeouts
- Report generation is inherently slow — use longer timeouts or async processing

## Key Points

- Set timeouts at every level: HTTP client, database driver, thread pool
- Use the shortest timeout that works for your use case
- TimeLimiter + CircuitBreaker + Retry work together: timeout triggers retry, retries feed the circuit breaker
- Monitor timeout rates — high timeout counts indicate a slow downstream service
- Always provide a fallback for timed-out requests
