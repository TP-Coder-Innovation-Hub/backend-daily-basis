# Environment Variables — 12-Factor Configuration

## Why Externalize

Hardcoded values in your code mean you deploy differently for dev, staging, and prod. The 12-factor app methodology says: store config in environment variables. Your codebase contains no secrets and works in any environment.

## Step 1: @Value Annotation

```java
@Service
public class PaymentService {
    @Value("${payment.gateway.url}")
    private String gatewayUrl;

    @Value("${payment.timeout.seconds:30}")
    private int timeoutSeconds;

    @Value("${payment.api-key}")
    private String apiKey;
}
```

`@Value` is fine for a handful of properties. The `:30` syntax provides a default value.

## Step 2: @ConfigurationProperties (Preferred)

```java
@Configuration
@ConfigurationProperties(prefix = "payment")
@ConstructorBinding
public record PaymentProperties(
    String gatewayUrl,
    int timeoutSeconds,
    String apiKey,
    RetryConfig retry
) {
    public record RetryConfig(
        int maxAttempts,
        long delayMs,
        double multiplier
    ) {}
}
```

```yaml
# application.yml
payment:
  gateway-url: https://api.payment.com
  timeout-seconds: 30
  api-key: ${PAYMENT_API_KEY}
  retry:
    max-attempts: 3
    delay-ms: 1000
    multiplier: 2.0
```

Why `@ConfigurationProperties` over `@Value`:
- Type-safe — compiler catches errors
- Grouped — related properties in one object
- Validatable — use `@Valid` with Bean Validation
- IDE-friendly — autocomplete in `application.yml`

## Step 3: Spring Profiles

```yaml
# application.yml (shared defaults)
server:
  port: 8080

spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/dev
    username: dev
    password: dev
logging:
  level:
    root: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}
    hikari:
      maximum-pool-size: 20
logging:
  level:
    root: WARN
```

Activate a profile:

```bash
java -jar app.jar --spring.profiles.active=prod
# or
export SPRING_PROFILES_ACTIVE=prod
```

## Step 4: Profile-Specific Beans

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:h2:mem:devdb")
            .driverClassName("org.h2.Driver")
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource(
            @Value("${spring.datasource.url}") String url) {
        return DataSourceBuilder.create()
            .url(url)
            .type(HikariDataSource.class)
            .build();
    }
}
```

## Config Hierarchy (Highest Wins)

```
1. Command-line arguments (--server.port=8081)
2. Environment variables (SERVER_PORT=8081)
3. application-{profile}.yml
4. application.yml
5. @PropertySource files
6. Default values in code
```

## Key Points

- Never commit secrets — use `${ENV_VAR}` placeholders in YAML
- Use `@ConfigurationProperties` for grouped config, `@Value` for single values
- Profiles handle environment differences without code changes
- In production, inject secrets via environment variables or secret managers, not files
