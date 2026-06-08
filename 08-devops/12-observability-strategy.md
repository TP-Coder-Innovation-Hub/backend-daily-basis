# Observability Strategy — Metrics + Logs + Traces

## The Three Pillars

| Pillar | Answers | Example |
|--------|---------|---------|
| Metrics | "How many? How fast? How often?" | Error rate, latency p95, throughput |
| Logs | "What happened? What went wrong?" | Stack traces, business events |
| Traces | "Where is it slow? What called what?" | Service waterfall, span durations |

Each pillar alone is insufficient. Together they give you complete visibility.

## What to Monitor First

Start with what affects users, not infrastructure.

### Layer 1: User-Facing Metrics (Day 1)

```java
@Configuration
public class ObservabilityConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

```
- HTTP error rate (5xx) — are users seeing failures?
- Request latency (p95, p99) — is the app slow?
- Throughput (requests/sec) — can we handle the load?
```

### Layer 2: Business Metrics (Week 1)

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final MeterRegistry registry;

    public OrderResponse createOrder(OrderRequest request) {
        var order = process(request);
        registry.counter("orders.created",
            "category", order.getCategory()).increment();
        return toResponse(order);
    }
}
```

```
- Orders created per minute
- Payment success rate
- Active users
```

### Layer 3: Infrastructure Metrics (Week 2)

```
- JVM heap usage, GC pauses
- Database connection pool utilization
- Thread pool saturation
- Disk, CPU, memory at the host level
```

## Alerting Strategy

### Alert on Symptoms, Not Causes

```
BAD:  Alert on "CPU > 80%"
GOOD: Alert on "p95 latency > 500ms for 5 minutes"
```

CPU might be high because of a legitimate traffic spike. Latency affects users directly.

### Alert Severity

| Level | Response Time | Example |
|-------|---------------|---------|
| Critical | Immediate (page) | Error rate > 5%, service down |
| Warning | Within 1 hour | p95 latency > 2x normal |
| Info | Next business day | Deploy completed, scale event |

### Practical Alerts

```yaml
# Prometheus alert rules
groups:
  - name: product-service
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) / rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for 5 minutes"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 latency above 500ms"

      - alert: DatabasePoolExhaustion
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 3m
        labels:
          severity: critical
```

## SLOs and Error Budgets

An SLO (Service Level Objective) is a reliability target. Example: "99.9% of requests complete in under 500ms."

```
Error budget = 1 - SLO
99.9% SLO → 0.1% error budget per month = 43.2 minutes of downtime allowed
```

If you've burned your error budget by mid-month, freeze deployments and focus on reliability.

## Runbooks

Every alert needs a runbook — a document that explains:

1. What the alert means
2. How to verify the issue
3. Steps to fix it
4. Who to escalate to

```markdown
# Runbook: HighErrorRate

## What: Error rate above 5% for 5 minutes
## Verify: Check /actuator/health and recent deployments
## Fix:
  1. Check recent deployment: kubectl rollout history deployment/product-service
  2. If new deployment, rollback: kubectl rollout undo deployment/product-service
  3. Check downstream services: /actuator/health on payment-service
  4. Check database connectivity: /actuator/health/db
## Escalate: #platform-engineering Slack channel
```

## Practical Production Setup

```
Spring Boot → Micrometer → Prometheus → Grafana (dashboards)
Spring Boot → Logback → Logstash → Elasticsearch → Kibana (logs)
Spring Boot → Micrometer Tracing → Zipkin/Jaeger (traces)
Prometheus → Alertmanager → Slack/PagerDuty (alerts)
```

## Key Points

- Start with user-facing metrics: error rate, latency, throughput
- Alert on symptoms (latency, errors) not causes (CPU, disk)
- Define SLOs and track error budgets
- Write runbooks for every alert — during an incident is too late
- Correlate metrics, logs, and traces using trace IDs for fast debugging
