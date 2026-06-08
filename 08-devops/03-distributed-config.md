# Distributed Config — Spring Cloud Config Server

## The Problem

Ten microservices, each with its own configuration. Changing a database URL means updating ten config files and redeploying ten services. Spring Cloud Config centralizes configuration in one place.

## Step 1: Config Server Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config server's application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/example/config-repo
          default-label: main
          clone-on-start: true
          search-paths: '{application}'
```

## Step 2: Config Repository Structure

```
config-repo/
  product-service/
    application.yml
    application-staging.yml
    application-prod.yml
  order-service/
    application.yml
    application-staging.yml
    application-prod.yml
```

```yaml
# config-repo/product-service/application.yml
server:
  port: 8080

spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}

feature-flags:
  new-search: false
  bulk-export: true
```

```yaml
# config-repo/product-service/application-prod.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

feature-flags:
  new-search: true
```

## Step 3: Config Client (Your Service)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml (required for config client)
spring:
  application:
    name: product-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:default}
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
```

The client fetches config from the server on startup. `fail-fast: true` with retry means it keeps trying if the config server is temporarily down.

## Step 4: Encryption of Sensitive Values

Install the JCE unlimited strength extension and set an encryption key:

```bash
# Set the encrypt key as an environment variable
export ENCRYPT_KEY=my-super-secret-key
```

Encrypt values through the config server:

```bash
curl http://config-server:8888/encrypt -d 'my-db-password'
# Returns: AQCGw...
```

Use the encrypted value in your config files:

```yaml
spring:
  datasource:
    password: '{cipher}AQCGw...'
```

The config server decrypts it before serving to clients.

## Step 5: Dynamic Refresh Without Restart

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```java
@RestController
@RefreshScope
public class FeatureFlagController {
    @Value("${feature-flags.new-search}")
    private boolean newSearchEnabled;

    @GetMapping("/features")
    public Map<String, Boolean> getFeatures() {
        return Map.of("newSearch", newSearchEnabled);
    }
}
```

```bash
# Trigger refresh (pulls latest config from server)
curl -X POST http://product-service:8080/actuator/refresh
```

`@RefreshScope` tells Spring to recreate the bean with new config values. No restart needed.

For multiple services, use Spring Cloud Bus with RabbitMQ or Kafka to broadcast refresh events.

## Key Points

- Centralized config in a Git repository — version controlled, auditable
- Profile-specific files handle environment differences
- Encrypt sensitive values with `{cipher}` prefix
- `@RefreshScope` enables config reload without restart
- Use Spring Cloud Bus to refresh all services at once
