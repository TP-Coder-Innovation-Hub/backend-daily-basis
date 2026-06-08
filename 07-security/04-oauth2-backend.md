# OAuth2 Backend — Machine-to-Machine Authentication

## Client Credentials Flow

When one backend service calls another, there is no user. The client credentials flow lets a service authenticate as itself:

```
1. Service A sends client_id + client_secret to auth server
2. Auth server returns an access token
3. Service A includes token when calling Service B
4. Service B validates the token
```

No user involvement. No browser redirect. Pure machine-to-machine.

## Step 1: OAuth2 Client Configuration (Caller Service)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          order-service:
            client-id: ${ORDER_SERVICE_CLIENT_ID}
            client-secret: ${ORDER_SERVICE_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            token-uri: https://auth.example.com/oauth2/token
        provider:
          order-service:
            token-uri: https://auth.example.com/oauth2/token
```

## Step 2: Call Another Service with Token

```java
@Configuration
public class OAuth2ClientConfig {
    @Bean
    public RestClient orderServiceClient(
            OAuth2AuthorizedClientManager clientManager) {
        var interceptor = new OAuth2ClientHttpRequestInterceptor(
            clientManager, "order-service");
        return RestClient.builder()
            .baseUrl("https://order-service.internal")
            .requestInterceptor(interceptor)
            .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class PaymentOrchestrator {
    private final RestClient orderServiceClient;

    public OrderResponse getOrder(Long orderId) {
        return orderServiceClient.get()
            .uri("/api/orders/{id}", orderId)
            .retrieve()
            .body(OrderResponse.class);
    }
}
```

Spring automatically fetches a token before the first request and caches it until expiration.

## Step 3: Resource Server Configuration (Receiving Service)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          jwk-set-uri: https://auth.example.com/oauth2/jwks
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s ->
                s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/api/internal/**")
                    .hasAuthority("SCOPE_order.read")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

Spring validates the JWT automatically — checks signature, expiration, and issuer. No custom filter needed.

## Step 4: Scope-Based Authorization

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @GetMapping
    @PreAuthorize("hasAuthority('SCOPE_order.read')")
    public List<OrderSummary> listOrders() {
        return orderService.listAll();
    }

    @PostMapping
    @PreAuthorize("hasAuthority('SCOPE_order.write')")
    public ResponseEntity<OrderResponse> create(
            @RequestBody OrderRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(orderService.create(request));
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('SCOPE_order.read')")
    public ResponseEntity<OrderResponse> get(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.get(id));
    }
}
```

## Step 5: Token Exchange

When Service A calls Service B, which calls Service C, you need to forward the original token or exchange it:

```java
@Configuration
public class TokenForwardingConfig {
    @Bean
    public RestTemplate restTemplate(
            OAuth2AuthorizedClientManager clientManager) {
        var template = new RestTemplate();
        template.setInterceptors(List.of(
            (request, body, execution) -> {
                var currentToken = SecurityContextHolder.getContext()
                    .getAuthentication();
                if (currentToken instanceof JwtAuthenticationToken jwtAuth) {
                    request.getHeaders().set("Authorization",
                        "Bearer " + jwtAuth.getToken().getTokenValue());
                }
                return execution.execute(request, body);
            }
        ));
        return template;
    }
}
```

## Key Points

- Client credentials flow is for services, not users
- Spring's `OAuth2AuthorizedClientManager` handles token acquisition and caching
- Resource server validates JWTs automatically — no custom code
- Use scopes to control what each client can access
- For token forwarding across service chains, propagate the original JWT
