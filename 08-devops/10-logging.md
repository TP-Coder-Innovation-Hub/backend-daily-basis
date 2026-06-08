# Logging — Structured Logging and Aggregation

## Why Structured Logging

Plain text logs are hard to parse and search. JSON logs are machine-readable — log aggregation systems (ELK, Datadog, Splunk) index every field for instant filtering.

```json
{"timestamp":"2024-01-15T10:30:00.123Z","level":"INFO","logger":"c.e.o.OrderService","message":"Order created","traceId":"abc123","orderId":42,"customerId":"C-100","duration":125}
```

## Step 1: Logback JSON Configuration

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProperty scope="context" name="APP_NAME"
        source="spring.application.name" defaultValue="app"/>
    <springProperty scope="context" name="ENV"
        source="spring.profiles.active" defaultValue="dev"/>

    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{
                "app":"${APP_NAME}",
                "env":"${ENV}"
            }</customFields>
            <includeMdc>true</includeMdc>
            <includeContext>true</includeContext>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>

    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
</configuration>
```

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

## Step 2: Correlation IDs for Request Tracing

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    private static final String HEADER = "X-Correlation-ID";
    public static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        var correlationId = request.getHeader(HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }
        MDC.put(MDC_KEY, correlationId);
        response.setHeader(HEADER, correlationId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

MDC (Mapped Diagnostic Context) attaches the correlation ID to every log line in the request thread. The JSON encoder includes MDC fields automatically.

## Step 3: Structured Logging in Code

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    private final OrderRepository repository;

    public OrderResponse createOrder(OrderRequest request) {
        var start = Instant.now();
        var order = new Order();
        order.setCustomerId(request.customerId());
        order.setItems(request.items());
        var saved = repository.save(order);

        log.info("Order created orderId={} customerId={} itemCount={} duration={}ms",
            saved.getId(),
            saved.getCustomerId(),
            saved.getItems().size(),
            Duration.between(start, Instant.now()).toMillis());

        return toResponse(saved);
    }

    public OrderResponse getOrder(Long id) {
        log.debug("Fetching order orderId={}", id);
        return repository.findById(id)
            .map(this::toResponse)
            .orElseThrow(() -> {
                log.warn("Order not found orderId={}", id);
                return new ResourceNotFoundException("Order not found");
            });
    }
}
```

Key=value pairs in log messages are parsed as structured fields by most log aggregation systems.

## Step 4: Log Levels in Production

| Level | When to Use |
|-------|-------------|
| ERROR | Unrecoverable failures requiring immediate attention |
| WARN | Recoverable issues, degraded behavior |
| INFO | Business events (order created, payment processed) |
| DEBUG | Technical details for troubleshooting |
| TRACE | Very verbose (SQL queries, HTTP bodies) |

```yaml
# application-prod.yml
logging:
  level:
    root: WARN
    com.example: INFO
    org.springframework.web: WARN
    org.hibernate.SQL: WARN
```

## Step 5: ELK Stack Integration

```yaml
# logstash.conf
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}
filter {
  mutate {
    rename => { "[headers][X-Correlation-ID]" => "correlationId" }
  }
}
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
}
```

```xml
<!-- Send logs to Logstash over TCP -->
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash:5000</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

## Key Points

- Use JSON logging in production — searchable, filterable, aggregatable
- Correlation IDs (in MDC) link all log lines for a single request
- Log business events at INFO, technical details at DEBUG
- Keep production log levels at INFO/WARN — DEBUG floods the system
- ELK stack: Logback → Logstash → Elasticsearch → Kibana for search and visualization
