# Metrics — Spring Boot Actuator and Prometheus

## Spring Boot Actuator

Actuator exposes operational endpoints: health, metrics, info, env. Add Prometheus integration to feed metrics to a monitoring system.

## Step 1: Add Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## Step 2: Expose Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when-authorized
  metrics:
    tags:
      application: product-service
    export:
      prometheus:
        enabled: true
```

```bash
curl http://localhost:8080/actuator/prometheus
```

This endpoint returns metrics in Prometheus format. Prometheus scrapes it periodically.

## Step 3: Custom Metrics with @Timed

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;

    @Timed(value = "orders.create", description = "Time to create an order")
    public OrderResponse createOrder(OrderRequest request) {
        var order = new Order();
        order.setCustomerId(request.customerId());
        order.setItems(request.items());
        order.setTotal(calculateTotal(request.items()));
        var saved = repository.save(order);
        return toResponse(saved);
    }

    @Timed(value = "orders.list", description = "Time to list orders")
    public Page<OrderResponse> listOrders(Pageable pageable) {
        return repository.findAll(pageable).map(this::toResponse);
    }
}
```

`@Timed` automatically records:
- `orders.create.count` — number of invocations
- `orders.create.sum` — total time
- `orders.create.max` — maximum time
- `orders.create.percentile` — percentile distribution

## Step 4: Custom Metrics with MeterRegistry

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final MeterRegistry registry;
    private final PaymentGateway gateway;

    public PaymentResult process(PaymentRequest request) {
        var timer = registry.timer("payments.process",
            "method", request.method());
        return timer.record(() -> {
            var result = gateway.charge(request);
            registry.counter("payments.completed",
                "status", result.status(),
                "method", request.method()).increment();
            return result;
        });
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class OrderMetrics {
    private final MeterRegistry registry;

    public void recordOrderCreated(BigDecimal amount, String category) {
        registry.counter("orders.created.total",
            "category", category).increment();
        registry.summary("orders.amount",
            "category", category).record(amount.doubleValue());
    }

    public void recordActiveOrders(int count) {
        registry.gauge("orders.active", count);
    }
}
```

## Step 5: The RED Method

Three metrics define service health:

| Metric | What It Measures | Alert On |
|--------|-----------------|----------|
| **Rate** | Requests per second | Sudden drops or spikes |
| **Errors** | Failed request rate | Error rate > 1% |
| **Duration** | Response time (p50, p95, p99) | p95 > SLO |

```java
@Configuration
public class MetricsConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

## Step 6: Grafana Dashboard Concepts

Prometheus collects metrics. Grafana visualizes them. Key panels:

```
Panel 1: Request Rate
  sum(rate(http_server_requests_seconds_count{uri!~".*actuator.*"}[5m]))
  by (uri, method)

Panel 2: Error Rate
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))

Panel 3: Latency P95
  histogram_quantile(0.95,
    sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))

Panel 4: JVM Memory
  jvm_memory_used_bytes{area="heap"}

Panel 5: HikariCP Active Connections
  hikaricp_connections_active
```

## Key Points

- Actuator + Micrometer gives you metrics for free — JVM, HTTP, database connections
- Use `@Timed` on service methods to track business metrics
- Follow the RED method: Rate, Errors, Duration for every service
- Prometheus scrapes `/actuator/prometheus`; Grafana queries Prometheus
- Tag metrics with business dimensions (category, method, status) for meaningful dashboards
