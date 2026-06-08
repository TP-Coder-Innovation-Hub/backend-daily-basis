# Config Files — YAML, Properties, and Precedence

## application.yml vs application.properties

Both do the same thing. YAML supports hierarchy and lists. Properties is flat. Use YAML.

```properties
# application.properties — flat, no nesting
spring.datasource.url=jdbc:postgresql://localhost/db
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=10
```

```yaml
# application.yml — hierarchical, clearer structure
spring:
  datasource:
    url: jdbc:postgresql://localhost/db
    username: admin
    password: secret
    hikari:
      maximum-pool-size: 10
```

## Profile-Specific Files

```
src/main/resources/
  application.yml              # shared defaults
  application-dev.yml          # dev overrides
  application-staging.yml      # staging overrides
  application-prod.yml         # prod overrides
  application-test.yml         # test overrides
```

Spring merges profile files on top of the base file. Profile values override base values.

## Multi-Document YAML

Split sections within a single file using `---`:

```yaml
# application.yml — all profiles in one file
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost/dev

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
    hikari:
      maximum-pool-size: 20

server:
  port: 8443

---
spring:
  config:
    activate:
      on-profile: test
  datasource:
    url: jdbc:h2:mem:testdb
```

## Config Precedence (Highest Priority First)

```
1. Command-line arguments          --server.port=9090
2. JVM system properties           -Dserver.port=9090
3. OS environment variables        SERVER_PORT=9090
4. application-{profile}.yml       (outside jar)
5. application-{profile}.yml       (inside jar)
6. application.yml                  (outside jar)
7. application.yml                  (inside jar)
```

"Outside jar" means a `config/` directory next to your jar or the current working directory. This lets ops teams override without rebuilding.

## Step 1: External Config Directory

```bash
# Place config files next to your jar
myapp/
  app.jar
  config/
    application.yml        # overrides internal config
    application-prod.yml   # overrides internal profile config
```

```bash
java -jar app.jar --spring.config.location=file:./config/
```

## Step 2: Property Placeholder Resolution

```yaml
app:
  name: product-service
  version: @project.version@  # Maven resource filtering

database:
  host: ${DB_HOST:localhost}   # default: localhost
  port: ${DB_PORT:5432}
  url: jdbc:postgresql://${database.host}:${database.port}/products
```

The `${DB_HOST:localhost}` syntax means: use the `DB_HOST` env var, or fall back to `localhost`.

## Step 3: Loading Order for Complex Config

```java
@Configuration
@PropertySource("classpath:additional.properties")
@PropertySource(value = "file:${config.path}", ignoreResourceNotFound = true)
public class AdditionalConfig {
}
```

`@PropertySource` loads extra files. The `file:` prefix loads from the filesystem. Use `ignoreResourceNotFound = true` when the file is optional.

## Key Points

- Use YAML — hierarchical config is easier to read and maintain
- Profile-specific files keep environment differences isolated
- Multi-document YAML works for small projects; separate files scale better
- External config (outside the jar) overrides internal config without rebuilding
- Always provide sensible defaults with `${VAR:default}` syntax
