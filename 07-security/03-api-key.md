# API Key — Service-to-Service Authentication

## Why API Keys

OAuth2 is for users. API keys are for services. When your backend calls another backend, an API key is simpler than OAuth2 client credentials.

## When API Key vs OAuth2

| API Key | OAuth2 |
|---------|--------|
| Simple, internal service calls | Complex authorization flows |
| One string to validate | Token lifecycle management |
| No expiration (or manual rotation) | Automatic token expiration |
| Quick to implement | Full framework support |

## Step 1: API Key Storage

```sql
CREATE TABLE api_keys (
    id BIGSERIAL PRIMARY KEY,
    key_hash VARCHAR(128) NOT NULL UNIQUE,
    key_prefix VARCHAR(8) NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    last_used_at TIMESTAMP
);

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);
```

Store the hash, not the raw key. The prefix lets you look up keys quickly without scanning all hashes.

## Step 2: Key Generation

```java
@Service
@RequiredArgsConstructor
public class ApiKeyService {
    private final ApiKeyRepository repository;

    public GeneratedApiKey generateKey(String serviceName, List<String> scopes) {
        var rawKey = "sk_" + Base64.getUrlEncoder()
            .withoutPadding()
            .encodeToString(UUID.randomUUID().toString().getBytes())
            + UUID.randomUUID().toString().replace("-", "").substring(0, 8);
        var prefix = rawKey.substring(0, 8);
        var hash = hashKey(rawKey);

        var entity = new ApiKey();
        entity.setKeyHash(hash);
        entity.setKeyPrefix(prefix);
        entity.setServiceName(serviceName);
        entity.setScopes(scopes);
        entity.setActive(true);
        entity.setCreatedAt(Instant.now());
        repository.save(entity);

        return new GeneratedApiKey(rawKey, prefix, serviceName, scopes);
    }

    private String hashKey(String key) {
        return Hashing.sha256()
            .hashString(key, StandardCharsets.UTF_8)
            .toString();
    }
}

public record GeneratedApiKey(
    String key, String prefix,
    String serviceName, List<String> scopes
) {}
```

## Step 3: API Key Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class ApiKeyAuthFilter extends OncePerRequestFilter {
    private final ApiKeyRepository repository;

    private static final String HEADER_NAME = "X-API-Key";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        var apiKey = request.getHeader(HEADER_NAME);
        if (apiKey != null) {
            var prefix = apiKey.substring(0, Math.min(8, apiKey.length()));
            var hash = Hashing.sha256()
                .hashString(apiKey, StandardCharsets.UTF_8).toString();

            var keyEntity = repository
                .findByKeyPrefixAndActiveTrue(prefix)
                .filter(k -> k.getKeyHash().equals(hash))
                .filter(k -> k.getExpiresAt() == null
                    || k.getExpiresAt().isAfter(Instant.now()))
                .orElse(null);

            if (keyEntity != null) {
                repository.updateLastUsed(keyEntity.getId(), Instant.now());
                var authorities = keyEntity.getScopes().stream()
                    .map(scope -> (GrantedAuthority)
                        () -> "SCOPE_" + scope.toUpperCase())
                    .toList();
                var auth = new PreAuthenticatedAuthenticationToken(
                    keyEntity.getServiceName(), null, authorities);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } else {
                response.sendError(HttpServletResponse.SC_UNAUTHORIZED,
                    "Invalid API key");
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

## Step 4: Register the Filter

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    private final ApiKeyAuthFilter apiKeyFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s ->
                s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/internal/**").authenticated()
                .anyRequest().permitAll())
            .addFilterBefore(apiKeyFilter,
                UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

## Step 5: Scope-Based Authorization

```java
@RestController
@RequestMapping("/internal/data")
public class InternalDataController {
    @GetMapping("/users")
    @PreAuthorize("hasAuthority('SCOPE_USERS_READ')")
    public List<UserSummary> listUsers() {
        return userService.listAll();
    }

    @PostMapping("/sync")
    @PreAuthorize("hasAuthority('SCOPE_DATA_WRITE')")
    public ResponseEntity<Void> syncData(@RequestBody SyncRequest request) {
        dataService.sync(request);
        return ResponseEntity.ok().build();
    }
}
```

## Key Rotation

```java
@PutMapping("/api-keys/{id}/rotate")
public ResponseEntity<GeneratedApiKey> rotate(@PathVariable Long id) {
    var existing = repository.findById(id).orElseThrow();
    existing.setActive(false);
    repository.save(existing);
    var newKey = apiKeyService.generateKey(
        existing.getServiceName(), existing.getScopes());
    return ResponseEntity.ok(newKey);
}
```

Rotate keys by generating a new one and deactivating the old. Give the consuming service time to update before deactivating.

## Key Points

- Store hashes, not raw keys — same principle as passwords
- Use a prefix lookup for fast retrieval without full table scans
- Track `last_used_at` to find unused keys for cleanup
- Support scopes for fine-grained access control per service
- Rotate keys regularly — provide a transition window during rotation
